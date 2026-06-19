# Metrics Reference

> **Complete guide to all architectural metrics computed by Graphify Pre-Merge Review.**

---

## Overview

All metrics in Graphify Pre-Merge Review are **deterministic** and **graph-based**. They are computed from the knowledge graph structure—not guessed by an LLM. This ensures consistency, reproducibility, and trustworthiness.

---

## Metric Categories

| Category | Metrics | Purpose |
|----------|---------|---------|
| **Impact** | Blast Radius, Affected Nodes | Understand change scope |
| **Risk** | God Node Hits, Centrality | Identify high-risk changes |
| **Architecture** | Community Sprawl, Cohesion Delta | Assess architectural health |
| **Quality** | Circular Dependencies, Cross-Community Coupling | Detect violations |

---

## 1. Blast Radius

### Definition

**Blast Radius** measures how many existing nodes (functions, files, modules) would be affected by a change. It's calculated through **reverse dependency traversal**.

### Formula

```
Blast Radius = Count of all nodes that transitively depend on changed nodes
```

### Calculation

```
Changed Node → Reverse BFS → All Dependent Nodes
```

1. Start from each changed node
2. Traverse edges in reverse direction (dependents → dependencies)
3. Count all reachable nodes

### Example

```
Changed: database.py

Dependents:
  ├── query_executor.py (calls database.py)
  │   └── api_handler.py (calls query_executor.py)
  ├── analytics.py (calls database.py)
  │   └── report_generator.py (calls analytics.py)
  └── cache.py (calls database.py)

Blast Radius: 6 nodes affected
```

### Interpretation

| Blast Radius | Risk Level | Recommendation |
|--------------|------------|----------------|
| 1-5 nodes | Low | Safe to merge |
| 6-15 nodes | Medium | Review recommended |
| 16+ nodes | High | Requires extra scrutiny |

### Use Case

**Question:** "How many other files will break if I change this one?"

**Answer:** The blast radius tells you exactly how many functions/files depend on the changed node. A large blast radius means:
- More tests needed
- Higher risk of regressions
- Consider breaking into smaller changes

---

## 2. God Node Hits

### Definition

**God Nodes** are files or functions with high centrality—they are central hubs that many other nodes depend on. Modifying a god node is inherently risky.

### Formula

```
God Node = Betweenness Centrality > Threshold (default: 0.70)
```

### Centrality Metrics

| Metric | Description | Interpretation |
|--------|-------------|----------------|
| **Betweenness Centrality** | Fraction of shortest paths passing through node | Bottleneck indicator |
| **Degree Centrality** | Number of direct connections | Hub indicator |
| **Closeness Centrality** | Average distance to all other nodes | Centrality indicator |

### Example

```
auth.py:
  Betweenness Centrality: 0.87
  Degree: 47
  Is God Node: YES
  
utils.py:
  Betweenness Centrality: 0.23
  Degree: 12
  Is God Node: NO
```

### Interpretation

| Centrality | Risk Level | Action |
|------------|------------|--------|
| < 0.50 | Low | Normal review |
| 0.50 - 0.70 | Medium | Extra attention |
| > 0.70 | High | God node - requires approval |

### Use Case

**Question:** "Is this PR touching a high-risk central file?"

**Answer:** Check if any changed node has centrality > 0.70. If yes:
- Flag for senior review
- Require additional tests
- Consider refactoring to reduce centrality

---

## 3. Community Sprawl

### Definition

**Community Sprawl** measures how many architectural modules (communities) a change touches. Changes should ideally be cohesive—contained within one or two communities.

### Formula

```
Sprawl = Count of unique communities among changed nodes
```

### Calculation

1. Detect communities using Leiden algorithm
2. Map each changed node to its community
3. Count unique communities

### Example

```
Changed files:
  ├── ui/button.py → Community 0 (UI)
  ├── ui/form.py → Community 0 (UI)
  ├── db/query.py → Community 1 (Database)
  ├── db/connection.py → Community 1 (Database)
  └── analytics/report.py → Community 2 (Analytics)

Community Sprawl: 3 communities
```

### Interpretation

| Communities Touched | Risk Level | Recommendation |
|---------------------|------------|----------------|
| 1 | Low | Cohesive change |
| 2 | Medium | Acceptable |
| 3+ | High | Consider splitting PR |

### Use Case

**Question:** "Is this PR touching too many modules at once?"

**Answer:** Count unique communities among changed nodes. High sprawl indicates:
- Low cohesion
- PR should be split
- Multiple reviewers needed

---

## 4. Circular Dependencies

### Definition

**Circular Dependencies** occur when A depends on B, B depends on C, and C depends on A. These create tight coupling and make code harder to understand and test.

### Detection Algorithm

```
Graph → DFS with back-edge detection → Cycles Found
```

### Types of Cycles

| Cycle Type | Description | Severity |
|------------|-------------|----------|
| **Import Cycle** | Circular imports between modules | High |
| **Call Cycle** | Circular function calls | Medium |
| **Inheritance Cycle** | Circular class inheritance | Critical |

### Example

```
New circular dependency introduced:
  
  ui/button.py → db/query.py → cache.py → ui/button.py
  
This is a CYCLE introduced by this PR.
```

