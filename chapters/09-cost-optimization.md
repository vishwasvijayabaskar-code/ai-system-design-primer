# Chapter 9: Cost Optimization

## Overview

LLM costs scale with tokens, not compute. A poorly designed system can burn $10K/day on API calls. This chapter covers the strategies that cut costs 5-10x without sacrificing quality — prompt caching, model tiering, batching, and knowing when to fine-tune.

## Cost Anatomy

```
Cost per request = (input_tokens × input_price) + (output_tokens × output_price)

Example (Claude Sonnet):
  System prompt:     2,000 tokens × $3/1M  = $0.006
  RAG context:       3,000 tokens × $3/1M  = $0.009
  User message:        100 tokens × $3/1M  = $0.0003
  Response:            500 tokens × $15/1M = $0.0075
  Total:                                     $0.023 per request

  At 100K requests/day = $2,300/day = $69K/month
```

## Strategy 1: Model Tiering

Route requests to the cheapest model that meets quality requirements.

```
Request → Classifier → Simple (70%)  → Haiku    $0.001/req
                       Medium (25%)  → Sonnet   $0.023/req
                       Complex (5%)  → Opus     $0.075/req

Blended cost: 0.7×$0.001 + 0.25×$0.023 + 0.05×$0.075 = $0.010/req
vs all-Sonnet: $0.023/req → 57% savings
```

**How to classify:**
- Keyword rules (FAQ patterns → Haiku)
- Small classifier model (Haiku classifying for routing)
- Task type (classification → cheap, reasoning → expensive)
- User tier (free users → cheap, enterprise → best)

## Strategy 2: Prompt Caching

### Provider-Level Caching

```
Request 1: [System prompt: 2000 tokens] + [User: "What's your return policy?"]
  → Full price: 2100 tokens input

Request 2: [System prompt: 2000 tokens] + [User: "How do I track my order?"]
  → Cached prefix: 2000 tokens at 90% discount + 100 tokens full price
  → Effective cost: 300 token-equivalents instead of 2100
```

**Anthropic prompt caching:** Mark static portions of your prompt with `cache_control`. Same prefix across requests gets cached for 5 minutes. Up to 90% discount on cached tokens.

**OpenAI automatic caching:** Automatically caches identical prompt prefixes. 50% discount. No configuration needed.

### Application-Level Caching

```python
cache = Redis()

def get_response(prompt, temperature=0):
    # Only cache deterministic responses
    if temperature > 0:
        return call_llm(prompt)

    cache_key = hash(prompt)
    cached = cache.get(cache_key)
    if cached:
        return cached

    response = call_llm(prompt)
    cache.set(cache_key, response, ttl=3600)
    return response
```

## Strategy 3: Prompt Optimization

```
Before (500 tokens):
  "You are a helpful, friendly, knowledgeable customer support
   agent working for Acme Corporation. Your job is to assist
   customers with their questions and concerns. Please be polite
   and professional at all times. When answering questions, please
   provide detailed and comprehensive responses..."

After (100 tokens):
  "Acme Corp support agent. Answer from context only.
   Be concise. If unsure, say 'Let me escalate this.'
   Never share internal pricing."

Savings: 80% fewer input tokens, same quality
```

**Rules:**
- Remove filler words from system prompts
- Use structured formats (lists > paragraphs)
- Fewer few-shot examples (3 not 10)
- Trim RAG context (smaller chunks, fewer chunks)
- Use `max_tokens` to cap output length

## Strategy 4: Batching

```
Real-time path:                    Batch path:
User → LLM → Response             Collect requests
  (1 request at a time)              → Send batch (50 at once)
  Full price                         → 50% discount (Anthropic Batch API)
  Low latency                        → Higher latency (up to 24h)
```

**When to batch:**
- Email classification (not time-sensitive)
- Content moderation queue
- Data extraction from documents
- Nightly report generation
- Embedding generation

**Anthropic Batch API:** 50% cost reduction, 24-hour SLA. OpenAI Batch API: similar.

