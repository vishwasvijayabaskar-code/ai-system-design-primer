# Practice Problem: Design a Code Review Agent

> Difficulty: ⭐⭐⭐ | Key Concepts: Multi-step agents, tool use, context management

## Problem Statement

Design an AI agent that automatically reviews pull requests on GitHub. It should catch bugs, suggest improvements, and enforce coding standards as a first-pass reviewer before human review.

## Requirements

### Functional
- Triggered on every PR via GitHub webhook
- Reviews code diff, posts inline comments on specific lines
- Catches: bugs, security issues, performance problems, style violations
- Provides overall summary (approve / request changes / comment)
- Learns team conventions from past approved PRs
- Supports Python, TypeScript, Go, Rust

### Non-Functional
- Complete review in < 2 minutes
- False positive rate < 15%
- Cost < $0.50 per PR review
- Handle PRs up to 5,000 lines changed

## Constraints
- Cannot access full repo (cost) — only the diff + referenced files
- Monorepo with 100K+ files
- Proprietary code — no fine-tuning

## Hints

1. A 5,000-line diff is ~20K tokens. How do you fit it?
2. File-by-file vs holistic review — trade-offs?
3. How do you reduce false positives without missing real bugs?
4. How does the agent learn team conventions?

## Solution Walkthrough

### Architecture

```
GitHub Webhook ──→ Parse PR ──→ Split by file
                                    │
                       ┌────────────┤
                       ▼            ▼
                 ┌──────────┐  ┌──────────┐
                 │ File-by- │  │ Cross-   │
                 │ file     │  │ file     │
                 │ (Haiku   │  │ (Sonnet  │
                 │ parallel)│  │ holistic)│
                 └────┬─────┘  └────┬─────┘
                      │             │
                 ┌────▼─────────────▼────┐
                 │  Merge & Deduplicate  │
                 │  Filter confidence<0.7│
                 └──────────┬────────────┘
                            │
                 ┌──────────▼────────────┐
                 │  Post inline comments │
                 │  + PR summary         │
                 └───────────────────────┘
```

### Key Decisions

**Two-Pass Review:** Haiku per-file (parallel, cheap) catches syntax/style. Sonnet cross-file catches architectural issues. Merge + deduplicate.

**Context Management:**
- Small PRs (< 500 lines): full diff in one call
- Medium (500-2000): file-by-file parallel
- Large (2000-5000): file-by-file + summaries → cross-file analysis

**Noise Reduction:** Every finding gets category + confidence + severity. Only post: critical > 0.5, warning > 0.7, info > 0.9.

**Convention Learning:** RAG over past 6 months of approved review comments. Include team linter config in system prompt.

**Cost:** Average PR (800 lines, 12 files): ~$0.015. Haiku parallel pass ~$0.005, Sonnet holistic ~$0.01.
