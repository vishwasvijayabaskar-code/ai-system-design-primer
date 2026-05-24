# Practice Problem: Design an LLM-Powered Search Engine

> Difficulty: ⭐⭐⭐ | Key Concepts: Hybrid search, embeddings, query understanding

## Problem Statement

Design a search engine for a developer documentation platform (like Stripe Docs or MDN). Users type natural language queries and get AI-synthesized answers with links to relevant documentation pages.

## Requirements

### Functional
- Natural language queries ("how do I handle webhooks in Python?")
- AI-generated answer with code examples extracted from docs
- Link to source documentation pages
- Autocomplete / query suggestions
- Search analytics (what are users searching for?)
- Support 50K documentation pages across 200 products

### Non-Functional
- Search results in < 1 second (P99)
- AI answer in < 3 seconds (streaming)
- Handle 10K queries per minute at peak
- Index updates within 10 minutes of doc changes
- Cost < $0.02 per query average

## Constraints
- Documentation changes 100+ times per day (CI/CD deploys)
- Some docs are versioned (v1, v2, v3 of same API)
- Code examples must be syntactically correct (no hallucinated code)
- Must work for 15 programming languages

## Hints

1. How do you handle versioned docs? User searching "auth" should get their version, not all versions.
2. Code examples in docs — chunk by function or by page?
3. $0.02/query at 10K QPM — which model tier?
4. How do you prevent hallucinated code examples?

## Solution Walkthrough

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Search System                          │
│                                                         │
│  Query ──→ Understanding ──→ Retrieval ──→ Generation   │
│             │                  │              │          │
│          ┌──▼───┐        ┌────▼────┐    ┌────▼────┐    │
│          │Intent│        │Hybrid   │    │ Haiku   │    │
│          │Detect│        │Search   │    │+context │    │
│          │+Query│        │Vec+BM25 │    │ stream  │    │
│          │Expand│        │+filter  │    │         │    │
│          └──────┘        └─────────┘    └─────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Key Decisions

**Query Understanding:** Haiku classifies intent (API reference vs tutorial vs troubleshooting) and expands query with synonyms. "webhooks python" → "webhook endpoint handler Python Flask Django event listener". Cost: $0.001/query.

**Hybrid Search:** Qdrant with BM25 plugin. Vector search catches semantic matches, BM25 catches exact API names (`stripe.PaymentIntent.create`). Filter by product + version from query context.

**Chunking Strategy:**
- Prose sections: 512 tokens, sentence-boundary
- Code blocks: Keep entire code block as one chunk (never split mid-function)
- API reference: One chunk per endpoint/method
- Store metadata: product, version, language, section type

**Answer Generation:** Haiku (not Sonnet — cost constraint). System prompt enforces: only use provided context, include code from sources verbatim (don't generate new code), cite source URL.

**Hallucination Prevention:**
- Code examples: Extract verbatim from retrieved chunks, don't generate
- Facts: Every claim must reference a retrieved chunk
- Validation: Post-generation check — does every code block appear in sources?

**Cost:** Query understanding ($0.001) + embedding ($0.0001) + Haiku answer ($0.003) = ~$0.004/query. Well under $0.02.
