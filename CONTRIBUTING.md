# Contributing to AI System Design Primer

Thanks for your interest in contributing! This guide covers how to contribute effectively.

## Ways to Contribute

### 1. Improve Existing Chapters
- Fix typos, broken links, or outdated information
- Add better diagrams or examples
- Add real-world case studies
- Improve practice problems

### 2. Add Practice Problems
Practice problems should follow this format:

```markdown
**Design a [system] for [use case].** Requirements:
- [Requirement 1]
- [Requirement 2]
- [Requirement 3]

Consider: [list of architectural decisions to think about]
```

### 3. Report Issues
- Outdated pricing or model information
- Technical inaccuracies
- Missing important topics

## Style Guide

### Writing Style
- **Be concise.** System design interviews have time limits. Content should be scannable.
- **Use tables** for comparisons (not long paragraphs)
- **Use ASCII diagrams** (renders on GitHub without images)
- **Include trade-offs** — never recommend X without explaining when NOT to use X
- **Add "Common Pitfalls"** — what goes wrong in real systems

### Chapter Template

```markdown
# Chapter Title

## Overview (2-3 sentences)
## Key Concepts
## Architecture Diagram (ASCII art)
## Implementation Patterns
## Common Pitfalls (numbered list, 4-6 items)
## Practice Problem
## Further Reading (3-5 links)
```

### Code Style
- Use pseudocode or Python for examples
- Keep examples short (< 30 lines)
- Comment the "why", not the "what"

### Diagrams
Use ASCII art for all diagrams. This renders everywhere without external image hosting.

```
Good:
┌──────────┐     ┌──────────┐
│  Client  │────→│  Server  │
└──────────┘     └──────────┘

Bad:
![diagram](https://some-external-url/diagram.png)
```

## Pull Request Process

1. Fork the repo
2. Create a feature branch (`git checkout -b improve-rag-chapter`)
3. Make your changes
4. Ensure all links work
5. Submit a PR with a clear description of what you changed and why

## Questions?

Open an issue or start a discussion. We're happy to help you get started.
