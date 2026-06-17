# I Tried to Run an AI Agent on a $48 Server. It Took 10 Minutes to Say "Hello."

*What actually limits local LLM agents on a CPU-only VPS — measured, not guessed.*

---

## The setup

I run a self-hosted AI assistant on a small DigitalOcean droplet: 4 shared vCPUs, 8 GB RAM, no GPU. It powers a Telegram bot, runs scheduled tasks, and searches the web. Until now it talked to the cloud — Google's Gemini Flash over the API. Fast, basically free, and good enough.

But cloud dependence has a cost beyond money: when Google's preview models get overloaded, my whole assistant goes down. So I asked the obvious question:

**Can I run the model locally instead, with [Ollama](https://ollama.com/), on the same machine?**

This is the story of finding out — including the one measurement that flipped my conclusion from "no" to "actually, kind of yes."

---

## Act 1: Everything fails

I started optimistically and worked down.

**`qwen2.5:7b`** — the OOM killer terminated it mid-inference. A 7B model plus its KV cache plus the OS and my other services didn't fit in 8 GB. I added a 4 GB swapfile and shrank the context window. It survived, but each turn took ~20 minutes.

**`gemma3:4b`** — smaller, should be faster. Instead it failed instantly with `400: does not support tools`. Gemma 3's chat template has no tool-calling support, and an agent without tools is useless.

**`qwen2.5:3b`** — finally worked end to end. It also took **10 minutes and 25 seconds** to answer a one-line question, and got the arithmetic wrong.

At this point the easy conclusion is "CPUs are too slow for agents, get a GPU." That conclusion is wrong, or at least incomplete. I only found out because I stopped guessing and started measuring.

---

## Act 2: Measuring the real bottleneck

An LLM does two very different jobs per request:

1. **Prefill** — reading and digesting the input prompt. Compute-heavy.
2. **Generation** — producing output tokens, one at a time.

Ollama's native API (`/api/generate`) returns timing for each separately, in nanoseconds. So I could isolate them instead of staring at a stopwatch.

First, the number that mattered most. **How big is my agent's prompt, really?**

```
promptTokens: 20,816
tools: 23  (schemas ≈ 14,700 tokens — about 70% of the prompt)
```

Twenty thousand tokens. And roughly **70% of it was just the JSON schemas describing the agent's tools** — file system, web search, memory, cron, image generation, and 18 others — sent on every single turn.

Then I measured prefill speed across prompt sizes on `qwen2.5:3b`:

| Prompt size | Prefill time | Throughput |
|---|---|---|
| ~73 tokens | 2.0 s | ~36 tok/s |
| ~850 tokens | 25 s | ~34 tok/s |
| ~1,650 tokens | 33 s | ~49 tok/s |
| ~4,100 tokens | 174 s | ~23 tok/s |

(The shared CPU is noisy — throughput swings ±2x depending on the neighbors. But the trend is clear and roughly linear.)

At ~23 tokens/second, a 20,816-token prompt takes about **14 minutes to read** — before the model generates a single word. That, not generation, was my 10-minute wall.

**Generation, by comparison, was fine:** ~10 tok/s on the 3B, ~18 tok/s on a 1.5B. A 200-token answer is ~11–20 seconds. Slow, but not the problem.

---

## Act 3: The finding that changed everything

If the cost is reading a giant, mostly-static prompt every time — what if it didn't have to be read every time?

Modern llama.cpp (which Ollama wraps) keeps a KV cache per slot. I wanted to know: **is that cache reused across separate API requests that share a prefix?** The docs don't say. So I tested it directly with a brand-new prompt the model had never seen:

```
First request  (1,266 tokens):  48.4 s   (26 tok/s)
Second request (identical):       0.1 s   (≈9,400 tok/s)
```

**A 480x speedup.** The second identical request skipped prefill almost entirely. The cache survived between two completely separate HTTP calls.

This reframes the whole problem. The expensive part — that 20k-token system prompt — is *stable*. It's the same on every turn. If the cache holds, you pay the prefill **once**, and every turn after that is nearly free. Even a tool call only needs to prefill the small new suffix (the tool's result); the cached prefix stays cached.

---

## What actually moves the needle (and what doesn't)

I tested the rest of the usual advice too:

| Lever | Result | Worth it? |
|---|---|---|
| **Trim the prompt** (`tools.profile: minimal` + only the tools you need) | Cuts ~70% of 20.8k tokens → prompt drops to ~6k | ✅ Biggest reliable win |
| **KV prefix cache** (keep prompt prefix stable) | 48 s → 0.1 s on cache hit | ✅ Changes the game — *if* the slot stays warm |
| **Keep model warm** (`OLLAMA_KEEP_ALIVE=-1`) | Default unloads after 5 min; cold reload ≈ 5.5 s, and it kills the cache | ✅ Required for the cache to help scheduled tasks |
| **Smaller model** (1.5B vs 3B) | ~1.8x faster generation, lower quality | ⚠️ Trade-off |
| **Flash attention + quantized KV cache** (`q8_0`) | 21 vs 26 tok/s — within noise | ❌ No prefill gain on CPU; saves RAM only |

The two big levers aren't hardware. They're **trim the prompt** and **keep the prefix cached and warm**. Flash attention and cache quantization — the things people reach for first — did nothing for speed on a CPU without a GPU.

---

## The honest verdict

Put it together: a trimmed ~6k-token prompt + a warm slot + a stable prefix gets you a **first turn of ~4 minutes (paid once), then subsequent turns in seconds.** That moves a local agent from "completely unusable" to "marginally usable for async work" — a Telegram bot you don't need to answer in real time, or a scheduled job.

But let's be honest about the comparison. Gemini Flash answers the same query in 2–4 seconds, for free, and it's smarter. For daily interactive use, the cloud still wins decisively.

So when is local worth it? Three cases:

- **Offline / air-gapped** environments where the cloud isn't an option.
- **Privacy** — when the data genuinely can't leave the box.
- **Learning** — and this is the underrated one. Profiling this taught me more about how LLM inference actually works than months of using APIs. The bottleneck wasn't where I assumed, the "obvious" optimizations were useless, and the real win was an undocumented caching behavior I only found by measuring.

The internet told me "CPUs are too slow for agents." The truth is more useful: **the prompt was too big, and I was paying for it on every turn instead of once.** That's a software problem, not a hardware one.

---

## Reproduce it yourself

The key trick is using Ollama's native timing fields instead of a stopwatch:

```bash
curl -s http://127.0.0.1:11434/api/generate -d '{
  "model": "qwen2.5:3b",
  "prompt": "<your prompt>",
  "stream": false,
  "keep_alive": -1,
  "options": {"num_predict": 8}
}' | python3 -c '
import json,sys
d=json.load(sys.stdin)
pe=d["prompt_eval_count"]; pd=d["prompt_eval_duration"]/1e9
print(f"prefill: {pe} tokens in {pd:.1f}s = {pe/pd:.1f} tok/s")'
```

Send the same prompt twice and watch the second prefill collapse to near zero. That's the whole discovery in one command.

---

*Running on: DigitalOcean, 4 vCPU (shared, AVX2-only), 8 GB RAM, Ubuntu 24.04, Ollama 0.30.7, qwen2.5. No GPU.*