### Interpretation

| New Cycles | Risk Level | Action |
|------------|------------|--------|
| 0 | Low | No action |
| 1 | Medium | Review carefully |
| 2+ | High | Block merge until resolved |

### Use Case

**Question:** "Does this PR introduce new circular dependencies?"

**Answer:** Compare cycles in head graph vs base graph. Any new cycles are flagged as high risk.

---

## 5. Cross-Community Coupling

### Definition

**Cross-Community Coupling** occurs when a change adds edges between nodes in different communities. This can indicate layering violations or unintended dependencies.

### Detection

```
For each new edge in G_diff:
  If edge.from.community != edge.to.community:
    Flag as cross-community coupling
```

### Example

```
New edge detected:
  ui/button.py (Community: UI) → db/query.py (Community: Database)
  
This is a LAYERING VIOLATION.
UI should not directly call Database.
```

### Interpretation

| Cross-Community Edges | Risk Level | Action |
|-----------------------|------------|--------|
| 0 | Low | No action |
| 1-2 | Medium | Review for necessity |
| 3+ | High | Likely architectural violation |

### Use Case

**Question:** "Does this PR accidentally couple two unrelated layers?"

**Answer:** Check for new edges between different communities. Flag any that violate architectural boundaries.

---

## 6. Cohesion Delta

### Definition

**Cohesion** measures how tightly nodes within a community are connected. High cohesion is good (nodes belong together); low cohesion is bad (nodes should be separated).

### Formula

```
Cohesion = (Internal Edges) / (Possible Internal Edges)

Where:
  Internal Edges = edges between nodes in the same community
  Possible Internal Edges = n * (n-1) / 2 for n nodes
```

### Cohesion Delta

```
Cohesion Delta = Cohesion_after - Cohesion_before
```

### Example

```
Community "UI" before PR:
  Nodes: 10
  Internal Edges: 36
  Possible Edges: 45
  Cohesion: 36/45 = 0.80

Community "UI" after PR:
  Nodes: 12
  Internal Edges: 40
  Possible Edges: 66
  Cohesion: 40/66 = 0.61

Cohesion Delta: 0.61 - 0.80 = -0.19 (DECREASED)
```

### Interpretation

| Delta | Meaning | Risk Level |
|-------|---------|------------|
| > 0 | Cohesion improved | Low (good) |
| -0.1 to 0 | Slight decrease | Medium |
| < -0.1 | Significant decrease | High |

### Use Case

**Question:** "Is this PR making the architecture cleaner or messier?"

**Answer:** Compare cohesion before and after. A decrease indicates the change is fragmenting the community.

---

## 7. Merge Order Risk

### Definition

**Merge Order Risk** assesses whether a PR is safe to merge now, or if another PR should merge first due to overlapping changes.

### Calculation

```
For each open PR:
  Calculate community overlap with current PR
  
If overlap > threshold:
  Flag merge order dependency
```

### Example

```
Current PR #143:
  Communities: UI, Database

Open PRs:
  PR #142: Communities: Database, Analytics
  PR #141: Communities: Authentication

Overlap:
  PR #143 ∩ PR #142 = Database (shared community)
  
Recommendation: Merge PR #142 before PR #143
```

### Interpretation

| Overlap | Risk Level | Action |
|---------|------------|--------|
| None | Low | Safe to merge |
| 1 community | Medium | Check for conflicts |
| 2+ communities | High | Coordinate merge order |

### Use Case

**Question:** "Is this change safe to merge first, or should another PR go first?"

**Answer:** Compare communities touched by current PR with all open PRs. Flag any overlaps.

---

## Risk Score Calculation

### Overall Risk Score

```
Risk Score = Σ (Weight × Normalized Metric)

Where:
  Blast Radius Risk = 0.25 × (blast_radius / threshold)
  God Node Risk = 0.25 × (god_nodes_modified / total_god_nodes)
  Sprawl Risk = 0.20 × (communities_touched / threshold)
  Cycle Risk = 0.15 × (new_cycles_count)
  Coupling Risk = 0.15 × (cross_community_edges)
```

### Risk Levels

| Score | Level | Color | Action |
|-------|-------|-------|--------|
| 0.0 - 0.3 | Low | 🟢 | Safe to merge |
| 0.3 - 0.6 | Medium | 🟡 | Review recommended |
| 0.6 - 1.0 | High | 🔴 | Requires approval |

---

## Summary Table

| Metric | What It Measures | Good Value | Bad Value |
|--------|------------------|------------|-----------|
| **Blast Radius** | Nodes affected by change | 1-5 | 16+ |
| **God Node Hits** | High-centrality files modified | 0 | 2+ |
| **Community Sprawl** | Modules touched | 1-2 | 3+ |
| **Circular Dependencies** | New cycles introduced | 0 | 1+ |
| **Cross-Community Coupling** | Layering violations | 0 | 1+ |
| **Cohesion Delta** | Architecture health | > 0 | < -0.1 |
| **Merge Order Risk** | PR conflicts | None | Overlap |

---

All metrics are deterministic, reproducible, and based on graph structure—not LLM inference. This ensures trust in the analysis results.