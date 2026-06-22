# Agent Integration

> **How the AI agent orchestrates Graphify's primitives to produce human-readable PR reviews.**

---

## Overview

The agent is the orchestration layer that transforms raw graph metrics into actionable, human-readable feedback. It calls Graphify's MCP tools, aggregates results, calculates risk scores, and posts structured comments on pull requests.

---

## Why an Agent?

Running five different Graphify commands and interpreting raw numbers is tedious for humans. An AI agent excels at:

| Task | Human | Agent |
|------|-------|-------|
| **Orchestrating multiple commands** | Manual, error-prone | Automatic, consistent |
| **Synthesizing findings** | Time-consuming | Instant |
| **Generating natural language** | Variable quality | Consistent format |
| **Posting PR comments** | Manual copy-paste | Automatic |

The agent handles the mechanical work, leaving humans to focus on the actual architectural decisions.

---

## Agent Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Agent Orchestrator                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Trigger   │───▶│   Graph     │───▶│   Impact    │         │
│  │  Handler    │    │  Builder    │    │  Analyzer   │         │
│  └─────────────┘    └─────────────┘    └──────┬──────┘         │
│                                                │                 │
│                                                ▼                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    MCP Tool Calls                        │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │ graphify_    │  │ graphify_    │  │ graphify_    │   │   │
│  │  │ affected     │  │ centrality   │  │ community    │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │ graphify_    │  │ graphify_    │  │ graphify_    │   │   │
│  │  │ diff_edges   │  │ cycles       │  │ cohesion     │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                │                 │
│                                                ▼                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Result    │───▶│   Risk      │───▶│   Report    │         │
│  │  Aggregator │    │  Calculator │    │  Generator  │         │
│  └─────────────┘    └─────────────┘    └──────┬──────┘         │
│                                                │                 │
│                                                ▼                 │
│                                         ┌─────────────┐         │
│                                         │ PR Comment  │         │
│                                         │   Poster    │         │
│                                         └─────────────┘         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Trigger Mechanisms

### 1. Webhook Trigger

```yaml
# GitHub Webhook
on:
  pull_request:
    types: [opened, synchronize]

# GitLab Webhook
on:
  merge_request:
    types: [opened, updated]
```

**Flow:**
1. PR is created/updated
2. Webhook fires
3. Agent receives event
4. Agent starts analysis

### 2. Command Trigger

```markdown
<!-- User comments on PR -->
/graphify review
```

**Flow:**
1. User comments `/graphify review` on PR
2. Agent receives command
3. Agent starts analysis

### 3. CI Integration

```yaml
# As a CI step
jobs:
  graphify-review:
    runs-on: ubuntu-latest
    steps:
      - name: Run Graphify Review
        run: graphify pr --base main --head $BRANCH --comment
```

---

## MCP Tools Available

The agent has access to these Graphify MCP tools:

### Core Analysis Tools

| Tool | Purpose | Returns |
|------|---------|---------|
| `graphify_affected` | Reverse dependency traversal | List of dependent nodes |
| `graphify_centrality` | Centrality metrics for node | Betweenness, degree, closeness |
| `graphify_community_nodes` | Nodes in a community | List of nodes |
| `graphify_diff_edges` | Compare edges between graphs | Added/removed edges |
| `graphify_detect_cycles` | Find circular dependencies | List of cycles |
| `graphify_cohesion` | Calculate community cohesion | Cohesion score |

### PR-Specific Tools

| Tool | Purpose | Returns |
|------|---------|---------|
| `graphify_pr_impact` | Full PR impact analysis | Complete impact report |
| `graphify_list_prs` | List open PRs with impact data | PR list with metadata |
| `graphify_triage_prs` | Get actionable PRs for review | Prioritized PR list |

### Graph Navigation Tools

| Tool | Purpose | Returns |
|------|---------|---------|
| `graphify_get_node` | Get node details | Node metadata |
| `graphify_get_neighbors` | Get node neighbors | Connected nodes |
| `graphify_get_community` | Get community by ID | Community nodes |
| `graphify_god_nodes` | Get most connected nodes | Top N god nodes |
| `graphify_graph_stats` | Get graph statistics | Node/edge counts |
| `graphify_query_graph` | Search the graph | Relevant nodes |
| `graphify_shortest_path` | Find path between nodes | Path sequence |

---

## Agent Workflow

### Step 1: Receive Trigger

```python
async def handle_pr_event(event: PREvent):
    pr_number = event.pr_number
    base_branch = event.base_ref
    head_branch = event.head_ref
    
    await analyze_pr(pr_number, base_branch, head_branch)
```

### Step 2: Build Graphs

```python
async def build_graphs(base: str, head: str):
    # Build base graph from target branch
    base_graph = await graphify_build(branch=base)
    
    # Build head graph from PR branch
    head_graph = await graphify_build(branch=head)
    
    return base_graph, head_graph
```

### Step 3: Compute Diff

```python
async def compute_diff(base_graph, head_graph):
    # Get all changes
    diff = await graphify_diff(base_graph, head_graph)
    
    return {
        'nodes_added': diff.nodes_added,
        'nodes_removed': diff.nodes_removed,
        'nodes_modified': diff.nodes_modified,
        'edges_added': diff.edges_added,
        'edges_removed': diff.edges_removed,
    }
```

### Step 4: Analyze Impact

