# LinkedIn Post

I tried to run an AI agent on a $48/month server.

It took 10 minutes to answer "what's 7 times 8."

Here's what I learned — including the one finding that completely changed my conclusion. 🧵

---

The setup: I host my own AI assistant (OpenClaw + Telegram) on a small DigitalOcean VPS. It runs on Google's Gemini API. Cheap, fast, but it depends on the cloud.

So I asked the obvious question: can I cut the cloud out entirely and run a local model with Ollama on the same box?

4 vCPU, 8 GB RAM. No GPU. What could go wrong.

Everything, at first.

→ The 7B model got killed by the OOM reaper.
→ The 4B model couldn't call tools at all.
→ The 3B model finally worked... in 10 minutes and 25 seconds. Per message.

I almost wrote it off as "CPUs are just too slow for agents." Then I stopped guessing and started measuring.

The real culprit wasn't generation speed. It was the PROMPT.

My agent's system prompt is 20,816 tokens — and 70% of that is just the JSON schemas describing its tools. On this CPU, reading that prompt takes ~14 minutes BEFORE the model writes a single word.

But here's the finding that flipped everything:

🔑 The KV cache is reused across separate requests.

I sent a fresh 1,266-token prompt. First run: 48 seconds. Second identical run: 0.1 seconds. A 480x speedup.

Meaning: a stable prompt prefix is paid for ONCE. Every turn after that is nearly free.

So the path from "unusable" to "actually viable" turned out to be two levers — not a bigger server:

1. Trim the prompt (most of it is tool definitions you don't need)
2. Keep the model warm so the cache survives

Honest ending? Local still loses to Gemini Flash for daily use (2-4s, free, smarter). But for offline, privacy, or just understanding how this stuff actually works under the hood — the gap is smaller than the internet told me.

Full writeup with all the numbers in the comments. 👇

#LLM #SelfHosted #Ollama #AIEngineering #LocalLLM #DevOps
