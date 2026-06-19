# Solution: Pre-Merge Graph Differencing

> **Turning Graphify from a passive documentation tool into an active architectural gatekeeper.**

---

## Executive Summary

The solution is elegantly simple: instead of building a knowledge graph only from the target branch, we build **two graphs** and compute their **difference**. This reveals exactly what architectural changes a pull request introduces—before it merges.

---

## The Core Insight

Graphify already has all the primitives needed for pre-merge analysis:

- **Graph construction** — Build graphs from codebases
- **Community detection** — Identify architectural modules (Leiden algorithm)
- **Centrality metrics** — Find god nodes (high betweenness centrality)
- **Reverse traversal** — Calculate blast radius (what depends on this node)
- **Graph diff** — Compare two graphs and identify changes

The enhancement is simply: **apply these capabilities to two graphs instead of one.**

---

## How It Works

### Step 1: Build Two Graphs

```
┌─────────────────────────────────────────────────────────────┐
│                     Graph Construction                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Base Graph (G_base)          Head Graph (G_head)          │
│   ┌─────────────────┐          ┌─────────────────┐         │
│   │   main branch   │          │   PR branch     │         │
│   │                 │          │                 │         │
│   │  • Nodes        │          │  • Nodes        │         │
│   │  • Edges        │          │  • Edges        │         │
│   │  • Communities   │          │  • Communities  │         │
│   │  • Centrality    │          │  • Centrality   │         │
│   └─────────────────┘          └─────────────────┘         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Graph | Source | Purpose |
|-------|--------|---------|
| **Base Graph** | Target branch (e.g., `main`) | Represents current architecture |
| **Head Graph** | PR branch | Represents proposed changes |

### Step 2: Compute the Difference

```
┌─────────────────────────────────────────────────────────────┐
│                        Graph Diff                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   G_diff = G_head - G_base                                   │
│                                                             │
│   • Nodes added:     [new files, functions, classes]       │
│   • Nodes removed:    [deleted files, functions, classes]  │
│   • Nodes modified:   [changed files, functions, classes]  │
│   • Edges added:      [new dependencies, calls, imports]   │
│   • Edges removed:    [removed dependencies]               │
│   • Communities changed: [affected modules]                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Step 3: Analyze Architectural Impact

From the diff, we compute:

| Analysis | Question Answered |
|----------|-------------------|
| **Blast Radius** | How many existing nodes would be affected by this change? |
| **God Node Hits** | Does this PR touch a high-centrality file? |
| **Community Sprawl** | Are changes spread across too many modules? |
| **Circular Dependencies** | Does this PR introduce new cycles? |
| **Cross-Community Coupling** | Are unrelated layers being coupled? |
| **Cohesion Delta** | Is architecture getting cleaner or messier? |

---

## What Graphify Computes Before Merge

### 1. Blast Radius (Affected Nodes)

```bash
graphify affected <changed-file>
```

**What it does:** Reverse dependency traversal from changed nodes.

**What it tells you:**
- How many functions/files would be impacted
- The "blast radius" of the change
- Whether the change is localized or far-reaching

**Example:**
```
Changing database.py would affect:
  • 23 functions in 7 files
  • Communities: DB (12), Analytics (8), API (3)
  
Risk: HIGH - large blast radius
```

### 2. God Node Detection

```bash
graphify centrality --node <file>
```

**What it does:** Calculate betweenness centrality for changed nodes.

**What it tells you:**
- Is this file a central hub?
- How many other files depend on it?
- Does this change need extra scrutiny?

**Example:**
```
auth.py centrality: 0.87 (threshold: 0.70)
  • 47 files depend on this node
  • This is a GOD NODE
  
Risk: HIGH - god node modified
```

### 3. Community Sprawl

```bash
graphify community --list --files <changed-files>
```

**What it does:** Map changed files to their communities.

**What it tells you:**
- Are changes cohesive (one community)?
- Or scattered (many communities)?
- Should the PR be split?

**Example:**
```
Changed files span 4 communities:
  • UI (3 files)
  • DB (2 files)
  • Analytics (2 files)
  • Auth (1 file)
  
Risk: MEDIUM - consider splitting PR
```

