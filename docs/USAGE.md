# Usage Guide

> **How to use Graphify Pre-Merge Architectural Review in your workflow.**

---

## Quick Start

### Basic PR Analysis

```bash
# Analyze a PR against main branch
graphify pr --base main --head feature-branch

# Analyze with specific repository
graphify pr --base main --head feature-branch --repo owner/repo
```

### Generate PR Comment

```bash
# Generate and post a PR comment
graphify pr --base main --head feature-branch --comment

# Use agent-powered analysis
graphify pr --base main --head feature-branch --agent opencode
```

---

## Command Reference

### `graphify pr`

Analyze a pull request for architectural impact.

```bash
graphify pr [OPTIONS]
```

**Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `--base <branch>` | Target branch (e.g., `main`, `develop`) | `main` |
| `--head <branch>` | Source branch (the PR branch) | Required |
| `--repo <owner/repo>` | Repository in `owner/repo` format | Auto-detected |
| `--comment` | Post results as PR comment | `false` |
| `--agent <agent>` | Use agent for analysis (e.g., `opencode`) | None |
| `--output <format>` | Output format: `json`, `markdown`, `text` | `markdown` |

**Examples:**

```bash
# Basic analysis
graphify pr --base main --head feature/auth

# Post comment to GitHub PR
graphify pr --base main --head feature/auth --comment

# JSON output for CI integration
graphify pr --base main --head feature/auth --output json

# Agent-powered analysis
graphify pr --base main --head feature/auth --agent opencode --comment
```

---

### `graphify diff`

Compare two graphs and output the difference.

```bash
graphify diff --base <graph.json> --head <graph.json> [OPTIONS]
```

**Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `--base <file>` | Base graph JSON file | Required |
| `--head <file>` | Head graph JSON file | Required |
| `--output <format>` | Output format: `json`, `text` | `json` |

**Examples:**

```bash
# Compare two graphs
graphify diff --base graph-main.json --head graph-pr.json

# Text output
graphify diff --base graph-main.json --head graph-pr.json --output text
```

---

### `graphify affected`

Calculate blast radius for changed nodes.

```bash
graphify affected <node> [OPTIONS]
```

**Arguments:**

| Argument | Description |
|----------|-------------|
| `<node>` | Node label or ID to analyze |

**Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `--depth <n>` | Maximum traversal depth | `10` |
| `--format <format>` | Output format: `json`, `tree`, `list` | `list` |

**Examples:**

```bash
# Find all dependents of a file
graphify affected src/database.py

# JSON output with depth limit
graphify affected src/database.py --depth 5 --format json

# Tree view
graphify affected src/database.py --format tree
```

---

### `graphify centrality`

Get centrality metrics for a node.

```bash
graphify centrality <node> [OPTIONS]
```

**Arguments:**

| Argument | Description |
|----------|-------------|
| `<node>` | Node label or ID |

**Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `--metric <type>` | Metric type: `betweenness`, `degree`, `closeness`, `all` | `all` |
| `--threshold <n>` | God node threshold | `0.70` |

**Examples:**

```bash
# Get all centrality metrics
graphify centrality src/auth.py

# Get only betweenness centrality
graphify centrality src/auth.py --metric betweenness

# Check if god node (centrality > 0.70)
graphify centrality src/auth.py --threshold 0.70
```

---

### `graphify community`

Get community information.

```bash
graphify community [COMMAND] [OPTIONS]
```

**Subcommands:**

| Command | Description |
|---------|-------------|
| `list` | List all communities |
| `nodes <community>` | Get nodes in a community |
| `map <files>` | Map files to communities |

**Examples:**

```bash
# List all communities
graphify community list

# Get nodes in community 0
graphify community nodes 0

# Map changed files to communities
graphify community map src/ui.py src/db.py src/auth.py
```

---

### `graphify cycles`

Detect circular dependencies.

```bash
graphify cycles [OPTIONS]
```

**Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `--graph <file>` | Graph JSON file to analyze | Current graph |
| `--type <type>` | Cycle type: `import`, `call`, `all` | `all` |
| `--new-only` | Only show new cycles (vs base) | `false` |

