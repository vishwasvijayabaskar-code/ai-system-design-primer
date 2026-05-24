# Practice Problem: Design an AI Customer Support Bot

> Difficulty: ⭐⭐ | Key Concepts: RAG, agents, guardrails, evaluation

## Problem Statement

Design an AI-powered customer support system for an e-commerce company with 50K support tickets per day. The system should handle common inquiries automatically and escalate complex cases to human agents.

## Requirements

### Functional
- Handle common queries: order status, returns, shipping, account issues
- Access customer order history, product catalog, and FAQ knowledge base
- Cite sources in responses (link to help articles)
- Escalate to human agent when confidence is low or customer is frustrated
- Support live chat (streaming) and email (batch) channels
- Multi-language support (English, Spanish, French minimum)

### Non-Functional
- Response time < 3 seconds for chat, < 5 minutes for email
- Accuracy > 90% on common queries (measured by human eval)
- Hallucination rate < 2%
- Handle 500 concurrent chat sessions
- Cost < $0.05 per interaction average

## Constraints
- Must integrate with existing Zendesk ticket system
- Customer PII must never be sent to external LLM providers (SOC2 requirement)
- Cannot make account changes (refunds, cancellations) without human approval for amounts > $100

## Hints

Before reading the solution, consider:

1. **RAG design:** What's your knowledge base? How do you chunk FAQ articles vs order data?
2. **Model selection:** Same model for all queries, or tier by complexity?
3. **Escalation logic:** How do you detect when to hand off to a human?
4. **PII handling:** How do you query order data without sending PII to the LLM?
5. **Evaluation:** How do you measure if the bot is actually helping?

## Solution Walkthrough

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Support System                         │
│                                                         │
│  Chat/Email ──→ Intent Classifier (Haiku) ──→ Router    │
│                                                  │      │
│                      ┌───────────────────────────┤      │
│                      ▼                           ▼      │
│               ┌──────────┐               ┌──────────┐   │
│               │  Simple  │               │ Complex  │   │
│               │  Path    │               │  Path    │   │
│               │ (Haiku)  │               │ (Sonnet) │   │
│               └────┬─────┘               └────┬─────┘   │
│                    │                          │          │
│               ┌────▼─────┐               ┌───▼──────┐   │
│               │   RAG    │               │  Agent   │   │
│               │ FAQ only │               │ RAG +    │   │
│               │          │               │ Tools    │   │
│               └────┬─────┘               └────┬─────┘   │
│                    │                          │          │
│               ┌────▼──────────────────────────▼─────┐   │
│               │         Guardrails Layer            │   │
│               │  PII scrub │ Hallucination check    │   │
│               │  Tone check │ Escalation trigger    │   │
│               └─────────────────────────────────────┘   │
│                              │                          │
│                         Response                        │
└─────────────────────────────────────────────────────────┘
```

### Key Design Decisions

**1. Model Tiering**
- Intent classification: Haiku ($0.001/req) — fast, handles "what category?"
- Simple FAQ: Haiku + RAG ($0.003/req) — 70% of tickets
- Complex issues: Sonnet + RAG + tools ($0.02/req) — 25% of tickets
- Escalation: Human agent — 5% of tickets
- Blended cost: ~$0.007/interaction (well under $0.05 target)

**2. RAG Strategy**
- FAQ articles: Chunk by section (H2 headers), 256-512 tokens, embed with text-embedding-3-small
- Order data: Don't embed — query directly from database, inject structured data into prompt
- Product catalog: Chunk by product, include key attributes as metadata for filtering

**3. PII Protection**
- Customer data stays in your database — never sent to LLM
- Prompt template: "Customer has order #[ORDER_ID], placed [DATE], status: [STATUS]"
- LLM sees anonymized structured data, not raw customer records
- Response template fills in customer name server-side after LLM generates

**4. Escalation Triggers**
- Sentiment detection: angry/frustrated language → escalate
- Confidence: model self-reports < 0.7 confidence → escalate
- Loop detection: > 3 back-and-forth without resolution → escalate
- Explicit request: customer says "talk to a human" → escalate
- Policy: refund > $100, account deletion, legal threats → always escalate

**5. Evaluation**
- Automated: LLM-as-judge scores 10% of responses nightly
- Human: QA team reviews 2% of responses weekly
- Metrics: resolution rate, escalation rate, CSAT score, avg handle time
- A/B test: 50% AI, 50% human for first 2 weeks to baseline quality

### Cost Analysis

```
50K tickets/day:
  70% simple (Haiku):   35,000 × $0.003 = $105
  25% complex (Sonnet):  12,500 × $0.02  = $250
  5% escalated (human):   2,500 × $0     = $0 (human cost separate)
  Embeddings:             50,000 × $0.0001 = $5
  
  Total: ~$360/day = $10,800/month
  vs. 50K human-handled tickets: ~$250K/month
  
  Savings: 95%+
```
