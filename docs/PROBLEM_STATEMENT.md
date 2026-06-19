# Problem Statement

> **The invisible architectural risks that slip through code review.**

---

## Executive Summary

Modern code review practices excel at catching style violations, logical errors, and missing tests. However, they are fundamentally blind to **architectural risks**—structural changes that introduce coupling, violate layering, or create technical debt.

This document outlines the two critical gaps in today's development workflow that Graphify Pre-Merge Architectural Review addresses.

---

## The Two Fundamental Gaps

### Gap 1: Post-Merge Knowledge Graphs Only

| Aspect | Current Reality |
|--------|-----------------|
| **When graphs are built** | After code merges to `main` |
| **What teams get** | A fresh map of the codebase |
| **The problem** | Architectural damage is already in the main branch |

Teams use Graphify today to build knowledge graphs **after** code lands. While useful for documentation and exploration, this is inherently **reactive**:

- New coupling patterns are already established
- God nodes have already grown
- Cross-community tangles are already merged
- Technical debt has already accumulated

**The damage is done.** The graph shows you what went wrong, but not until it's too late to prevent it.

---

### Gap 2: No Architectural Review in Pull Requests

| Aspect | Current Reality |
|--------|-----------------|
| **What code review focuses on** | Style, logic, tests |
| **What code review misses** | Structural risks |
| **When problems are discovered** | During merge conflicts, bugs, or refactors |

Code review processes are designed to answer:
- Does this code follow our style guide?
- Does the logic work correctly?
- Are there sufficient tests?

They are **not designed** to answer:
- Does this change accidentally couple the UI and database layers?
- Is this PR touching a god node that 47 other files depend on?
- Are changes spread across too many modules (low cohesion)?
- Will this change break if another PR merges first?

These structural risks go unnoticed until they manifest as:
- Merge conflicts requiring expensive rework
- Bugs caused by unexpected dependencies
- Costly refactoring projects to untangle coupled code
- Degraded developer velocity over time

---

## The Cost of These Gaps

### Immediate Costs

| Cost Type | Description |
|-----------|-------------|
| **Merge conflicts** | PRs that touch overlapping files require manual resolution |
| **Bug introduction** | Unexpected dependencies cause runtime failures |
| **Review fatigue** | Reviewers can't assess architectural impact, leading to rubber-stamping |

### Long-Term Costs

| Cost Type | Description |
|-----------|-------------|
| **Technical debt accumulation** | Small architectural violations compound over time |
| **God node growth** | Central files become increasingly difficult to modify safely |
| **Layering violations** | Clean architecture degrades into a tangled mess |
| **Developer velocity decline** | Changes take longer as the codebase becomes harder to understand |

---

## Why Traditional Approaches Fail

### Static Analysis Tools

Static analysis tools (linters, type checkers) catch syntax and type errors but don't understand **architectural boundaries**:
- They don't know that `ui/` shouldn't import from `db/`
- They can't identify when a file has become a god node
- They don't track community cohesion over time

### Architecture Decision Records (ADRs)

ADRs document decisions but don't enforce them:
- They're passive documents, not active checks
- They require manual maintenance
- They don't catch violations in individual PRs

### Code Review Checklists

Checklists remind reviewers to consider architecture but:
- They rely on human intuition
- They're inconsistent across reviewers
- They can't quantify blast radius or cohesion changes

---

## The Missing Piece: Pre-Merge Graph Intelligence

What's needed is a system that can:

1. **Analyze structural changes before merge** — not after
2. **Quantify architectural impact** — not just flag violations
3. **Provide actionable recommendations** — not just warnings
4. **Integrate seamlessly into existing workflows** — not require new processes

**Graphify Pre-Merge Architectural Review fills this gap** by leveraging Graphify's existing graph construction, community detection, centrality metrics, and reverse traversal capabilities—applied to two graphs instead of one.

---

## Summary

| Problem | Impact | Solution |
|---------|--------|----------|
| Post-merge graphs only | Architectural damage already merged | Pre-merge graph differencing |
| No architectural review in PRs | Structural risks go unnoticed | Data-driven structural impact reports |

The next document, [SOLUTION.md](./SOLUTION.md), details how we solve these problems.