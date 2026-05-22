# Chapter 2: Prompt Engineering at Scale

## Overview

Prompt engineering isn't just writing good prompts — it's building **systems** around prompts. At scale, you need versioning, testing, evaluation, and deployment pipelines for prompts, just like you do for code.

## Key Concepts

### System Prompts vs User Prompts

```
┌────────────────────────────────────────────┐
│  System Prompt (you control)               │
│  "You are a customer support agent for     │
│   Acme Corp. Follow these rules..."        │
├────────────────────────────────────────────┤
│  Few-shot Examples (you control)           │
│  "User: How do I reset? → Agent: Go to..." │
├────────────────────────────────────────────┤
│  Retrieved Context (dynamic)               │
│  [RAG results, user history, etc.]         │
├────────────────────────────────────────────┤
│  User Message (user controls)              │
│  "My account is locked"                    │
└────────────────────────────────────────────┘
```

**Architecture insight:** Your system prompt is your most important config file. Treat it like code — version it, review it, test it.

### Prompt Patterns

#### 1. Zero-Shot
Send the task directly. No examples.

```
Classify this email as spam or not spam: "{email}"
```

**When to use:** Simple tasks, large models (GPT-4o, Claude Sonnet). Fast and cheap.

#### 2. Few-Shot
Include examples in the prompt.

```
Classify these emails:
Email: "You won a prize!" → spam
Email: "Meeting at 3pm" → not_spam
Email: "Buy cheap watches" → spam

Email: "{new_email}" →
```

**When to use:** When zero-shot accuracy is insufficient. Typically 3-5 examples. More examples = more tokens = more cost.

#### 3. Chain-of-Thought (CoT)

Force the model to reason step-by-step before answering.

```
Question: If a store has 3 boxes with 12 items each, and returns 5, how many remain?

Think step by step:
1. 3 boxes × 12 items = 36 total items
2. 36 - 5 returned = 31 items remain

Answer: 31
```

**System design impact:** CoT uses 2-5x more output tokens. Only use for tasks that need reasoning (math, logic, multi-step decisions). Don't use for classification or extraction.

#### 4. Structured Output

Force the model to respond in a specific format.

```
Extract the following fields from this invoice. Respond in JSON only:
{
  "vendor": "",
  "amount": 0,
  "date": "",
  "line_items": []
}
```

**Better approach:** Use provider-specific structured output features:
- OpenAI: `response_format: { type: "json_schema", json_schema: {...} }`
- Anthropic: Tool use with input schema
- Both guarantee valid JSON — no parsing failures.

### Prompt Versioning

In production, prompts change constantly. You need a system.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Prompt v1   │────→│  Prompt v2   │────→│  Prompt v3   │
│  baseline    │     │  add CoT     │     │  fix edge    │
│  acc: 78%    │     │  acc: 85%    │     │  acc: 89%    │
└──────────────┘     └──────────────┘     └──────────────┘
```

**Options for prompt storage:**

| Approach | Pros | Cons |
|----------|------|------|
| In code (string literals) | Version controlled, reviewed in PRs | Deploy to change a prompt |
| Config files (YAML/JSON) | Easy to edit, can hot-reload | Still requires deployment |
| Database | Change without deploy, A/B test | Harder to review, no git history |
| Prompt management tool (Langfuse, PromptLayer) | Best of all worlds | Another dependency |

**Recommendation:** Start with prompts in code. Move to a management tool when you need A/B testing or non-engineers editing prompts.

### A/B Testing Prompts

```
User Request
    │
    ├──50%──→ Prompt v2 (candidate) ──→ Log result + metrics
    │
    └──50%──→ Prompt v1 (control)   ──→ Log result + metrics
    │
    └──→ Compare: accuracy, latency, cost, user satisfaction
