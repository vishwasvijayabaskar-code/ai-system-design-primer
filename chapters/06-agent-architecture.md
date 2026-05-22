# Chapter 6: Agent Architecture

## Overview

An AI agent is an LLM that can take actions — call tools, make decisions, and loop until a task is complete. Agents are the difference between a chatbot (answers questions) and an AI system (does work). This chapter covers the architecture patterns for building reliable agents.

## Agent vs Chatbot

```
Chatbot:    User → LLM → Response (one shot)

Agent:      User → LLM → Think → Act → Observe → Think → Act → ... → Done
                    │       │       │
                    │    ┌──▼──┐  ┌─▼────────┐
                    │    │Tool │  │Environment│
                    │    │Call │  │ Result    │
                    │    └─────┘  └──────────┘
```

**Key difference:** Agents have a *loop*. They observe results, reason about next steps, and keep going until the task is complete or they get stuck.

## The ReAct Pattern

Most agents follow ReAct (Reasoning + Acting):

```
┌─────────────────────────────────────────┐
│              Agent Loop                  │
│                                         │
│  1. THINK: "I need to find the user's   │
│     order. Let me search the database." │
│                                         │
│  2. ACT: search_orders(user_id=123)     │
│                                         │
│  3. OBSERVE: [{order_id: 456, ...}]     │
│                                         │
│  4. THINK: "Found it. Now I need to     │
│     check the refund policy."           │
│                                         │
│  5. ACT: get_policy("refunds")          │
│                                         │
│  6. OBSERVE: "Refunds within 30 days..."│
│                                         │
│  7. THINK: "Order is 5 days old, within │
│     policy. I can process the refund."  │
│                                         │
│  8. ACT: respond_to_user("Your order    │
│     qualifies for a refund...")         │
│                                         │
│  9. DONE                                │
└─────────────────────────────────────────┘
```

## Tool Use

Tools are functions the agent can call. They're the agent's hands.

```python
# Define tools with clear descriptions
tools = [
    {
        "name": "search_orders",
        "description": "Search customer orders by user ID, order ID, or date range",
        "parameters": {
            "user_id": {"type": "string", "required": True},
            "status": {"type": "string", "enum": ["pending", "shipped", "delivered"]},
            "date_from": {"type": "string", "format": "date"}
        }
    },
    {
        "name": "issue_refund",
        "description": "Process a refund for an order. Requires manager approval for > $500",
        "parameters": {
            "order_id": {"type": "string", "required": True},
            "amount": {"type": "number", "required": True},
            "reason": {"type": "string", "required": True}
        }
    }
]
```

**Tool design principles:**
- **Clear names and descriptions** — the LLM reads these to decide which tool to use
- **Typed parameters** — prevents the LLM from passing wrong types
- **Error messages** — return helpful errors, not stack traces
- **Idempotent when possible** — agent might retry if it's unsure the action succeeded

## Agent Architectures

### 1. Simple Loop (Single Agent)

```
User Task → Agent Loop → Done
                │
           ┌────▼────┐
           │  Tools   │
           │ ┌──────┐ │
           │ │Search │ │
           │ │ API   │ │
           │ │ DB    │ │
           │ └──────┘ │
           └──────────┘
```

**Best for:** Simple tasks with 2-5 tool calls. Customer support, data lookup, simple automation.

### 2. Router + Specialist Agents

```
User Task → Router Agent → Classify task type
                │
         ┌──────┼──────────┐
         ▼      ▼          ▼
    ┌────────┐ ┌────────┐ ┌────────┐
    │Billing │ │Tech    │ │Account │
    │ Agent  │ │Support │ │ Agent  │
    │        │ │ Agent  │ │        │
    │Tools:  │ │Tools:  │ │Tools:  │
    │-refund │ │-logs   │ │-reset  │
    │-invoice│ │-restart│ │-update │
    └────────┘ └────────┘ └────────┘
```

**Best for:** Multiple domains with different tools. Each specialist has a focused prompt and relevant tools only.

### 3. Planner + Executor

```
User Task → Planner Agent → Step-by-step plan
                                │
                    ┌───────────▼───────────┐
                    │     Execution Plan     │
                    │  1. Search codebase    │
                    │  2. Identify bug       │
                    │  3. Write fix          │
                    │  4. Run tests          │
                    │  5. Create PR          │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────┐
                    │   Executor Agent       │
                    │   (executes each step) │
                    │   Re-plans if stuck    │
                    └───────────────────────┘
```

**Best for:** Complex multi-step tasks. Code agents, research agents, data analysis.

### 4. Multi-Agent Collaboration

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Researcher│────→│  Writer  │────→│ Reviewer │
│           │     │          │     │          │
│ Gathers   │     │ Drafts   │     │ Checks   │
│ info      │     │ content  │     │ quality  │
└──────────┘     └──────────┘     └─────┬────┘
                                        │
                                   Pass/Fail
                                   │       │
                                   ▼       ▼
                                 Done    Revise
                                        (back to Writer)
