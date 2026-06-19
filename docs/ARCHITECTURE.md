# Architecture

> **Technical implementation details for the pre-merge architectural review system.**

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Graphify Pre-Merge Review System                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │   Git Host   │───▶│   Webhook    │───▶│    Agent     │              │
│  │  (GitHub/    │    │   Handler    │    │  Orchestrator│              │
│  │   GitLab)    │    │              │    │              │              │
│  └──────────────┘    └──────────────┘    └──────┬───────┘              │
│                                                   │                      │
│                                                   ▼                      │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                      Graphify Core                                │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │  │
│  │  │   Graph    │  │ Community  │  │ Centrality │  │   Graph    │ │  │
│  │  │ Builder    │  │ Detection  │  │  Metrics   │  │   Differ   │ │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘ │  │
│  │                                                                   │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │  │
│  │  │  Reverse   │  │  Cohesion  │  │   Cycle    │  │   Impact   │ │  │
│  │  │ Traversal  │  │  Calculator│  │  Detector  │  │  Analyzer  │ │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                   │                      │
│                                                   ▼                      │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                      Report Generator                             │  │
│  │  • Risk Score Calculation                                         │  │
│  │  • Finding Aggregation                                            │  │
│  │  • Recommendation Engine                                          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                   │                      │
│                                                   ▼                      │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                      PR Comment Poster                             │  │
│  │  • GitHub API / GitLab API                                        │  │
│  │  • Structured Markdown Output                                     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Graph Builder

**Purpose:** Construct knowledge graphs from source code.

**Inputs:**
- Repository URL or local path
- Branch name (e.g., `main`, `feature-branch`)

**Outputs:**
- Graph JSON with nodes, edges, metadata

**Process:**
```
Source Code → AST Parsing → Node Extraction → Edge Inference → Graph JSON
```

**Node Types:**
| Type | Description |
|------|-------------|
| `File` | Source files |
| `Function` | Functions/methods |
| `Class` | Classes |
| `Module` | Python modules, JS modules |
| `Document` | Markdown, docs |

**Edge Types:**
| Type | Description |
|------|-------------|
| `calls` | Function A calls function B |
| `imports` | File A imports file B |
| `contains` | Module contains file |
| `inherits` | Class A inherits from class B |
| `implements` | Class implements interface |

---

### 2. Community Detection

**Purpose:** Identify architectural modules (clusters of related nodes).

**Algorithm:** Leiden algorithm for community detection.

**Process:**
```
Graph → Leiden Algorithm → Communities (clusters of nodes)
```

**Output:**
```json
{
  "communities": [
    {
      "id": 0,
      "name": "UI",
      "nodes": ["button.py", "form.py", "modal.py"],
      "cohesion": 0.72
    },
    {
      "id": 1,
      "name": "Database",
      "nodes": ["query.py", "connection.py", "models.py"],
      "cohesion": 0.85
    }
  ]
}
```

---

### 3. Centrality Metrics

**Purpose:** Identify god nodes (highly connected, high-risk files).

**Metrics:**
| Metric | Formula | Purpose |
|--------|---------|---------|
| **Betweenness Centrality** | Fraction of shortest paths passing through node | Identify bottlenecks |
| **Degree Centrality** | Number of direct connections | Identify hubs |
| **Closeness Centrality** | Average distance to all other nodes | Identify central nodes |

**God Node Threshold:**
```
God Node = Betweenness Centrality > 0.70
```

**Output:**
```json
{
  "centrality": {
    "auth.py": {
      "betweenness": 0.87,
      "degree": 47,
      "is_god_node": true
    },
    "utils.py": {
      "betweenness": 0.23,
      "degree": 12,
      "is_god_node": false
    }
  }
}
```

---

### 4. Graph Differencer

**Purpose:** Compare two graphs and identify changes.

**Process:**
```
G_base (main) ──┐
                ├──▶ Diff Engine ──▶ G_diff
G_head (PR)   ──┘
```

**Diff Output:**
```json
{
  "nodes_added": ["new_file.py", "new_function()"],
  "nodes_removed": ["old_file.py"],
  "nodes_modified": ["database.py", "auth.py"],
  "edges_added": [
    {"from": "ui/button.py", "to": "db/query.py", "type": "calls"}
  ],
  "edges_removed": [],
  "communities_affected": [0, 2, 5]
}
```

---

### 5. Reverse Traversal (Blast Radius)

**Purpose:** Calculate how many nodes would be affected by a change.

**Algorithm:** Breadth-first search (BFS) in reverse direction.

**Process:**
```
Changed Node → Reverse BFS → All Dependent Nodes
```

**Output:**
```json
{
  "affected_nodes": [
    {"node": "database.py", "dependents": 23},
    {"node": "auth.py", "dependents": 47}
  ],
  "total_blast_radius": 70
}
```

---

### 6. Cycle Detector

**Purpose:** Identify circular dependencies.

**Algorithm:** Depth-first search (DFS) with cycle detection.

**Process:**
```
Graph → DFS → Cycles Found
```

**Output:**
```json
{
  "cycles": [
    {
      "path": ["A", "B", "C", "A"],
      "type": "import",
      "severity": "high"
    }
  ],
  "new_cycles": [
    {
      "path": ["ui.py", "db.py", "ui.py"],
      "introduced_by": "this PR"
    }
  ]
}
```

---

### 7. Cohesion Calculator

**Purpose:** Measure how tightly nodes within a community are connected.

