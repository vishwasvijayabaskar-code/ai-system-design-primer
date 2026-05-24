# Chapter 12: Scaling Patterns

## Overview

Scaling AI systems is fundamentally different from scaling traditional web services. LLM calls are 100x slower, 1000x more expensive, and rate-limited by providers. This chapter covers the patterns that make AI systems work at scale — horizontal scaling, caching, async processing, and edge deployment.

## The Scaling Challenge

```
Traditional API:                    LLM API:
  Latency: 50ms                      Latency: 2-5 seconds
  Cost: $0.000001/req                 Cost: $0.01-0.10/req
  Rate limit: 10K rps                 Rate limit: 100-1000 rpm
  Stateless                           Stateful (conversation history)
  Deterministic                       Probabilistic
```

## Pattern 1: Async Processing

Most LLM tasks don't need real-time responses.

```
Synchronous (bad for batch):
  User → API → LLM (3s) → Response
  (User waits 3 seconds. Multiply by 10K requests = queue hell)

Asynchronous (good for batch):
  User → API → Queue → Worker → LLM → Store Result
  User → Poll/Webhook → Get Result
  (User gets immediate acknowledgment. Workers process at LLM's pace)
```

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  API     │────→│  Queue   │────→│ Workers  │────→│  Store   │
│  Server  │     │  (SQS/   │     │ (N pods) │     │  (DB/S3) │
│          │     │  Redis)  │     │          │     │          │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
     │                                                    │
     │              ┌──────────────────────┐              │
     └──────────────│  Webhook / Polling   │──────────────┘
                    └──────────────────────┘
