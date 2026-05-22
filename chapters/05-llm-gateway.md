# Chapter 5: LLM Gateway & Routing

## Overview

An LLM Gateway is a proxy layer between your application and model providers. It handles routing, fallbacks, rate limiting, cost tracking, and caching. Every production AI system needs one — calling providers directly is how you get surprise $10K bills and 3am outages.

## Why You Need a Gateway

```
Without gateway:                    With gateway:
App ──→ OpenAI (hardcoded)          App ──→ Gateway ──→ OpenAI
                                                   ├──→ Anthropic
If OpenAI is down → you're down                    ├──→ Google
If costs spike → no visibility                     └──→ Local/Ollama
If rate limited → errors everywhere
                                    Fallbacks, caching, cost tracking,
                                    rate limiting — all in one place.
```

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      LLM Gateway                         │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  Router   │  │  Cache   │  │  Rate    │              │
│  │          │  │          │  │  Limiter │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │              │                    │
│  ┌────▼──────────────▼──────────────▼────┐              │
│  │            Request Pipeline           │              │
│  │                                       │              │
│  │  Validate → Route → Cache Check →     │              │
│  │  Rate Limit → Transform → Send →      │              │
│  │  Log → Return                         │              │
│  └────────────────┬──────────────────────┘              │
│                   │                                      │
│  ┌────────────────▼──────────────────────┐              │
│  │          Provider Adapters            │              │
│  │  ┌────────┐ ┌────────┐ ┌────────┐    │              │
│  │  │OpenAI  │ │Anthropic│ │Google  │    │              │
│  │  └────────┘ └────────┘ └────────┘    │              │
│  │  ┌────────┐ ┌────────┐               │              │
│  │  │Ollama  │ │Custom  │               │              │
│  │  └────────┘ └────────┘               │              │
│  └───────────────────────────────────────┘              │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  Cost    │  │  Metrics │  │  Logs    │              │
│  │ Tracker  │  │          │  │          │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

## Key Features

### 1. Model Routing

Route requests to different models based on task complexity, cost, or latency requirements.

```
Request → Classifier → Simple?  → Haiku ($0.001)
                       Medium?  → Sonnet ($0.01)
                       Complex? → Opus ($0.05)
```

**Routing strategies:**

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| Static | Hardcoded model per endpoint | Simple apps, predictable workloads |
| Header-based | Client specifies model in request | Multi-tenant, developer control |
| Content-based | Classify request complexity, route accordingly | Cost optimization |
| Latency-based | Route to fastest available provider | Real-time applications |
| Cost-based | Route to cheapest model that meets quality bar | Budget-constrained |

### 2. Fallbacks

```
Primary: OpenAI GPT-4o
    │
    ├── 429 (rate limited) ──→ Anthropic Claude Sonnet
    │
    ├── 500 (server error) ──→ Anthropic Claude Sonnet
    │
    ├── Timeout (> 10s) ────→ Anthropic Claude Sonnet
    │
    └── Anthropic fails ───→ Google Gemini Pro
                                │
                                └── All fail ──→ Queue for retry
```