## Strategy 5: Output Optimization

```python
# Don't ask for essays when you need a word
# Bad:
prompt = "Explain whether this email is spam or not and why"
# Response: 200 tokens → $0.003

# Good:
prompt = "Classify as 'spam' or 'not_spam'. One word only."
# Response: 1 token → $0.000015

# Even better: Use logprobs for classification
# No generation needed — check probability of "spam" vs "not_spam"
```

## Strategy 6: Fine-Tuning for Cost

```
Before fine-tuning:
  Model: GPT-4o ($10/1M output)
  Prompt: 2000 tokens (system + few-shot examples)
  Quality: 92% accuracy

After fine-tuning:
  Model: GPT-4o-mini fine-tuned ($2.40/1M output)
  Prompt: 200 tokens (no few-shot needed, behavior learned)
  Quality: 94% accuracy

  Cost reduction: ~85% (cheaper model + shorter prompts)
```

**Fine-tune when:**
- High volume (>10K requests/day for same task)
- Stable task (requirements don't change weekly)
- Have training data (>100 high-quality examples)
- Few-shot examples are large portion of prompt cost

## Cost Monitoring Dashboard

```
┌─────────────────────────────────────────────────────────┐
│                 Cost Dashboard                           │
│                                                         │
│  Daily spend: $847 (↓12% from yesterday)                │
│                                                         │
│  ┌──────────────────────────────────────────┐           │
│  │ By Feature          │ Cost   │ Req/day  │           │
│  │─────────────────────┼────────┼──────────│           │
│  │ Customer support     │ $420   │ 45,000  │           │
│  │ Content generation   │ $280   │ 12,000  │           │
│  │ Code review          │ $95    │ 3,000   │           │
│  │ Classification       │ $52    │ 180,000 │           │
│  └──────────────────────────────────────────┘           │
│                                                         │
│  ┌──────────────────────────────────────────┐           │
│  │ By Model            │ Cost   │ Tokens   │           │
│  │─────────────────────┼────────┼──────────│           │
│  │ Claude Sonnet        │ $520   │ 28M     │           │
│  │ Claude Haiku          │ $180   │ 150M    │           │
│  │ GPT-4o-mini          │ $95    │ 80M     │           │
│  │ text-embedding-3     │ $52    │ 250M    │           │
│  └──────────────────────────────────────────┘           │
│                                                         │
│  Alerts: ⚠️ Content generation cost +34% this week     │
└─────────────────────────────────────────────────────────┘
```

## Common Pitfalls

1. **Not measuring cost per feature.** "We spent $15K on OpenAI" is useless. Track cost per feature, per user, per request.
2. **Optimizing too early.** Get it working first, then optimize. Premature optimization with Haiku when you need Sonnet wastes engineering time.
3. **Ignoring embedding costs.** Embedding 10M documents at $0.02/1M tokens seems cheap until you need to re-embed weekly.
4. **No spending alerts.** A bug that loops LLM calls can burn thousands in hours. Set daily/hourly spend alerts.
5. **Using the biggest model for everything.** 80% of requests don't need GPT-4o or Claude Opus. Measure quality with cheaper models first.

## Practice Problem

**Optimize costs for an AI writing assistant processing 500K requests/day.** Current state:
- All requests go to Claude Sonnet ($0.023/req) = $11,500/day
- 60% are simple edits (grammar, spelling)
- 30% are rewrites (tone, style)
- 10% are creative generation (blog posts, marketing copy)
- Average prompt is 3000 tokens (2000 system + 500 few-shot + 500 user)

Target: Reduce to < $3,000/day without dropping quality below 90% satisfaction.

Consider: model tiering, prompt caching, prompt optimization, batching for non-real-time tasks, fine-tuning ROI calculation.

## Further Reading

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Anthropic Batch API](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing)
- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [LiteLLM Cost Tracking](https://docs.litellm.ai/docs/budget_manager)