**Examples:**

```bash
# Find all cycles
graphify cycles

# Find only import cycles
graphify cycles --type import

# Find new cycles introduced by PR
graphify cycles --graph graph-pr.json --new-only --base graph-main.json
```

---

### `graphify cohesion`

Calculate community cohesion.

```bash
graphify cohesion [OPTIONS]
```

**Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `--community <id>` | Specific community ID | All communities |
| `--delta` | Compare with base graph | `false` |
| `--base <file>` | Base graph for delta comparison | Required if `--delta` |

**Examples:**

```bash
# Get cohesion for all communities
graphify cohesion

# Get cohesion for specific community
graphify cohesion --community 0

# Compare cohesion before/after
graphify cohesion --delta --base graph-main.json --head graph-pr.json
```

---

### `graphify list-prs`

List open PRs with impact data.

```bash
graphify list-prs [OPTIONS]
```

**Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `--repo <owner/repo>` | Repository | Auto-detected |
| `--base <branch>` | Base branch filter | Auto-detected |
| `--format <format>` | Output format: `json`, `table` | `table` |

**Examples:**

```bash
# List all open PRs
graphify list-prs

# Filter by base branch
graphify list-prs --base develop

# JSON output
graphify list-prs --format json
```

---

### `graphify triage-prs`

Get actionable PRs for review (prioritized by risk).

```bash
graphify triage-prs [OPTIONS]
```

**Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `--repo <owner/repo>` | Repository | Auto-detected |
| `--base <branch>` | Base branch filter | Auto-detected |
| `--limit <n>` | Maximum PRs to return | `10` |

**Examples:**

```bash
# Get top 10 PRs needing review
graphify triage-prs

# Get top 5
graphify triage-prs --limit 5
```

---

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/graphify-review.yml
name: Graphify Architectural Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Setup Graphify
        uses: graphify/setup@v1
      
      - name: Run Graphify Review
        run: |
          graphify pr \
            --base ${{ github.base_ref }} \
            --head ${{ github.head_ref }} \
            --repo ${{ github.repository }} \
            --comment
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### GitLab CI

```yaml
# .gitlab-ci.yml
graphify-review:
  stage: test
  image: graphify/cli:latest
  script:
    - graphify pr --base $CI_MERGE_REQUEST_TARGET_BRANCH_NAME --head $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME --comment
  only:
    - merge_requests
  variables:
    GITLAB_TOKEN: $GITLAB_TOKEN
```

### Jenkins Pipeline

```groovy
pipeline {
  agent any
  
  stages {
    stage('Graphify Review') {
      when {
        changeRequest()
      }
      steps {
        sh '''
          graphify pr \
            --base ${CHANGE_TARGET} \
            --head ${CHANGE_BRANCH} \
            --comment
        '''
      }
    }
  }
}
```

---

## Webhook Integration

### GitHub Webhook

```json
{
  "name": "graphify-review",
  "active": true,
  "events": ["pull_request"],
  "config": {
    "url": "https://your-server.com/graphify/webhook",
    "content_type": "json"
  }
}
```

### GitLab Webhook

```json
{
  "url": "https://your-server.com/graphify/webhook",
  "merge_requests_events": true,
  "token": "your-secret-token"
}
```

---

## Agent Integration

### Using OpenCode Agent

```bash
# Trigger agent-powered analysis
graphify pr --base main --head feature/auth --agent opencode --comment
```

The agent will:
1. Build both graphs
2. Compute all metrics
3. Generate a human-readable report
4. Post the comment automatically

### Custom Agent Integration

```python
import asyncio
from graphify import GraphifyClient

async def analyze_pr(pr_number: int, base: str, head: str):
    client = GraphifyClient()
    
    # Build graphs
    base_graph = await client.build_graph(branch=base)
    head_graph = await client.build_graph(branch=head)
    
    # Compute diff
    diff = await client.diff(base_graph, head_graph)
    
    # Analyze impact
    impact = await client.analyze_impact(diff)
    
    # Generate report
    report = client.generate_report(impact)
    
    # Post comment
    await client.post_comment(pr_number, report)

# Run
asyncio.run(analyze_pr(123, "main", "feature/auth"))
```