**Formula:**
```
Cohesion = (Internal Edges) / (Possible Internal Edges)
```

**Interpretation:**
| Cohesion Score | Meaning |
|----------------|---------|
| 0.8 - 1.0 | Highly cohesive (good) |
| 0.5 - 0.8 | Moderately cohesive |
| 0.0 - 0.5 | Low cohesion (bad) |

**Output:**
```json
{
  "cohesion_delta": {
    "community_0": {"before": 0.72, "after": 0.65, "delta": -0.07},
    "community_1": {"before": 0.85, "after": 0.87, "delta": +0.02}
  }
}
```

---

### 8. Impact Analyzer

**Purpose:** Aggregate all metrics into a risk assessment.

**Risk Scoring:**
```
Risk Score = Σ (Metric Weight × Metric Value)

Example:
  Blast Radius Risk = 0.3 × (affected_nodes / threshold)
  God Node Risk = 0.3 × (god_nodes_modified / total_god_nodes)
  Sprawl Risk = 0.2 × (communities_touched / threshold)
  Cycle Risk = 0.2 × (new_cycles_count)
```

**Risk Levels:**
| Score | Level | Action |
|-------|-------|--------|
| 0.0 - 0.3 | Low | Safe to merge |
| 0.3 - 0.6 | Medium | Review recommended |
| 0.6 - 1.0 | High | Requires approval |

---

## Agent Orchestrator

### Trigger Mechanisms

| Trigger | Description |
|---------|-------------|
| **Webhook** | GitHub/GitLab webhook on PR creation/update |
| **Command** | User comment `/graphify review` on PR |
| **CI Integration** | As a step in CI pipeline |

### MCP Tools Available

| Tool | Purpose |
|------|---------|
| `graphify_affected` | Reverse dependency traversal |
| `graphify_centrality` | Get centrality metrics for node |
| `graphify_community_nodes` | Get nodes in a community |
| `graphify_diff_edges` | Compare edges between graphs |
| `graphify_detect_cycles` | Find circular dependencies |
| `graphify_cohesion` | Calculate community cohesion |
| `graphify_pr_impact` | Get full PR impact analysis |
| `graphify_list_prs` | List open PRs with impact data |
| `graphify_triage_prs` | Get actionable PRs for review |

### Agent Workflow

```python
async def analyze_pr(pr_number: int, base: str, head: str):
    # 1. Build graphs
    base_graph = await graphify_build(base)
    head_graph = await graphify_build(head)
    
    # 2. Compute diff
    diff = await graphify_diff(base_graph, head_graph)
    
    # 3. Analyze impact
    blast_radius = await graphify_affected(diff.nodes_modified)
    god_nodes = await graphify_centrality(diff.nodes_modified)
    communities = await graphify_community_nodes(diff.nodes_modified)
    cycles = await graphify_detect_cycles(head_graph)
    cohesion = await graphify_cohesion(base_graph, head_graph)
    
    # 4. Calculate risk
    risk_score = calculate_risk(
        blast_radius, god_nodes, communities, cycles, cohesion
    )
    
    # 5. Generate report
    report = generate_report(risk_score, findings)
    
    # 6. Post comment
    await post_pr_comment(pr_number, report)
```

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         Data Flow                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PR Created                                                     │
│      │                                                          │
│      ▼                                                          │
│  Webhook Triggered ──▶ Agent Orchestrator                      │
│                              │                                  │
│                              ▼                                  │
│                      Build Base Graph (main)                    │
│                              │                                  │
│                              ▼                                  │
│                      Build Head Graph (PR branch)               │
│                              │                                  │
│                              ▼                                  │
│                      Compute Graph Diff                         │
│                              │                                  │
│              ┌───────────────┼───────────────┐                 │
│              │               │               │                 │
│              ▼               ▼               ▼                 │
│         Blast Radius   God Node        Community               │
│         Analysis       Detection        Sprawl                 │
│              │               │               │                 │
│              └───────────────┼───────────────┘                 │
│                              │                                  │
│                              ▼                                  │
│                      Aggregate Results                          │
│                              │                                  │
│                              ▼                                  │
│                      Calculate Risk Score                       │
│                              │                                  │
│                              ▼                                  │
│                      Generate Report                            │
│                              │                                  │
│                              ▼                                  │
│                      Post PR Comment                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Integration Points

### GitHub Integration

```yaml
# .github/workflows/graphify-review.yml
name: Graphify Architectural Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Graphify Review
        uses: graphify/pre-merge-review@v1
        with:
          base: ${{ github.base_ref }}
          head: ${{ github.head_ref }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

### GitLab Integration

```yaml
# .gitlab-ci.yml
graphify-review:
  stage: test
  script:
    - graphify pr --base $CI_MERGE_REQUEST_TARGET_BRANCH_NAME --head $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME --comment
  only:
    - merge_requests
```

---

## Summary

The architecture consists of:

1. **Graph Builder** — Constructs knowledge graphs from source code
2. **Community Detection** — Identifies architectural modules
3. **Centrality Metrics** — Finds god nodes
4. **Graph Differencer** — Compares two graphs
5. **Reverse Traversal** — Calculates blast radius
6. **Cycle Detector** — Finds circular dependencies
7. **Cohesion Calculator** — Measures module cohesion
8. **Impact Analyzer** — Aggregates metrics into risk score
9. **Agent Orchestrator** — Coordinates analysis and posts results

All components are deterministic and graph-based—no LLM guessing. The agent only interprets and presents the data.