```

**Key metrics to track:**
- **Task accuracy** — does it get the right answer?
- **Latency** — time to first token, total time
- **Cost** — tokens used × price
- **User satisfaction** — thumbs up/down, escalation rate
- **Safety** — hallucination rate, refusal rate

## Architecture: Prompt Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                  Prompt Pipeline                         │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Template  │→│ Variable  │→│ Validator │              │
│  │ Store     │  │ Injection │  │          │              │
│  └──────────┘  └──────────┘  └──────────┘              │
│       │              │              │                    │
│       │         ┌────▼─────┐  ┌────▼─────┐             │
│       │         │ Context  │  │ Token    │             │
│       │         │ Window   │  │ Counter  │             │
│       │         │ Check    │  │          │             │
│       │         └────┬─────┘  └────┬─────┘             │
│       │              │              │                    │
│       │         ┌────▼──────────────▼────┐              │
│       │         │   Assembled Prompt     │              │
│       │         └────────────┬───────────┘              │
│       │                      │                           │
│  ┌────▼─────┐          ┌────▼─────┐                     │
│  │ Version  │          │   LLM    │                     │
│  │ Tracker  │          │   Call   │                     │
│  └──────────┘          └────┬─────┘                     │
│                             │                            │
│                        ┌────▼─────┐                     │
│                        │  Logger  │                     │
│                        │ (prompt, │                     │
│                        │ response,│                     │
│                        │ metrics) │                     │
│                        └──────────┘                     │
└─────────────────────────────────────────────────────────┘
```

## Implementation Patterns

### Pattern 1: Template Engine

```python
# Don't do this:
prompt = f"You are a {role}. Answer about {topic}. Context: {context}"

# Do this:
class PromptTemplate:
    def __init__(self, template, version):
        self.template = template
        self.version = version

    def render(self, **kwargs):
        prompt = self.template.format(**kwargs)
        if count_tokens(prompt) > MAX_CONTEXT:
            raise ContextOverflowError(f"Prompt is {count_tokens(prompt)} tokens")
        return prompt

# Usage
support_prompt = PromptTemplate(
    template="""You are a support agent for {company}.
Rules:
- Only answer from provided context
- If unsure, say "Let me escalate this"
- Never share internal pricing

Context:
{context}

Customer question: {question}""",
    version="2.3.1"
)
```

### Pattern 2: Prompt Chains

Break complex tasks into multiple LLM calls.

```
User Query: "Summarize this legal document and flag any risky clauses"

Step 1: Extract clauses (fast model, structured output)
    → [clause_1, clause_2, ..., clause_n]

Step 2: Classify risk per clause (fast model, parallel)
    → [{clause: ..., risk: "high"}, {clause: ..., risk: "low"}, ...]

Step 3: Summarize high-risk clauses (smart model)
    → Final summary with flagged risks
```

**Why chain?** Single-prompt approach would need a huge context window and expensive model. Chaining lets you use cheap models for easy steps and expensive models only where needed.

### Pattern 3: Guardrailed Prompts

```
System prompt:
  "You are a medical information assistant.
   NEVER provide diagnoses.
   NEVER recommend specific medications.
   ALWAYS suggest consulting a doctor for medical decisions.
   If the user asks for a diagnosis, respond with:
   'I can provide general health information, but please consult
    a healthcare professional for diagnosis and treatment.'"
```

**Defense in depth:** Don't rely only on the system prompt. Add output validation:

```python
def validate_response(response):
    banned_patterns = ["you should take", "your diagnosis is", "I recommend"]
    for pattern in banned_patterns:
        if pattern.lower() in response.lower():
            return SAFE_FALLBACK_RESPONSE
    return response
```

## Common Pitfalls

1. **Prompt injection.** Users can override your system prompt. "Ignore all previous instructions and..." — use input sanitization and output validation.
2. **Prompt bloat.** Adding rules for every edge case creates 5000-token system prompts. Each request costs more. Audit and trim regularly.
3. **Not testing prompts.** A small prompt change can break 20% of cases. Build evaluation sets and run them before deploying prompt changes.
4. **Hardcoding everything.** Prompts in code means deploying to fix a typo. Use config files or a prompt management tool.
5. **Ignoring token limits.** Your system prompt + few-shot examples + RAG context + user message must fit in the context window with room for the response. Count tokens before sending.

## Practice Problem

**Design a prompt management system for a customer support chatbot.** Requirements:
- 50 support agents use it daily
- Product team wants to A/B test prompt variations
- Must support different prompts per product category (billing, technical, returns)
- Need to roll back bad prompt versions in < 1 minute
- Track which prompt version generated each response

Consider: storage, versioning, deployment, rollback, metrics collection, access control.

## Further Reading

- [Anthropic's Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering)
- [OpenAI Prompt Engineering Best Practices](https://platform.openai.com/docs/guides/prompt-engineering)
- [Langfuse Prompt Management](https://langfuse.com/docs/prompts)
- [Braintrust Prompt Playground](https://www.braintrust.dev/)
