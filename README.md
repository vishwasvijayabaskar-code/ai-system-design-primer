# AI System Design Primer 🧠

> Learn how to design large-scale AI systems. Prep for the AI system design interview.

<p align="center">
  <a href="#table-of-contents">Chapters</a> •
  <a href="#how-to-use">How to Use</a> •
  <a href="#contributing">Contributing</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/chapters-12-6366f1?style=flat-square" alt="Chapters" />
  <img src="https://img.shields.io/badge/status-actively%20maintained-22c55e?style=flat-square" alt="Status" />
  <img src="https://img.shields.io/badge/license-MIT-blue?style=flat-square" alt="License" />
</p>

---

## Why This Exists

The original [system-design-primer](https://github.com/donnemartin/system-design-primer) (298k+ stars) is the gold standard for traditional system design — but it was written before the LLM era.

**AI system design is fundamentally different:**
- You're designing around probabilistic models, not deterministic APIs
- Latency profiles are 100x worse (seconds, not milliseconds)
- Costs scale with tokens, not just compute
- Evaluation is subjective and domain-specific
- Data pipelines include embeddings, vector stores, and retrieval
- Failure modes include hallucination, not just downtime

This primer fills that gap.

---

## How to Use

- **Interview prep** — work through the practice problems in each chapter
- **Reference** — bookmark specific chapters for your current project
- **Learning** — read front-to-back for a complete AI systems education

**Estimated reading time:** ~4 hours for all chapters

---

## Table of Contents

### Fundamentals

| # | Chapter | Description |
|---|---------|-------------|
| 1 | [LLM Fundamentals](chapters/01-llm-fundamentals.md) | Tokens, context windows, temperature, inference vs training, model families |
| 2 | [Prompt Engineering at Scale](chapters/02-prompt-engineering.md) | System prompts, few-shot, chain-of-thought, prompt versioning, A/B testing |
| 3 | [RAG Architecture](chapters/03-rag-architecture.md) | Retrieval-augmented generation — chunking, embedding, retrieval, reranking |
| 4 | [Vector Databases](chapters/04-vector-databases.md) | HNSW, IVF, PQ. Choosing between Pinecone, Qdrant, Chroma, pgvector |

### Production Systems

| # | Chapter | Description |
|---|---------|-------------|
| 5 | [LLM Gateway & Routing](chapters/05-llm-gateway.md) | Multi-model routing, fallbacks, cost optimization, rate limiting |
| 6 | [Agent Architecture](chapters/06-agent-architecture.md) | Tool use, ReAct, planning, multi-agent coordination, state machines |
| 7 | [Evaluation & Testing](chapters/07-evaluation.md) | LLM-as-judge, human eval, regression testing, CI/CD for prompts |
| 8 | [Guardrails & Safety](chapters/08-guardrails.md) | Content filtering, hallucination detection, PII redaction, jailbreak prevention |

### Scaling & Operations

| # | Chapter | Description |
|---|---------|-------------|
| 9 | [Cost Optimization](chapters/09-cost-optimization.md) | Prompt caching, model tiering, batching, when to fine-tune vs prompt |
| 10 | [Observability](chapters/10-observability.md) | Tracing, logging, metrics, debugging LLM pipelines in production |
| 11 | [Fine-Tuning vs Prompting](chapters/11-fine-tuning.md) | When to fine-tune, data preparation, evaluation, deployment |
| 12 | [Scaling Patterns](chapters/12-scaling-patterns.md) | Horizontal scaling, caching strategies, async processing, edge deployment |

### Practice Problems

| Problem | Difficulty | Key Concepts |
|---------|-----------|--------------|
| [Design an AI Customer Support Bot](chapters/problems/customer-support-bot.md) | ⭐⭐ | RAG, agents, guardrails, evaluation |
| [Design a Code Review Agent](chapters/problems/code-review-agent.md) | ⭐⭐⭐ | Multi-step agents, tool use, context management |
| [Design a RAG Pipeline for 10M Documents](chapters/problems/rag-at-scale.md) | ⭐⭐⭐ | Vector DBs, chunking strategies, reranking |
| [Design an LLM-Powered Search Engine](chapters/problems/llm-search.md) | ⭐⭐⭐ | Hybrid search, embeddings, query understanding |
| [Design a Multi-Agent Coding System](chapters/problems/multi-agent-coding.md) | ⭐⭐⭐⭐ | Agent orchestration, planning, tool use, evaluation |
| [Design a Real-Time AI Voice Assistant](chapters/problems/voice-assistant.md) | ⭐⭐⭐⭐ | Streaming, latency, speech-to-text, TTS, state management |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        AI Application                           │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Guardrails│  │  Prompt  │  │  Agent   │  │  Evaluation  │   │
│  │ & Safety  │  │ Manager  │  │ Runtime  │  │  Pipeline    │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│       │              │             │               │            │
├───────┴──────────────┴─────────────┴───────────────┴────────────┤
│                      LLM Gateway / Router                       │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────────────┐ │
│  │ Routing  │  │ Fallback │  │ Caching  │  │ Cost Tracking    │ │
│  └────┬────┘  └────┬─────┘  └────┬────┘  └───────┬──────────┘ │
├───────┴─────────────┴─────────────┴───────────────┴─────────────┤
│                      Model Providers                            │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ OpenAI  │  │Anthropic │  │  Google  │  │  Local   │        │
│  │ GPT-4o  │  │ Claude   │  │ Gemini   │  │ Ollama   │        │
│  └─────────┘  └──────────┘  └──────────┘  └──────────┘        │
├─────────────────────────────────────────────────────────────────┤
│                      Data Layer                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Vector DB│  │ Document │  │  Cache   │  │ Logging  │       │
│  │ Qdrant   │  │  Store   │  │  Redis   │  │ Langfuse │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Concepts Quick Reference

### LLM Selection Matrix

| Need | Model | Why |
|------|-------|-----|
| Best quality | Claude Opus 4 / GPT-4o | Highest reasoning ability |
| Fast + cheap | Claude Haiku / GPT-4o-mini | 10x cheaper, good enough for most tasks |
| Code generation | Claude Sonnet 4 | Best code quality benchmarks |
| Local / private | Llama 3.2 via Ollama | No data leaves your machine |
| Embeddings | text-embedding-3-small | Best price/performance for search |

### Cost Comparison (per 1M tokens, approx.)

| Model | Input | Output |
|-------|-------|--------|
| GPT-4o | $2.50 | $10.00 |
| GPT-4o-mini | $0.15 | $0.60 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| Claude Haiku 4 | $0.25 | $1.25 |
| Gemini 2.5 Flash | $0.15 | $0.60 |
| Llama 3.2 (local) | Free | Free |

### RAG vs Fine-Tuning Decision Tree

```
Need dynamic/recent data? → RAG
Need to change model behavior? → Fine-tune
Need domain knowledge? → RAG first, fine-tune if RAG isn't enough
Budget < $1000? → RAG (fine-tuning is expensive)
Latency critical? → Fine-tune (no retrieval overhead)
Data changes frequently? → RAG (fine-tuning requires retraining)
```

---

## Contributing

This is a living document. Contributions welcome!

**How to contribute:**
1. Fork the repo
2. Add or improve a chapter
3. Include diagrams (ASCII art preferred for GitHub rendering)
4. Submit a PR

**Chapter template:**
```markdown
# Chapter Title

## Overview (2-3 sentences)
## Key Concepts
## Architecture Diagram
## Implementation Patterns
## Common Pitfalls
## Practice Problem
## Further Reading
```

---

## Roadmap

- [x] README with architecture overview
- [x] Table of contents with all chapters
- [x] Chapter 1: LLM Fundamentals
- [x] Chapter 2: Prompt Engineering at Scale
- [x] Chapter 3: RAG Architecture
- [x] Chapter 4: Vector Databases
- [x] Chapter 5: LLM Gateway & Routing
- [x] Chapter 6: Agent Architecture
- [x] Chapter 7: Evaluation & Testing
- [x] Chapter 8: Guardrails & Safety
- [x] Chapter 9: Cost Optimization
- [x] Chapter 10: Observability
- [x] Chapter 11: Fine-Tuning vs Prompting
- [x] Chapter 12: Scaling Patterns
- [ ] Practice Problems (6 total)
- [ ] Anki flashcard deck

---

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=vishwasvijayabaskar-code/ai-system-design-primer&type=Date)](https://star-history.com/#vishwasvijayabaskar-code/ai-system-design-primer&Date)

---

**Maintained by:** [@vishwasvijayabaskar-code](https://github.com/vishwasvijayabaskar-code) | **Last updated:** May 2026
