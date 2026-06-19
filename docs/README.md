# Graphify Pre-Merge Architectural Review

> **Giving AI a map, and giving teams the power to review what matters.**

---

## Overview

Graphify Pre-Merge Architectural Review is a system that transforms how teams approach code review by catching **architectural issues before they merge**—not after.

Traditional code review focuses on style, logic, and tests. Structural risks—like accidentally coupling two layers, touching a high-risk central file, or spreading changes across many modules—go unnoticed until they cause merge conflicts, bugs, or expensive refactors.

This project leverages Graphify's existing graph intelligence capabilities to provide **pre-merge architectural analysis** for every pull request.

---

## The Core Innovation

| Traditional Workflow | Graphify-Enhanced Workflow |
|---------------------|----------------------------|
| Build knowledge graph **after** merge | Build graphs **before** merge |
| Architectural damage already in `main` | Risks flagged **before** merge |
| Reviewers rely on intuition | Reviewers get data-driven structural impact summary |
| God nodes grow unchecked | Every god node change highlighted for extra review |
| Cross-community coupling accumulates | Each new coupling edge visible in the diff |

---

## How It Works

```
┌─────────────────┐     ┌─────────────────┐
│  Base Graph     │     │  Head Graph     │
│  (main branch)  │     │  (PR branch)    │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     │
              ┌──────▼──────┐
              │  Graph Diff │
              └──────┬──────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
    ┌────▼────┐ ┌────▼────┐ ┌────▼────┐
    │ Blast   │ │ God Node│ │Community│
    │ Radius  │ │  Hits   │ │ Sprawl  │
    └─────────┘ └─────────┘ └─────────┘
                     │
              ┌──────▼──────┐
              │    Agent    │
              │  Synthesis  │
              └──────┬──────┘
                     │
              ┌──────▼──────┐
              │  PR Comment │
              │   Report    │
              └─────────────┘
```

1. **Build two graphs**: One from the target branch (`main`), one from the PR branch
2. **Compute the difference**: Identify all structural changes
3. **Analyze impact**: Calculate blast radius, god node hits, community sprawl, etc.
4. **Agent synthesis**: An AI agent interprets metrics and generates human-readable feedback
5. **Post PR comment**: Structured report with risk assessment and recommendations

---

## Key Capabilities

| Capability | What It Tells You |
|------------|-------------------|
| **Blast Radius** | How many existing functions/files would be impacted by this change |
| **God Node Detection** | Is the PR touching a central, high-risk file |
| **Community Sprawl** | Are changes spread across too many architectural modules |
| **Circular Dependencies** | Does the PR introduce new cycles in the dependency graph |
| **Cross-Community Coupling** | Are unrelated layers being accidentally coupled |
| **Cohesion Delta** | Is the architecture getting cleaner or messier |

---

## Why This Matters

Architectural drift is invisible in normal CI pipelines. Teams only notice it after months of merging "clean" PRs. Pre-merge graph differencing makes it visible instantly.

**Benefits:**
- Catch coupling before it becomes technical debt
- Identify high-risk changes that need extra scrutiny
- Prevent layering violations before they merge
- Make informed decisions about PR merge order
- Keep architecture clean and maintainable

---

## Documentation Structure

| File | Description |
|------|-------------|
| [PROBLEM_STATEMENT.md](./PROBLEM_STATEMENT.md) | Detailed explanation of the problems we're solving |
| [SOLUTION.md](./SOLUTION.md) | The complete solution architecture |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Technical implementation details |
| [METRICS.md](./METRICS.md) | All computed metrics explained |
| [AGENT_INTEGRATION.md](./AGENT_INTEGRATION.md) | How the AI agent orchestrates analysis |
| [USAGE.md](./USAGE.md) | How to use the system |

---

## Quick Start

```bash
# Analyze a PR against main branch
graphify pr --base main --head feature-branch

# Generate a PR comment with risk assessment
graphify pr --base main --head feature-branch --comment

# Use agent-powered analysis
graphify pr --base main --head feature-branch --agent opencode
```

---

## Project Status

This project is in the design/planning phase. The documentation here represents the complete vision for the pre-merge architectural review system.

---

## License

MIT

---

## Contributing

Contributions are welcome! Please read the documentation to understand the architecture before submitting PRs.