```

**Best for:** Tasks that benefit from different perspectives. Content creation, code review, research.

## State Management

Agents need state. The conversation history IS the state, but it grows unboundedly.

```
┌─────────────────────────────────────────┐
│           Agent State                    │
│                                         │
│  Short-term: Conversation history       │
│  (last N messages, summarized older)    │
│                                         │
│  Working memory: Current task context   │
│  (plan, partial results, scratchpad)    │
│                                         │
│  Long-term: Persistent storage          │
│  (user preferences, past interactions)  │
│                                         │
│  Tool state: External system state      │
│  (database records, file changes)       │
└─────────────────────────────────────────┘
```

**Context window management:**

| Strategy | How | Trade-off |
|----------|-----|-----------|
| Sliding window | Keep last N messages | Loses early context |
| Summarization | Summarize old messages, keep recent | Lossy, costs extra LLM call |
| RAG over history | Embed messages, retrieve relevant | Complex but best for long sessions |
| Truncation | Hard cut at token limit | Cheapest but loses context |

## Error Handling & Safety

### Max Iterations

```python
MAX_ITERATIONS = 10

for i in range(MAX_ITERATIONS):
    action = agent.think()

    if action.type == "done":
        return action.result

    result = execute_tool(action)
    agent.observe(result)

# If we get here, agent is stuck
return "I wasn't able to complete this task. Here's what I've done so far: ..."
```

**Always set a max iteration limit.** Without one, a confused agent loops forever, burning tokens and money.

### Human-in-the-Loop

```
Agent wants to: issue_refund(order=456, amount=$750)
    │
    ├── amount > $500? → Require human approval
    │                      │
    │                 ┌────▼────┐
    │                 │ Manager │
    │                 │ Approve?│
    │                 └────┬────┘
    │                      │
    │               Yes ───┼─── No
    │                │          │
    │           Execute     Reject + explain
    │
    └── amount <= $500? → Auto-approve
```

**Rules for human-in-the-loop:**
- Destructive actions (delete, send, pay) → always confirm
- High-value actions (> threshold) → require approval
- Ambiguous intent → ask for clarification
- First time seeing a pattern → confirm, then auto-approve similar

### Tool Permissions

```python
# Define permission levels per tool
TOOL_PERMISSIONS = {
    "search_orders": "read",       # Always allowed
    "update_order": "write",       # Needs confirmation
    "issue_refund": "dangerous",   # Needs manager approval
    "delete_account": "prohibited" # Never allow agent to do this
}
```

## Architecture: Production Agent

```
┌─────────────────────────────────────────────────────────┐
│                  Agent Runtime                           │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  Prompt   │  │  State   │  │ Tool     │              │
│  │ Manager   │  │ Manager  │  │ Registry │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │              │                    │
│  ┌────▼──────────────▼──────────────▼────┐              │
│  │              Agent Loop               │              │
│  │                                       │              │
│  │  Assemble Prompt → LLM Call →         │              │
│  │  Parse Action → Validate →            │              │
│  │  Execute Tool → Log → Repeat          │              │
│  └───────────────┬───────────────────────┘              │
│                  │                                       │
│  ┌───────────────▼───────────────────────┐              │
│  │          Safety Layer                 │              │
│  │  ┌────────┐ ┌────────┐ ┌────────┐    │              │
│  │  │Max iter│ │Approval│ │Rollback│    │              │
│  │  │ limit  │ │ gates  │ │        │    │              │
│  │  └────────┘ └────────┘ └────────┘    │              │
│  └───────────────────────────────────────┘              │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Tracing  │  │  Cost    │  │  Eval    │              │
│  │ (spans)  │  │ Tracking │  │ Metrics  │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

## Common Pitfalls

1. **No iteration limit.** Agent gets confused, loops 50 times, spends $20 on one request. Always cap iterations.
2. **Too many tools.** Giving an agent 50 tools confuses it. 5-10 well-defined tools per agent is optimal. Use router + specialists for more.
3. **No observability.** When an agent fails, you need to see the full trace: every thought, every tool call, every result. Use Langfuse, LangSmith, or similar.
4. **Unsafe tool access.** Agent with database write access and no guardrails = production data corruption. Gate destructive actions.
5. **Giant prompts.** Stuffing tool descriptions + conversation history + system prompt into one context window. Use summarization and RAG to manage state.
6. **No fallback for failures.** If a tool call fails, the agent should retry, try an alternative, or gracefully explain — not crash.

## Practice Problem

**Design a coding agent that can resolve GitHub issues.** Requirements:
- Reads issue description and comments
- Searches relevant code in the repo
- Proposes a fix (code changes)
- Runs existing tests to verify
- Creates a PR with the fix
- Handles repos up to 100K files

Consider: tool design, context management (code doesn't fit in one context window), planning strategy, safety (don't break production), evaluation (did it actually fix the issue?).

## Further Reading

- [Anthropic's Agent Patterns](https://docs.anthropic.com/en/docs/build-with-claude/agentic)
- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)
- [LangGraph Agent Tutorial](https://langchain-ai.github.io/langgraph/)
- [ReAct Paper](https://arxiv.org/abs/2210.03629)
