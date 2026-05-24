# Practice Problem: Design a Multi-Agent Coding System

> Difficulty: ⭐⭐⭐⭐ | Key Concepts: Agent orchestration, planning, tool use, evaluation

## Problem Statement

Design a system where multiple AI agents collaborate to build features from a product spec. One agent plans, one writes code, one writes tests, and one reviews — similar to how a real engineering team works.

## Requirements

### Functional
- Input: Product spec or GitHub issue describing a feature
- Output: Working code with tests, submitted as a PR
- Agents: Planner, Coder, Tester, Reviewer
- Must understand existing codebase (not just write new files)
- Handle repos up to 100K files
- Support Python and TypeScript

### Non-Functional
- Complete a typical feature (200-500 lines) in < 10 minutes
- Code passes existing test suite 95% of the time
- Cost < $5 per feature implementation
- Human can intervene at any stage (approve plan, reject code)

## Constraints
- Cannot load entire repo into context (too large)
- Must use existing project conventions (linter, test framework, patterns)
- Cannot deploy or run production code (sandbox only)
- Must handle merge conflicts if base branch changes

## Hints

1. How does the Planner decide which files to modify without seeing the whole repo?
2. How do agents share context? Full conversation history gets huge.
3. What happens when the Reviewer rejects code? How many revision loops?
4. How do you test the system itself? (Meta-evaluation problem)

## Solution Walkthrough

### Architecture

```
Spec ──→ Planner ──→ Plan
              │
              ▼
         ┌─────────────────────────────────────┐
         │          Execution Loop              │
         │                                     │
         │  Coder ──→ Code ──→ Tester ──→ Tests│
         │    │                    │            │
         │    │              Tests pass?        │
         │    │              │        │         │
         │    │             Yes      No         │
         │    │              │     Fix + retry  │
         │    │              ▼                  │
         │    │         Reviewer                │
         │    │           │    │                │
         │    │        Approve Reject           │
         │    │           │     │               │
         │    │          PR   Back to Coder     │
         │    │                (max 3 loops)    │
         └────┴────────────────────────────────┘
```

### Key Decisions

**Planner Agent (Sonnet):**
- Tool: `search_codebase(query)` — semantic search over file summaries
- Tool: `read_file(path)` — read specific files
- Tool: `list_directory(path)` — explore repo structure
- Creates: step-by-step plan, list of files to modify, acceptance criteria
- Human checkpoint: user approves plan before coding starts

**Coder Agent (Sonnet):**
- Gets: plan + relevant files (loaded by Planner)
- Tool: `write_file(path, content)` — create/modify files
- Tool: `read_file(path)` — read related files for context
- Uses: project's linter config, existing patterns from similar files
- Context management: only loads files referenced in plan (not whole repo)

**Tester Agent (Haiku):**
- Gets: changed files + existing test patterns
- Tool: `run_tests()` — execute test suite in sandbox
- Tool: `write_file(path, content)` — write test files
- Writes tests matching project's existing test framework
- Runs tests, reports pass/fail with output

**Reviewer Agent (Sonnet):**
- Gets: diff of all changes + plan + test results
- Checks: matches plan? follows conventions? handles edge cases? tests meaningful?
- Output: approve (create PR) or reject with specific feedback
- Max 3 revision loops, then escalate to human

**Codebase Understanding:**
```
Pre-index (one-time):
  1. Generate file summaries (Haiku, batch): path + 2-line description
  2. Embed summaries into vector DB
  3. Extract: imports graph, function signatures, test patterns

At plan time:
  1. Semantic search over file summaries: "Which files handle authentication?"
  2. Load relevant files (usually 5-15 files, not 100K)
  3. Include import graph to understand dependencies
```

**Cost Breakdown:**
```
Planner (Sonnet):  ~5 tool calls, 10K tokens  = $0.50
Coder (Sonnet):    ~10 tool calls, 20K tokens = $1.00
Tester (Haiku):    ~5 tool calls, 8K tokens   = $0.05
Reviewer (Sonnet): ~3 tool calls, 8K tokens   = $0.40
Revision loop (1): Coder + Tester again       = $1.05

Total (1 revision): ~$3.00 per feature ✓
```

**Evaluation:**
- Automated: Does code compile? Do tests pass? Does linter pass?
- SWE-bench style: Run against known issues with known solutions, compare
- Human: Engineering team reviews 20% of generated PRs, rates quality 1-5
- Track: acceptance rate, revision count, time to merge
