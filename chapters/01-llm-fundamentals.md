# Chapter 1: LLM Fundamentals

## Overview

Before designing AI systems, you need to understand how LLMs work at a systems level — not the math, but the engineering properties that affect your architecture decisions.

## Key Concepts

### Tokens

LLMs don't process text — they process tokens. A token is roughly 4 characters or 0.75 words in English.

```
"Hello, world!" → ["Hello", ",", " world", "!"] → 4 tokens
"System design" → ["System", " design"] → 2 tokens
```

**Why it matters for system design:**
- Cost is per-token (input and output priced separately)
- Context window is measured in tokens
- Latency scales with output token count
- You must estimate token usage for capacity planning

### Context Window

The context window is the maximum number of tokens the model can process in a single request (input + output combined).

| Model | Context Window | ~Pages of Text |
|-------|---------------|----------------|
| GPT-4o | 128K | ~200 pages |
| Claude Sonnet 4 | 200K | ~300 pages |
| Gemini 2.5 Pro | 1M | ~1500 pages |
| Llama 3.2 (8B) | 128K | ~200 pages |

**Architecture implications:**
- Larger context ≠ better. Performance degrades with very long contexts ("lost in the middle" problem).
- If your data exceeds the context window, you need RAG (Chapter 3).
- Context window determines how much retrieval you can stuff into a prompt.

### Temperature & Sampling

Temperature controls randomness. This affects system design because it determines reproducibility.

```
Temperature 0.0 → Nearly deterministic (same input ≈ same output)
Temperature 0.7 → Creative, varied responses
Temperature 1.0 → Very random, sometimes incoherent
```

**System design rules:**
- Classification/extraction: temperature 0
- Creative writing: temperature 0.7-1.0
- Code generation: temperature 0-0.3
- If you need reproducible outputs for testing, use temperature 0 + seed parameter

### Inference Latency

LLM latency has two components:

```
Total latency = Time to First Token (TTFT) + (Output tokens × Time per token)
```

| Component | Typical Range | What Affects It |
|-----------|---------------|-----------------|
| TTFT | 200ms - 2s | Model size, provider load, prompt length |
| Per-token | 10-50ms | Model size, provider |
| Total (500 tokens out) | 1-5s | Everything above |

**This is 100x slower than a typical API call.** Your architecture must account for this:
- Use streaming for user-facing responses
- Use async processing for batch workloads
- Cache aggressively (same prompt = same response at temp 0)
- Consider smaller models for latency-sensitive paths

### Model Families & Trade-offs

```
                    Quality
                      ↑
                      │
        Opus/GPT-4o ──┤── $$$, slow, best reasoning
                      │
     Sonnet/GPT-4o ──┤── $$, balanced
                      │
  Haiku/GPT-4o-mini ──┤── $, fast, good enough for most tasks
                      │
        Local/Llama ──┤── Free, private, variable quality
                      │
                      └──────────────────→ Speed/Cost
```

**The key insight:** Most production systems use MULTIPLE models. Route simple tasks to cheap/fast models, complex tasks to expensive/smart ones. This is called **model tiering** (covered in Chapter 5).

## Architecture Pattern: Model Tiering

```
Request → Classifier (Haiku) → Simple? → Haiku (fast, cheap)
                               │
                               └→ Complex? → Sonnet (balanced)
                               │
                               └→ Critical? → Opus (best quality)
```

**Real-world example:** A customer support bot:
- FAQ matching → Haiku ($0.001/request)
- General questions → Sonnet ($0.01/request)
- Escalation/complaints → Opus ($0.05/request)
- **Result:** 80% of requests hit the cheapest tier

## Common Pitfalls

1. **Assuming determinism.** LLMs are probabilistic. Even at temperature 0, outputs can vary across API calls. Design for this.
2. **Ignoring token costs.** A system processing 1M requests/day at $0.01 each = $10K/day. Cost modeling is essential.
3. **Treating LLMs like databases.** They don't have consistent retrieval. If you need exact data, use a database + RAG.
4. **Not accounting for latency.** A 3-second LLM call in a synchronous request path kills UX. Stream or go async.
5. **Using one model for everything.** Model tiering can cut costs 5-10x with minimal quality loss.

## Practice Questions

1. You're designing a system that processes 500K customer emails per day. Each email needs classification (5 categories) and a draft response. How would you choose models for each task?

2. Your RAG system's context window is filling up — you're stuffing 50 retrieved chunks (2000 tokens each) into a 128K window, leaving only 28K for the response. What are your options?

3. A stakeholder asks why your AI feature takes 3 seconds to respond when the rest of the app responds in 200ms. How do you explain this, and what architectural changes would you propose?

## Further Reading

- [Anthropic's guide to prompt engineering](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering)
- [OpenAI's tokenizer tool](https://platform.openai.com/tokenizer)
- [LLM pricing comparison (updated)](https://www.usagepricing.com/ai-token-pricing)