### 4. Circular Dependencies

```bash
graphify diagnose circular --graph G_head
```

**What it does:** Detect cycles in the head graph that don't exist in base.

**What it tells you:**
- Does this PR introduce new cycles?
- Are there new circular dependencies?

**Example:**
```
New circular dependency detected:
  A → B → C → A
  
Risk: HIGH - circular dependency introduced
```

### 5. Cross-Community Coupling

```bash
graphify diff-edges --base G_base --head G_head
```

**What it does:** Identify new edges between different communities.

**What it tells you:**
- Are unrelated layers being coupled?
- Is there a layering violation?

**Example:**
```
New cross-community edge:
  ui/button.py → db/query.py
  
This is a LAYERING VIOLATION
  UI should not directly call DB
  
Risk: HIGH - architectural violation
```

### 6. Cohesion Delta

```bash
graphify cohesion --base G_base --head G_head
```

**What it does:** Compare community cohesion before and after.

**What it tells you:**
- Is the architecture getting more cohesive?
- Or more fragmented?

**Example:**
```
Community "UI" cohesion:
  Before: 0.72
  After:  0.65
  
Cohesion DECREASED by 0.07
  
Risk: MEDIUM - architecture degrading
```

---

## The Agent Orchestration Layer

Running five different Graphify commands and interpreting raw numbers is tedious. An AI agent orchestrates the entire analysis and produces a human-readable PR comment.

### Agent Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent Orchestration                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. TRIGGER                                                  │
│     • Webhook on new PR, or                                  │
│     • User command: /graphify review                        │
│                                                             │
│  2. CALL GRAPHIFY MCP TOOLS                                  │
│     • graphify_affected(node)                                │
│     • graphify_centrality(node)                              │
│     • graphify_community_nodes(changed_files)                │
│     • graphify_diff_edges(base, head)                        │
│     • graphify_detect_cycles(graph)                          │
│                                                             │
│  3. AGGREGATE RESULTS                                        │
│     • Collect metrics                                        │
│     • Identify risks                                         │
│     • Calculate overall risk score                           │
│                                                             │
│  4. GENERATE REPORT                                          │
│     • Overall risk assessment (Low/Medium/High)              │
│     • Key findings                                           │
│     • Actionable recommendations                             │
│                                                             │
│  5. POST COMMENT                                             │
│     • Structured markdown comment on PR                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Example Agent-Generated PR Comment

```markdown
## 📊 Graphify Architectural Review

**Overall risk:** ⚠️ Medium-High

### Key Findings

- **Blast radius:** Changing `database.py` would affect **23 functions** in 7 files.
- **God node hit:** `auth.py` (centrality 0.87) is modified – high-risk change.
- **Community sprawl:** Changes touch **3 communities** (UI, DB, Analytics) – consider splitting the PR.
- **New coupling:** An edge from `ui/button.py` → `db/query.py` adds a layering violation.
- **Circular dependencies:** None introduced.

### Recommendations

- ✅ Add more tests for `database.py` due to large blast radius.
- ⚠️ Break the PR into separate UI and DB changes.
- 🚫 Do not merge before PR #142 – it modifies overlapping files in the `analytics` community.
```

---

## Why This Is a Great Solution

| Aspect | Value |
|--------|-------|
| **Leverages existing Graphify internals** | No need to rebuild graph algorithms—they already exist |
| **Solves a real, painful problem** | Architectural drift is invisible in normal CI; this makes it visible instantly |
| **Agent-native** | The agent orchestrates analysis and writes the comment—a perfect use case for LLM-augmented development |
| **Low friction, high value** | No changes to developer workflow—just an additional CI step or comment trigger |
| **Deterministic and trustworthy** | All metrics are computed from the graph, not guessed by an LLM; the agent only interprets and presents the data |

---

## Summary

By extending Graphify with **pre-merge graph differencing** and an **agent orchestration layer**, we transform a passive documentation tool into an **active architectural gatekeeper**.

Every pull request receives a data-driven risk report that complements human review—catching coupling, god node abuse, and cohesion erosion **before** they land in the main branch.

The next document, [ARCHITECTURE.md](./ARCHITECTURE.md), details the technical implementation.