```python
async def analyze_impact(diff, head_graph):
    # 1. Blast radius
    blast_radius = await graphify_affected(diff.nodes_modified)
    
    # 2. God node hits
    god_nodes = []
    for node in diff.nodes_modified:
        centrality = await graphify_centrality(node)
        if centrality.betweenness > 0.70:
            god_nodes.append(node)
    
    # 3. Community sprawl
    communities = await graphify_community_nodes(diff.nodes_modified)
    sprawl = len(set(communities))
    
    # 4. Circular dependencies
    cycles = await graphify_detect_cycles(head_graph)
    new_cycles = [c for c in cycles if c.is_new]
    
    # 5. Cross-community coupling
    cross_edges = await graphify_diff_edges(base_graph, head_graph)
    violations = [e for e in cross_edges.added 
                  if e.from_community != e.to_community]
    
    # 6. Cohesion delta
    cohesion = await graphify_cohesion(base_graph, head_graph)
    
    return {
        'blast_radius': blast_radius,
        'god_nodes': god_nodes,
        'sprawl': sprawl,
        'new_cycles': new_cycles,
        'violations': violations,
        'cohesion_delta': cohesion,
    }
```

### Step 5: Calculate Risk Score

```python
def calculate_risk(impact: ImpactAnalysis) -> RiskScore:
    score = 0.0
    
    # Blast radius risk (weight: 0.25)
    if impact.blast_radius > 15:
        score += 0.25
    elif impact.blast_radius > 5:
        score += 0.15
    else:
        score += 0.05
    
    # God node risk (weight: 0.25)
    score += 0.25 * (len(impact.god_nodes) / max_total_god_nodes)
    
    # Sprawl risk (weight: 0.20)
    if impact.sprawl > 3:
        score += 0.20
    elif impact.sprawl > 1:
        score += 0.10
    
    # Cycle risk (weight: 0.15)
    score += 0.15 * len(impact.new_cycles)
    
    # Coupling risk (weight: 0.15)
    score += 0.15 * len(impact.violations)
    
    return RiskScore(
        value=min(score, 1.0),
        level='low' if score < 0.3 else 'medium' if score < 0.6 else 'high'
    )
```

### Step 6: Generate Report

```python
def generate_report(risk: RiskScore, impact: ImpactAnalysis) -> str:
    # Overall risk
    risk_emoji = '🟢' if risk.level == 'low' else '🟡' if risk.level == 'medium' else '🔴'
    
    report = f"""## 📊 Graphify Architectural Review

**Overall risk:** {risk_emoji} {risk.level.title()}

### Key Findings

"""
    
    # Blast radius
    if impact.blast_radius > 5:
        report += f"- **Blast radius:** Changing affected files would impact **{impact.blast_radius} functions** in multiple files.\n"
    
    # God nodes
    if impact.god_nodes:
        nodes_str = ', '.join([f"`{n}`" for n in impact.god_nodes])
        report += f"- **God node hit:** {nodes_str} – high-risk change.\n"
    
    # Sprawl
    if impact.sprawl > 2:
        report += f"- **Community sprawl:** Changes touch **{impact.sprawl} communities** – consider splitting the PR.\n"
    
    # Violations
    if impact.violations:
        for v in impact.violations:
            report += f"- **New coupling:** An edge from `{v.from}` → `{v.to}` adds a layering violation.\n"
    
    # Cycles
    if impact.new_cycles:
        report += f"- **Circular dependencies:** {len(impact.new_cycles)} new cycle(s) introduced.\n"
    else:
        report += "- **Circular dependencies:** None introduced.\n"
    
    # Recommendations
    report += "\n### Recommendations\n\n"
    
    if impact.blast_radius > 10:
        report += "- ✅ Add more tests due to large blast radius.\n"
    
    if impact.sprawl > 2:
        report += "- ⚠️ Break the PR into smaller, focused changes.\n"
    
    if impact.god_nodes:
        report += "- ⚠️ Extra review required for god node modifications.\n"
    
    if impact.new_cycles:
        report += "- 🚫 Resolve circular dependencies before merging.\n"
    
    return report
```

### Step 7: Post Comment

```python
async def post_pr_comment(pr_number: int, report: str):
    # GitHub
    await github_client.create_review_comment(
        repo=repo,
        pr_number=pr_number,
        body=report,
    )
    
    # GitLab
    await gitlab_client.create_merge_request_note(
        project_id=project_id,
        merge_request_iid=pr_number,
        body=report,
    )
```

---

## Example Agent-Generated Comment

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

## Separation of Concerns

| Layer | Responsibility |
|-------|----------------|
| **Graphify Core** | Deterministic graph operations (centrality, communities, diff, cycles) |
| **Agent** | Orchestration, interpretation, natural language generation |

This separation ensures:
- **Trustworthiness:** All metrics are computed from the graph, not guessed
- **Consistency:** Same inputs always produce same outputs
- **Flexibility:** Agent can be updated without changing core logic

---

## Summary

The agent integration layer:

1. **Receives triggers** from webhooks, commands, or CI
2. **Calls Graphify MCP tools** to compute metrics
3. **Aggregates results** into a unified impact analysis
4. **Calculates risk scores** based on weighted metrics
5. **Generates human-readable reports** with findings and recommendations
6. **Posts PR comments** automatically

This transforms Graphify from a passive documentation tool into an active architectural gatekeeper—all without requiring developers to learn new commands or interpret raw graph data.