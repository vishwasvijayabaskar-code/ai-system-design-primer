# Chapter 7: Evaluation & Testing

## Overview

You can't improve what you don't measure. LLM evaluation is fundamentally different from traditional software testing — outputs are probabilistic, quality is subjective, and there's no single "correct answer" for most tasks. This chapter covers how to build evaluation systems that actually work.

## Why LLM Evaluation Is Hard

```
Traditional software:    f(input) → deterministic output → assert equals expected
LLM:                     f(input) → probabilistic output → ??? how to judge ???
```

**The core problem:** For "Summarize this article," there are thousands of valid summaries. You can't use `assertEqual`.

## Evaluation Methods

### 1. Exact Match

```python
# Only works for classification, extraction, yes/no
assert model_output == "positive"
assert extracted_email == "user@example.com"
```

**Use for:** Classification, entity extraction, structured output, factual Q&A with known answers.

### 2. LLM-as-Judge

Use a stronger model to evaluate a weaker model's output.

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Input   │────→│ Model    │────→│  Output  │
│          │     │ (tested) │     │          │
└──────────┘     └──────────┘     └────┬─────┘
                                       │
                                       ▼
                                 ┌──────────┐
                                 │  Judge   │
                                 │  Model   │
                                 │ (GPT-4o) │
                                 └────┬─────┘
                                      │
                                      ▼
                                 Score: 4/5
                                 Reason: "Accurate but
                                 missed key detail about..."
```

**Judge prompt example:**

```
Rate the following response on a scale of 1-5 for:
- Accuracy: Does it contain factual errors?
- Completeness: Does it address all parts of the question?
- Clarity: Is it well-written and easy to understand?

Question: {question}
Reference answer: {reference}
Model response: {response}

Return JSON: {"accuracy": N, "completeness": N, "clarity": N, "reasoning": "..."}
```

**Pitfalls of LLM-as-judge:**
- Position bias (prefers first option in A/B comparisons)
- Verbosity bias (prefers longer responses)
- Self-bias (GPT-4 rates GPT-4 outputs higher)
- **Mitigation:** Randomize order, use multiple judges, calibrate against human ratings

### 3. Human Evaluation

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Output A │────→│  Human   │────→│ A wins   │
│ Output B │────→│ Reviewer │────→│ B wins   │
│          │     │          │     │ Tie      │
└──────────┘     └──────────┘     └──────────┘
```

**When to use:** Final quality bar, subjective tasks (creative writing, tone), safety evaluation, calibrating LLM-as-judge.

**Cost:** Expensive and slow. Use for validation, not CI/CD.

### 4. Reference-Based Metrics

| Metric | What It Measures | Best For |
|--------|-----------------|----------|
| BLEU | N-gram overlap with reference | Translation |
| ROUGE | Recall of reference content | Summarization |
| BERTScore | Semantic similarity to reference | General text quality |
| Exact Match | Character-level equality | Extraction, classification |

**Warning:** BLEU/ROUGE correlate poorly with human judgment for open-ended tasks. Use LLM-as-judge instead.

## Evaluation Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 Evaluation Pipeline                       │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  Test    │  │  Model   │  │  Judge   │              │
│  │  Cases   │  │  Runner  │  │  Runner  │              │
│  │ (dataset)│  │          │  │          │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │              │                    │
│  ┌────▼──────────────▼──────────────▼────┐              │
│  │            Eval Loop                  │              │
│  │                                       │              │
│  │  For each test case:                  │              │
│  │    1. Run model on input              │              │
│  │    2. Score with judge(s)             │              │
│  │    3. Log results                     │              │
│  └───────────────┬───────────────────────┘              │
│                  │                                       │
│  ┌───────────────▼───────────────────────┐              │
│  │          Results Dashboard            │              │
│  │  - Accuracy by category               │              │
│  │  - Regression detection               │              │
│  │  - Cost per eval run                  │              │
│  │  - Score distribution                 │              │
│  └───────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────┘
```

## Building an Eval Dataset

**Start small, grow organically:**

```
Week 1:   20 hand-written test cases (golden dataset)
Week 4:   50 cases (add real failures from production)
Week 8:   100 cases (add edge cases found by users)
Month 6:  500+ cases (comprehensive coverage)
```

**Test case format:**

```json
{
  "id": "billing-refund-001",
  "category": "billing",
  "input": "I was charged twice for my subscription",
  "expected_behavior": "Acknowledge the issue, look up the account, offer a refund",
  "expected_tools": ["search_orders", "issue_refund"],
  "forbidden_content": ["I cannot help", "contact support"],
  "difficulty": "medium"
}
```

## CI/CD for Prompts

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ PR with  │────→│  CI runs │────→│ Compare  │────→│ Merge or │
│ prompt   │     │  eval    │     │ vs base  │     │ block    │
│ change   │     │  suite   │     │ scores   │     │          │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                                       │
                                  Score dropped
                                  > 5%? → Block PR
```

**Practical setup:**
1. Keep eval dataset in repo (JSON/YAML)
2. Run eval on every PR that touches prompts
3. Compare scores against main branch baseline
4. Block merge if accuracy drops > threshold
5. Track cost of eval runs (they use LLM calls too)

## Tools

| Tool | Type | Best For |
|------|------|----------|
| **Promptfoo** | OSS | CLI-first eval. CI/CD integration. Best DX |
| **RAGAS** | OSS | RAG-specific evaluation |
| **Langfuse** | OSS | Tracing + eval combined |
| **Braintrust** | Managed | Prompt playground + eval |
| **Arize Phoenix** | OSS | ML observability + LLM eval |

## Common Pitfalls

1. **No eval at all.** "We tested it manually and it seemed good." Ship it, regret it. Build evals from day 1.
2. **Too few test cases.** 5 test cases catch nothing. Need 50+ for basic coverage, 200+ for production confidence.
3. **Not testing edge cases.** Happy path works fine. Test: empty input, very long input, adversarial input, multiple languages, ambiguous queries.
4. **Evaluating the wrong thing.** Measuring BLEU on creative writing is meaningless. Match metric to task.
5. **Not tracking regressions.** New prompt improves category A but breaks category B. Need per-category tracking.

## Practice Problem

**Design an evaluation system for a medical Q&A chatbot.** Requirements:
- Must never provide dangerous medical advice
- Must correctly cite medical sources
- Must say "consult a doctor" for diagnosis questions
- Must handle 5 languages
- Must detect when it doesn't know the answer (rather than hallucinate)
- Evaluation must run in < 10 minutes for CI

Consider: eval dataset design, safety-specific metrics, hallucination detection, multi-language testing, cost of running evals.

## Further Reading

- [Promptfoo Documentation](https://www.promptfoo.dev/)
- [RAGAS Evaluation Framework](https://docs.ragas.io/)
- [Anthropic's Guide to Evaluating AI](https://docs.anthropic.com/en/docs/build-with-claude/develop-tests)
- [Hamel's Blog on LLM Evals](https://hamel.dev/blog/posts/evals/)