**Key rules:**
- Always have at least 2 providers
- Fallback models should be similar capability (don't fall from GPT-4o to GPT-4o-mini unless acceptable)
- Log every fallback — frequent fallbacks indicate a provider problem
- Set timeouts aggressively (5-10s for streaming, 30s for batch)

### 3. Semantic Caching

```
Request: "What is the capital of France?"
    │
    ▼
Cache lookup (semantic similarity > 0.95)
    │
    ├── HIT: "What's France's capital city?" → Return cached: "Paris"
    │
    └── MISS → Send to LLM → Cache response → Return
```

**Cache strategies:**

| Type | How | Hit Rate | Best For |
|------|-----|----------|----------|
| Exact match | Hash the prompt | Low | Batch processing, identical queries |
| Semantic | Embed prompt, find similar | Medium | User-facing, varied phrasing |
| Prefix | Cache common prompt prefixes | High | Shared system prompts (provider feature) |

**Anthropic prompt caching:** Caches the prefix of your prompt. If 90% of your prompt is the same system prompt + context, you pay full price once, then 90% discount on subsequent calls. Built into the API — no extra infra needed.

### 4. Rate Limiting

```
┌─────────────────────────────────┐
│        Rate Limiter             │
│                                 │
│  Per-user:     10 req/min       │
│  Per-tenant:   100 req/min      │
│  Per-provider: 500 req/min      │
│  Global:       1000 req/min     │
│                                 │
│  Token budget: 1M tokens/day    │
│                                 │
│  If limit hit → queue or reject │
└─────────────────────────────────┘
```

**Why token-based limits matter:** A single request with a 100K context window costs 100x more than a simple question. Rate limit on tokens, not just requests.

### 5. Cost Tracking

```python
# Track per-request costs
cost = (input_tokens * input_price) + (output_tokens * output_price)

# Aggregate by:
# - User / team / tenant
# - Model
# - Feature / endpoint
# - Time period

# Alert when:
# - Daily spend > $X
# - Single request > $Y
# - Spend rate increasing > Z%/hour
```

## Existing Solutions

| Tool | Type | Best For |
|------|------|----------|
| **LiteLLM** | OSS proxy | Unified API for 100+ providers. Most popular |
| **Portkey** | Managed | Enterprise gateway with observability |
| **Helicone** | Managed | Logging + caching focus |
| **Kong AI Gateway** | OSS/Enterprise | Already using Kong for API gateway |
| **Custom** | DIY | Full control, specific requirements |

**Recommendation:** Start with LiteLLM. It's free, handles 100+ providers, and adds fallbacks + cost tracking with minimal config.

## Implementation Pattern: Simple Gateway

```python
# Pseudocode for a minimal gateway
class LLMGateway:
    def __init__(self):
        self.providers = {
            "primary": OpenAIClient(),
            "fallback": AnthropicClient(),
        }
        self.cache = SemanticCache()
        self.rate_limiter = TokenBucketLimiter()
        self.cost_tracker = CostTracker()

    async def complete(self, request):
        # 1. Rate limit check
        if not self.rate_limiter.allow(request.user_id):
            raise RateLimitError()

        # 2. Cache check
        cached = self.cache.get(request.prompt)
        if cached:
            return cached

        # 3. Route to provider
        for provider_name in ["primary", "fallback"]:
            try:
                provider = self.providers[provider_name]
                response = await provider.complete(
                    request,
                    timeout=10
                )

                # 4. Track costs
                self.cost_tracker.log(
                    user=request.user_id,
                    model=request.model,
                    input_tokens=response.usage.input,
                    output_tokens=response.usage.output
                )

                # 5. Cache response
                self.cache.set(request.prompt, response)

                return response

            except (Timeout, RateLimit, ServerError) as e:
                log.warn(f"{provider_name} failed: {e}")
                continue

        raise AllProvidersFailedError()
```

## Common Pitfalls

1. **No fallback provider.** Single provider = single point of failure. OpenAI has had multiple multi-hour outages. Always have a backup.
2. **Caching non-deterministic outputs.** If temperature > 0, cached responses may not match what the model would generate fresh. Only cache at temperature 0 or accept the trade-off.
3. **Not tracking costs per-feature.** "We spent $15K on OpenAI last month" is useless. "The summarization feature costs $8K/month and serves 200 users" is actionable.
4. **Rate limiting only on requests.** A user sending 10 requests with 100K-token contexts costs 100x more than 10 simple requests. Limit on tokens, not just request count.
5. **No timeout on LLM calls.** LLM providers sometimes hang. Without timeouts, your request queue backs up and cascading failures begin.

## Practice Problem

**Design an LLM gateway for a B2B SaaS platform.** Requirements:
- 500 tenants, each with different usage tiers (free: 10K tokens/day, pro: 1M, enterprise: unlimited)
- Must support OpenAI, Anthropic, and self-hosted Llama models
- Automatic fallback when primary provider is down
- Per-tenant cost tracking and billing
- P99 gateway overhead < 50ms (don't add significant latency)
- Semantic caching with tenant isolation (Tenant A's cache shouldn't serve Tenant B)

Consider: routing strategy, cache architecture, rate limiting scheme, cost allocation, monitoring.

## Further Reading

- [LiteLLM Documentation](https://docs.litellm.ai/)
- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Portkey AI Gateway](https://portkey.ai/)
- [OpenAI Rate Limits Guide](https://platform.openai.com/docs/guides/rate-limits)