```

**Use async for:**
- Document processing
- Email classification/response drafts
- Content moderation queues
- Report generation
- Embedding generation
- Batch evaluations

## Pattern 2: Streaming

For user-facing responses, stream tokens as they're generated.

```
Without streaming:
  User clicks send → 3 seconds of nothing → full response appears
  (Users think it's broken)

With streaming:
  User clicks send → 200ms → first word appears → tokens stream in
  (Users see progress, perceived latency drops 10x)
```

```python
# Server-Sent Events (SSE) pattern
async def stream_response(request):
    async def generate():
        async for chunk in llm.stream(request.prompt):
            yield f"data: {json.dumps({'text': chunk})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

**Key metrics:**
- **TTFT (Time to First Token):** Should be < 500ms
- **Inter-token latency:** Should be < 50ms (feels smooth)
- **Total time:** Less important when streaming (user reads as it generates)

## Pattern 3: Horizontal Scaling

```
┌─────────────────────────────────────────────────────────┐
│                  Load Balancer                           │
│                                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│  │Worker 1 │  │Worker 2 │  │Worker 3 │  │Worker N │   │
│  │         │  │         │  │         │  │         │   │
│  │ LLM     │  │ LLM     │  │ LLM     │  │ LLM     │   │
│  │ Client  │  │ Client  │  │ Client  │  │ Client  │   │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘   │
│       │            │            │            │          │
│  ┌────▼────────────▼────────────▼────────────▼────┐    │
│  │              LLM Gateway                       │    │
│  │    (rate limiting, fallbacks, cost tracking)    │    │
│  └────────────────────┬───────────────────────────┘    │
│                       │                                 │
│            ┌──────────┼──────────┐                     │
│            ▼          ▼          ▼                     │
│        OpenAI    Anthropic    Local                    │
└─────────────────────────────────────────────────────────┘
```

**Scaling constraints:**
- Provider rate limits (not your infra) are usually the bottleneck
- Scale workers to match rate limits, not beyond
- Use multiple providers to increase effective rate limit
- Auto-scale based on queue depth, not CPU

## Pattern 4: Caching Layers

```
┌──────────────────────────────────────────┐
│           Cache Hierarchy                │
│                                          │
│  L1: Exact match cache (Redis)           │
│      Hit rate: 5-15%                     │
│      Latency: 1ms                        │
│                                          │
│  L2: Semantic cache (vector similarity)  │
│      Hit rate: 10-30%                    │
│      Latency: 10-50ms                    │
│                                          │
│  L3: Prompt prefix cache (provider)      │
│      Hit rate: 60-90%                    │
│      Cost reduction: 50-90%             │
│                                          │
│  L4: No cache → full LLM call           │
│      Latency: 1-5 seconds               │
│      Full cost                           │
└──────────────────────────────────────────┘
```

## Pattern 5: Multi-Provider Resilience

```
┌──────────┐
│ Request  │
└────┬─────┘
     │
┌────▼─────────────────────────────────────┐
│           Provider Router                │
│                                          │
│  Priority 1: OpenAI GPT-4o              │
│    ├── Available? → Send                 │
│    └── Rate limited / 500 → Next        │
│                                          │
│  Priority 2: Anthropic Claude Sonnet    │
│    ├── Available? → Send                 │
│    └── Rate limited / 500 → Next        │
│                                          │
│  Priority 3: Google Gemini Pro          │
│    ├── Available? → Send                 │
│    └── All down → Queue for retry       │
│                                          │
│  Circuit breaker per provider:           │
│    5 failures in 60s → skip for 5min     │
└──────────────────────────────────────────┘
```

## Pattern 6: Edge Deployment

Run smaller models closer to users for latency-sensitive tasks.

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Edge    │     │  Edge    │     │  Edge    │
│  US-East │     │  EU-West │     │  AP-SE   │
│  Llama   │     │  Llama   │     │  Llama   │
│  3.2 8B  │     │  3.2 8B  │     │  3.2 8B  │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
     └────────────────┼────────────────┘
                      │
              ┌───────▼───────┐
              │   Central     │
              │   Claude/GPT  │
              │   (complex    │
              │    tasks)     │
              └───────────────┘
```

**Edge for:** Classification, simple extraction, intent detection, content filtering
**Central for:** Complex reasoning, long generation, agent tasks

## Pattern 7: Request Coalescing

Combine similar concurrent requests into one LLM call.

```
User A: "What's your return policy?"  ─┐
User B: "How do I return an item?"     ├──→ One LLM call → Serve both
User C: "Return policy?"              ─┘

(Semantic similarity > 0.95 within 2-second window)
```

Savings: 3 LLM calls → 1 LLM call. Works well for FAQ-like traffic.

## Scaling Checklist

```
□ Streaming enabled for all user-facing responses
□ Async processing for batch/non-real-time tasks
□ At least 2 LLM providers configured with fallback
□ Caching layer (exact + semantic + provider prefix)
□ Rate limiting per user/tenant/global
□ Auto-scaling based on queue depth
□ Circuit breakers per provider
□ Cost alerts and spending caps
□ Horizontal scaling for workers
□ Monitoring: latency P50/P99, cost, quality, queue depth
```

## Common Pitfalls

1. **Scaling compute when the bottleneck is rate limits.** Adding more workers doesn't help if the provider limits you to 1000 RPM. Add providers or negotiate higher limits.
2. **No backpressure.** Without queue depth limits, a traffic spike fills your queue with hours of backlog. Set max queue size and reject/shed load.
3. **Synchronous LLM calls in request path.** A 3-second LLM call holding a connection thread = 100 threads for 33 concurrent requests. Use async I/O.
4. **Not pre-warming caches.** Common queries should be cached before launch, not learned cold.
5. **Over-scaling for peaks.** If traffic peaks 3x during business hours, use auto-scaling. Don't provision for peak 24/7.

## Practice Problem

**Design a scalable AI-powered search engine for an e-commerce platform.** Requirements:
- 10M products
- 1M searches/day (peak: 100 searches/second)
- Search results in < 500ms (P99)
- Natural language queries ("red dress under $50 for a wedding")
- Personalized results based on user history
- Must handle Black Friday (10x normal traffic)

Consider: embedding strategy, vector DB scaling, caching tiers, async vs sync paths, edge deployment for latency, cost at scale.

## Further Reading

- [Anthropic Rate Limits](https://docs.anthropic.com/en/docs/about-claude/rate-limits)
- [vLLM: High-Throughput LLM Serving](https://docs.vllm.ai/)
- [Ollama for Local Deployment](https://ollama.com/)
- [Ray Serve for LLM Scaling](https://docs.ray.io/en/latest/serve/)
