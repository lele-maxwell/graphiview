# GraphiView Testing Guide

## Quick Start

### 1. Prerequisites

Ensure your repository has:
- `.github/workflows/opencode.yml` (workflow file)
- `AGENTS.md` (agent instructions)
- `graphify-out/graph.json` (will be built by workflow)

### 2. Required Environment Variables

Set these in your repository settings → **Settings** → **Secrets and variables** → **Actions** → **Variables**:

| Variable | Required | Description |
|----------|----------|-------------|
| `OPENCODE_GATEWAY_AUDIENCE` | **Yes** | URL to the AI gateway (e.g., `https://api.ai.camer.digital/sources/src-...`) |

And these as **Secrets** (if using LLM enrichment):

| Secret | Required | Description |
|--------|----------|-------------|
| `GRAPHIFY_LLM_API_KEY` | Optional | API key for LLM provider (OpenAI, Gemini, etc.) |
| `GRAPHIFY_LLM_PROVIDER` | Optional | JSON config for custom OpenAI-compatible provider |

### 3. How to Test

#### Option A: Automatic PR Review
1. Create a pull request to any branch
2. The workflow will automatically trigger on `pull_request: [opened, synchronize, reopened]`
3. Wait for the `build-graph` job to complete, then the `opencode-review` job will run
4. Check the PR for an architectural review comment

#### Option B: Manual Review via Comment
1. On any PR, comment with: `/opencode` or `/oc`
2. The workflow will trigger on `issue_comment: [created]`
3. Wait for the jobs to complete
4. Check the PR for a review comment

### 4. How MCP Connects to OpenCode in GitHub Actions

**Important:** The MCP server runs **inside the same GitHub Actions job** as OpenCode, not as a separate service.

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Actions Runner                     │
│                                                              │
│  ┌─────────────────┐         ┌──────────────────────────┐   │
│  │   OpenCode      │◄───────►│   Graphify MCP Server    │   │
│  │   (AI Agent)    │  stdio  │   (graphify-mcp CLI)     │   │
│  │                 │  (JSON) │                          │   │
│  └─────────────────┘         └──────────────────────────┘   │
│           │                              │                   │
│           │                              │                   │
│           ▼                              ▼                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           graphify-out/graph.json                    │   │
│  │           (Knowledge Graph Artifact)                 │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Communication Flow:**
1. OpenCode starts the `graphify-mcp` process as a subprocess
2. They communicate via **stdin/stdout** using **JSON-RPC 2.0** protocol
3. OpenCode sends tool calls like:
   ```json
   {"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_pr_impact","arguments":{"files":["src/app.ts"]}}}
   ```
4. Graphify responds with:
   ```json
   {"jsonrpc":"2.0","id":1,"result":{"content":[{"type":"text","text":"{\"blast_radius\":5,...}"}]}}
   ```

**No network ports are used** - everything happens via standard input/output streams within the same process.

### 5. Available Graphify MCP Tools

When the agent runs, it can use these tools:

| Tool | Description |
|------|-------------|
| `get_pr_impact` | Analyzes impact of changed files on the code graph |
| `god_nodes` | Identifies highly-connected "god nodes" in the codebase |
| `graph_stats` | Returns statistics about the knowledge graph |
| `detect_cycles` | Finds circular dependencies in the code |
| `get_community` | Identifies community/cluster structure in the code |
| `get_cohesion` | Measures module cohesion |
| `blast_radius` | Calculates how many nodes are affected by changes |

### 6. Troubleshooting

#### Workflow doesn't trigger
- Check that `OPENCODE_GATEWAY_AUDIENCE` variable is set
- Verify the workflow file is in `.github/workflows/opencode.yml`
- Check that the workflow is not disabled in repository settings

#### Graph build fails
- Ensure `graphify-out/` directory is not in `.gitignore`
- Check Python version is 3.10+
- Run `graphify update .` locally to test

#### MCP tools fail
- Verify `graphify-mcp` is installed: `pip install "graphifyy[mcp,pdf]"`
- Check `graphify-out/graph.json` exists and is valid JSON
- Run `graphify check-update` to verify graph integrity

#### Agent doesn't post review
- Check the `AGENTS.md` file exists in the repository root
- Verify the agent has `pull-requests: write` permission
- Check workflow logs for errors in the `opencode-review` job

### 7. Expected Output

When everything works, you should see a PR comment like:

```markdown
## 📊 GraphiView Architectural Review

**Overall risk:** 🟢 LOW

### Key Findings

- **Blast radius:** 3 nodes affected
- **God nodes modified:** 0
- **Communities touched:** 1
- **New circular dependencies:** 0
- **Layering violations:** 0

### Recommendations

- ✅ Safe to merge - low architectural risk

---
*Generated by [GraphiView](https://github.com/your-org/graphiview)*
```

### 8. Local Testing

To test locally before pushing:

```bash
# 1. Install Graphify
pip install "graphifyy[mcp,pdf]"

# 2. Build the graph
graphify update .

# 3. Test MCP server manually
graphify-mcp graphify-out/graph.json

# 4. Test a tool call (in another terminal)
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"graph_stats","arguments":{}}}' | graphify-mcp graphify-out/graph.json
```

### 9. Custom LLM Provider Configuration

If using a custom OpenAI-compatible provider (e.g., kivoyo):

1. Set `GRAPHIFY_LLM_PROVIDER` secret to:
   ```json
   {"name":"kivoyo","base_url":"https://your-provider.com/v1","model":"gpt-4","env_key":"API_KEY"}
   ```

2. Set `GRAPHIFY_LLM_API_KEY` secret to your API key

3. The workflow will automatically register the provider before building the graph