---

## Configuration

### Configuration File

```yaml
# graphify.yaml
version: 1

# Graph construction
graph:
  exclude:
    - "node_modules/**"
    - "vendor/**"
    - "*.test.js"
  include:
    - "src/**"
    - "lib/**"

# Community detection
community:
  algorithm: leiden
  resolution: 1.0

# Centrality thresholds
centrality:
  god_node_threshold: 0.70

# Risk scoring
risk:
  weights:
    blast_radius: 0.25
    god_node: 0.25
    sprawl: 0.20
    cycles: 0.15
    coupling: 0.15
  
  thresholds:
    low: 0.3
    medium: 0.6
    high: 1.0

# Agent settings
agent:
  name: opencode
  auto_comment: true
  include_recommendations: true
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `GRAPHIFY_TOKEN` | API token for authentication | None |
| `GITHUB_TOKEN` | GitHub API token | None |
| `GITLAB_TOKEN` | GitLab API token | None |
| `GRAPHIFY_CONFIG` | Path to config file | `./graphify.yaml` |

---

## Output Formats

### JSON Output

```json
{
  "risk_score": 0.45,
  "risk_level": "medium",
  "findings": {
    "blast_radius": 23,
    "god_nodes": ["auth.py"],
    "community_sprawl": 3,
    "new_cycles": 0,
    "cross_community_edges": 1
  },
  "recommendations": [
    "Add more tests for database.py due to large blast radius",
    "Break the PR into separate UI and DB changes"
  ]
}
```

### Markdown Output

```markdown
## 📊 Graphify Architectural Review

**Overall risk:** ⚠️ Medium

### Key Findings

- **Blast radius:** 23 functions affected
- **God node hit:** auth.py (centrality 0.87)
- **Community sprawl:** 3 communities touched
- **New coupling:** ui/button.py → db/query.py

### Recommendations

- ✅ Add more tests for database.py
- ⚠️ Break the PR into smaller changes
```

---

## Best Practices

### 1. Run on Every PR

```yaml
# Run on all PRs
on:
  pull_request:
    types: [opened, synchronize]
```

### 2. Set Appropriate Thresholds

```yaml
# Adjust thresholds for your codebase
centrality:
  god_node_threshold: 0.70  # Adjust based on your architecture

risk:
  thresholds:
    low: 0.3
    medium: 0.6
    high: 1.0
```

### 3. Use Agent for Complex Analysis

```bash
# For complex PRs, use agent-powered analysis
graphify pr --base main --head feature/complex --agent opencode --comment
```

### 4. Review High-Risk PRs Manually

```yaml
# Block high-risk PRs
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - name: Run Graphify
        id: graphify
        run: |
          result=$(graphify pr --base main --head $BRANCH --output json)
          echo "risk_level=$(echo $result | jq -r '.risk_level')" >> $GITHUB_OUTPUT
      
      - name: Check Risk Level
        if: steps.graphify.outputs.risk_level == 'high'
        run: |
          echo "High risk PR requires manual approval"
          exit 1
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| "Graph not found" | Ensure the branch exists and is accessible |
| "Permission denied" | Check `GITHUB_TOKEN` or `GITLAB_TOKEN` permissions |
| "Timeout" | Increase timeout for large repositories |
| "No communities detected" | Ensure graph has enough nodes (>10) |

### Debug Mode

```bash
# Enable debug logging
graphify pr --base main --head feature --debug

# Verbose output
graphify pr --base main --head feature --verbose
```

---

## Summary

| Task | Command |
|------|---------|
| Analyze PR | `graphify pr --base main --head feature` |
| Post PR comment | `graphify pr --base main --head feature --comment` |
| Use agent | `graphify pr --base main --head feature --agent opencode` |
| Calculate blast radius | `graphify affected <node>` |
| Check centrality | `graphify centrality <node>` |
| Find communities | `graphify community list` |
| Detect cycles | `graphify cycles` |
| List PRs | `graphify list-prs` |
| Triage PRs | `graphify triage-prs` |