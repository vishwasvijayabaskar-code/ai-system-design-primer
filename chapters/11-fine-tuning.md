# Chapter 11: Fine-Tuning vs Prompting

## Overview

Fine-tuning changes a model's weights to specialize it for your task. Prompting gives the model instructions at inference time. The decision between them is one of the most common — and most misunderstood — choices in AI system design.

## Decision Framework

```
┌─────────────────────────────────────────────┐
│         Should You Fine-Tune?               │
│                                             │
│  Need recent/dynamic data?                  │
│    YES → RAG (not fine-tuning)              │
│                                             │
│  Need to change model behavior/style?       │
│    YES → Consider fine-tuning               │
│                                             │
│  Have < 100 training examples?              │
│    YES → Prompting (not enough data)        │
│                                             │
│  Task changes frequently?                   │
│    YES → Prompting (fine-tuning is slow)    │
│                                             │
│  > 10K requests/day, same task?             │
│    YES → Fine-tune (cost savings)           │
│                                             │
│  Latency critical, no room for RAG?         │
│    YES → Fine-tune (no retrieval overhead)  │
│                                             │
│  Still unsure?                              │
│    → Start with prompting. Always.          │
└─────────────────────────────────────────────┘
```

## Comparison

| Factor | Prompting | Fine-Tuning |
|--------|-----------|-------------|
| Setup time | Minutes | Days-weeks |
| Data needed | 0-10 examples | 100-10,000+ examples |
| Cost to start | $0 | $50-$5,000+ |
| Iteration speed | Seconds | Hours |
| Ongoing cost | Higher (longer prompts) | Lower (shorter prompts) |
| Flexibility | Change anytime | Retrain to change |
| Knowledge update | Change prompt/RAG | Retrain model |
| Max quality | Limited by prompt | Can exceed prompting |

## When Fine-Tuning Wins

### 1. Consistent Style/Format

```
Task: Convert natural language to SQL

Prompting approach (needs 5+ examples in every prompt):
  "Convert to SQL. Examples:
   'Show all users' → SELECT * FROM users;
   'Count orders today' → SELECT COUNT(*) FROM orders WHERE date = CURRENT_DATE;
   ..."
  Cost: 500 tokens of examples per request

Fine-tuned approach (learned from 1000 training pairs):
  "Convert to SQL: 'Show all users'"
  Cost: 20 tokens per request
  
  Savings: 96% fewer input tokens
```

### 2. Domain Expertise

```
Before: GPT-4o-mini doesn't know your company's internal terminology
After:  Fine-tuned GPT-4o-mini understands "CSAT", "P0 incident",
        "golden path", and your specific product names
```

### 3. Cost Optimization at Scale

```
Current:  Claude Sonnet + 2000-token system prompt
          100K req/day × $0.023 = $2,300/day

Fine-tuned: Claude Haiku-equivalent + 200-token prompt
            100K req/day × $0.002 = $200/day

Annual savings: $766,500
Fine-tuning cost: ~$500
ROI: obvious
```

## Fine-Tuning Pipeline

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Collect  │→│ Clean &  │→│  Train   │→│ Evaluate │→│ Deploy  │
│  Data    │  │ Format   │  │          │  │          │  │          │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

### Step 1: Data Collection

**Sources:**
- Production logs (real user interactions)
- Human-written ideal responses
- Expert annotations
- Synthetic data (generate with a stronger model, filter for quality)

**Minimum data:**

| Task Type | Minimum Examples | Ideal |
|-----------|-----------------|-------|
| Classification | 50 per class | 200+ per class |
| Text generation | 100 | 500-1000 |
| Conversation | 200 conversations | 1000+ |
| Code generation | 200 | 500-1000 |

### Step 2: Data Format

```jsonl
{"messages": [
  {"role": "system", "content": "You are a support agent for Acme Corp."},
  {"role": "user", "content": "How do I reset my password?"},
  {"role": "assistant", "content": "Go to Settings > Security > Reset Password. You'll receive a confirmation email within 2 minutes."}
]}
```

**Data quality rules:**
- Every example should be one you'd want the model to produce
- Remove contradictory examples
- Balance classes (don't have 900 positive, 100 negative)
- Include edge cases (10-20% of dataset)

### Step 3: Training

```python
# OpenAI fine-tuning (simplified)
client.fine_tuning.jobs.create(
    training_file="file-abc123",
    model="gpt-4o-mini-2024-07-18",
    hyperparameters={
        "n_epochs": 3,
        "learning_rate_multiplier": 1.0,
        "batch_size": 8
    }
)
```

### Step 4: Evaluation

```
                    ┌───────────────────────┐
                    │    Eval Dataset        │
                    │    (held out, 20%)     │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────┐
                    │  Run both models      │
                    │  Base vs Fine-tuned   │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────┐
                    │  Compare metrics      │
                    │  Accuracy: 85% → 94%  │
                    │  Latency: same        │
                    │  Cost: -80%           │
                    └───────────────────────┘
```

**Critical:** Always compare against the base model + prompting. If fine-tuning doesn't beat prompting significantly, it's not worth the maintenance cost.

## The Combined Approach

Best systems use BOTH prompting and fine-tuning:

```
Fine-tuned model (learned: style, format, domain knowledge)
    +
Prompt (provides: task-specific instructions, dynamic context)
    +
RAG (provides: current data, specific documents)
    =
Best of all worlds
```

## Common Pitfalls

1. **Fine-tuning before trying prompting.** Always start with prompting. Fine-tuning is a last resort optimization, not a first step.
2. **Too little data.** 20 examples won't teach a model anything useful. Need 100+ minimum, ideally 500+.
3. **Not holding out eval data.** Training on 100% of your data means you can't measure if fine-tuning actually helped.
4. **Forgetting about updates.** Fine-tuned models don't learn new information. You need to retrain when your domain changes.
5. **Fine-tuning for knowledge.** Fine-tuning teaches behavior, not facts. For factual knowledge, use RAG.
6. **Not monitoring drift.** Fine-tuned model quality degrades as user behavior changes. Monitor and retrain quarterly.

## Practice Problem

**Decide the right approach for each scenario:**
1. E-commerce chatbot that needs to know current inventory levels
2. Legal document classifier (10 document types, 50K labeled examples)
3. Internal tool that converts Slack messages to Jira tickets in your team's format
4. Medical triage system that needs to follow updated WHO guidelines
5. Code review bot that should match your team's style guide

For each: prompting, fine-tuning, RAG, or combination? Justify with cost, quality, and maintenance trade-offs.

## Further Reading

- [OpenAI Fine-Tuning Guide](https://platform.openai.com/docs/guides/fine-tuning)
- [Anthropic Fine-Tuning](https://docs.anthropic.com/en/docs/build-with-claude/fine-tuning)
- [When to Fine-Tune (Hamel Husain)](https://hamel.dev/blog/posts/fine-tuning/)
- [LoRA: Low-Rank Adaptation](https://arxiv.org/abs/2106.09685)
