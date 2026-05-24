# Chapter 10: Observability

## Overview

LLM systems fail silently. A traditional API returns a 500 error — you know it's broken. An LLM returns a confident, plausible, completely wrong answer — and you have no idea until a user complains. Observability for AI systems means tracing every step from input to output, measuring quality over time, and catching degradation before users do.

## What to Observe

```
Traditional API:          LLM System:
- Latency                 - Latency (TTFT + generation)
- Error rate              - Error rate + hallucination rate
- Throughput              - Token usage + cost
                          - Response quality (scores)
                          - Retrieval quality (for RAG)
                          - Tool call success rate (for agents)
                          - Prompt version performance
                          - User feedback (thumbs up/down)
```

## Tracing

Every LLM request should produce a trace — a structured log of everything that happened.

```
Trace: req_abc123
├── Span: query_rewrite (12ms)
│   ├── Input: "how do i fix it"
│   └── Output: "How to fix authentication error in login endpoint"
├── Span: embedding (45ms)
│   ├── Model: text-embedding-3-small
│   └── Tokens: 15
├── Span: retrieval (23ms)
│   ├── Source: qdrant
│   ├── Results: 5 chunks
│   └── Top score: 0.89
├── Span: reranking (180ms)
│   ├── Model: cohere-rerank-3
│   ├── Input: 5 chunks
│   └── Output: 3 chunks (reranked)
├── Span: llm_call (2,340ms)
│   ├── Model: claude-sonnet-4-20250514
│   ├── Input tokens: 3,200
│   ├── Output tokens: 450
│   ├── Cost: $0.016
│   ├── TTFT: 280ms
│   └── Prompt version: support-v2.3
├── Span: guardrails (35ms)
│   ├── PII check: pass
│   ├── Toxicity: 0.02
│   └── Hallucination: not_detected
└── Total: 2,635ms | Cost: $0.017 | Quality: pending
```

## Key Metrics

### Latency Metrics

| Metric | What | Target |
|--------|------|--------|
| TTFT | Time to first token | < 500ms |
| Total latency | Full response time | < 3s for streaming |
| P50 / P99 | Percentile latency | P99 < 5s |
| Gateway overhead | Time spent not in LLM | < 100ms |

### Quality Metrics

| Metric | How to Measure | Alert On |
|--------|---------------|----------|
| User satisfaction | Thumbs up/down ratio | < 80% positive |
| Hallucination rate | LLM-as-judge or source check | > 5% |
| Retrieval relevance | Reranker score, click-through | Avg score < 0.7 |
| Task completion | Did the agent finish? | < 90% completion |
| Escalation rate | Handed to human | > 15% |

### Cost Metrics

| Metric | Granularity | Alert On |
|--------|------------|----------|
| Cost per request | Per-request | Single request > $1 |
| Daily spend | Per-day | > 120% of daily budget |
| Cost per feature | Per-endpoint | Unexpected increase > 20% |
| Token waste | Input tokens that could be cached | Cache miss rate > 50% |

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                Observability Stack                       │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  Trace   │  │  Metrics │  │   Logs   │              │
│  │ Collector│  │ Collector│  │ Collector│              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │              │                    │
│  ┌────▼──────────────▼──────────────▼────┐              │
│  │           Storage Layer               │              │
│  │  Traces: Langfuse / Jaeger            │              │
│  │  Metrics: Prometheus / DataDog        │              │
│  │  Logs: Elasticsearch / Loki           │              │
│  └───────────────┬───────────────────────┘              │
│                  │                                       │
│  ┌───────────────▼───────────────────────┐              │
│  │          Dashboard & Alerts           │              │
│  │  - Real-time quality monitoring       │              │
│  │  - Cost tracking                      │              │
│  │  - Latency percentiles               │              │
│  │  - Regression detection               │              │
│  │  - PagerDuty / Slack alerts           │              │
│  └───────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────┘
```

## Tools

| Tool | Type | Best For |
|------|------|----------|
| **Langfuse** | OSS | LLM-specific tracing, evals, prompt management |
| **LangSmith** | Managed | LangChain ecosystem, playground |
| **Arize Phoenix** | OSS | ML + LLM observability |
| **Helicone** | Managed | Simple logging + caching |
| **OpenTelemetry** | OSS | Standard tracing (extend for LLMs) |
| **Datadog LLM** | Managed | Already using Datadog |

**Recommendation:** Langfuse for LLM-specific observability (traces, evals, prompt versioning). OpenTelemetry for infrastructure-level tracing. They complement each other.

## Debugging LLM Failures

```
User reports: "The bot gave me wrong information about my order"

Debug flow:
1. Find the trace by user ID + timestamp
2. Check retrieval: Did RAG return relevant chunks?
   → If not: chunking or embedding problem
3. Check prompt: Was the context correctly assembled?
   → If not: prompt template bug
4. Check LLM output: Did it hallucinate despite good context?
   → If yes: model issue, consider stronger model or better instructions
5. Check guardrails: Should this have been caught?
   → If yes: improve output validation
```

## Common Pitfalls

1. **No tracing.** When something goes wrong, you're debugging blind. Add tracing from day 1 — it's 10 lines of code with Langfuse.
2. **Only logging errors.** Successful but low-quality responses are invisible without quality tracking. Log everything, score a sample.
3. **Not correlating cost with quality.** Cheaper model might be fine. But you won't know unless you track both cost and quality together.
4. **Alerting on averages.** Average latency is 500ms, but P99 is 15s. Track percentiles.
5. **Not tracking prompt versions.** "Quality dropped last week." What changed? If you version prompts and correlate with quality metrics, you can pinpoint the cause.

## Practice Problem

**Design an observability system for a RAG-powered legal research assistant.** Requirements:
- Lawyers use it to find relevant case law
- Must track: retrieval accuracy, citation correctness, response latency
- Must detect when the system cites non-existent cases (hallucination)
- Must support A/B testing of different retrieval strategies
- Compliance: all traces must be retained for 7 years
- Budget: minimize observability costs (it's a startup)

Consider: what to trace, what to sample, storage strategy for long retention, alerting thresholds, dashboard design.

## Further Reading

- [Langfuse Documentation](https://langfuse.com/docs)
- [OpenTelemetry for LLMs](https://opentelemetry.io/)
- [Arize Phoenix Quickstart](https://docs.arize.com/phoenix)
- [Hamel's Guide to LLM Monitoring](https://hamel.dev/)
