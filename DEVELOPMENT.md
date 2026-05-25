# Development Notes — How This Workflow Was Built

This document explains the development process behind the AutoDoc Pipeline — specifically how the workflow was designed, built, debugged, and deployed using **Claude Code** and the **n8n MCP server** directly from VS Code, without manually building nodes in the n8n UI.

---

## The Development Stack

| Tool | Role |
|---|---|
| **VS Code** | Editor and terminal |
| **Claude Code** | AI coding assistant (Anthropic CLI, running inside VS Code) |
| **n8n MCP server (`n8n-mcp`)** | Bridge between Claude and the live n8n instance |
| **n8n Cloud** | The workflow runtime |

---

## What Is the n8n MCP Server?

MCP (Model Context Protocol) is a standard that lets AI assistants connect to external tools. The `n8n-mcp` server exposes two categories of tools directly to Claude:

**Knowledge tools** (no n8n instance needed — instant):
- `search_nodes` — find any of n8n's 1,650+ nodes by keyword
- `get_node` — read a node's full documentation, operations, and property schema
- `validate_node` — check a node configuration for errors before adding it
- `search_templates` — search 2,352+ official workflow templates
- `validate_workflow` — validate an entire workflow structure before deploying

**Instance management tools** (require live n8n API connection):
- `n8n_list_workflows` — list all workflows in the instance
- `n8n_get_workflow` — read a workflow's current state
- `n8n_create_workflow` — create a new workflow
- `n8n_update_partial_workflow` — make incremental changes (add/remove/update nodes, connections)
- `n8n_executions` — read execution logs and debug failed runs
- `n8n_validate_workflow` — validate a deployed workflow by ID

This means Claude can read documentation, build a workflow, deploy it, watch it fail, read the error logs, fix the node, and redeploy — all without the developer leaving VS Code.

---

## Setup

### 1. Install Claude Code

```bash
npm install -g @anthropic/claude-code
```

Open VS Code, install the Claude Code extension, and authenticate with your Anthropic account.

### 2. Configure the n8n MCP Server

Add the following to your Claude Code MCP configuration (`.claude/settings.json` or via the Claude Code settings panel):

```json
{
  "mcpServers": {
    "n8n-mcp": {
      "command": "npx",
      "args": ["-y", "n8n-mcp"],
      "env": {
        "N8N_API_URL": "https://your-n8n-instance.app.n8n.cloud",
        "N8N_API_KEY": "your-n8n-api-key"
      }
    }
  }
}
```

Your n8n API key is in your n8n instance under **Settings → API → Create API Key**.

Once configured, Claude can call n8n MCP tools directly within any conversation in VS Code.

---

## How the Workflow Was Built

### Step 1 — Describe the goal in plain English

The entire workflow was described to Claude in natural language:

> "I want a Slack slash command `/createguide` that takes Bitbucket PR URLs, fetches the code diff, sends it to Claude for analysis, posts a summary to Slack for human approval, then generates a full user guide and saves it to Google Docs."

Claude used this to plan the node architecture before writing anything.

### Step 2 — Discover the right nodes

For each integration, Claude ran `search_nodes` to find the correct node type, then `get_node` to read the full documentation and understand which parameters were required:

```
search_nodes("bitbucket")         → nodes-base.httpRequest (Bitbucket has no native node)
search_nodes("google docs")       → nodes-base.googleDocs
search_nodes("slack webhook")     → nodes-base.httpRequest (incoming webhooks)
search_nodes("wait")              → nodes-base.wait
```

### Step 3 — Build and deploy the workflow

Claude assembled the full workflow JSON and deployed it to the live n8n instance in one call using `n8n_create_workflow`. The workflow appeared immediately in the n8n Cloud dashboard.

### Step 4 — Iterate with partial updates

After the initial deployment, all further changes were made using `n8n_update_partial_workflow` — a diff-based tool that applies incremental operations without replacing the whole workflow. Operations available:

| Operation | What it does |
|---|---|
| `addNode` | Add a new node to the workflow |
| `removeNode` | Delete a node and clean up its connections |
| `updateNode` | Update a node's parameters |
| `addConnection` | Wire two nodes together |
| `removeConnection` | Disconnect two nodes |

Example: to update the URL on a single HTTP Request node, Claude called:

```json
{
  "type": "updateNode",
  "nodeName": "Fetch PR Metadata",
  "updates": {
    "parameters": {
      "url": "={{ 'https://api.bitbucket.org/2.0/repositories/...' }}"
    }
  }
}
```

This is far faster than replacing the entire workflow JSON on every change.

### Step 5 — Debug using execution logs

When a workflow run failed, Claude called `n8n_executions` to read the execution log directly — seeing which node failed, what the input was, and what the error message said. No need to open the n8n UI to find the error.

This was the primary debugging loop:
1. Trigger the workflow (via Slack or manual run)
2. `n8n_executions` — read the failure
3. `get_node` — check the correct parameter structure
4. `n8n_update_partial_workflow` — apply the fix
5. Re-trigger

---

## Key Technical Decisions Made During Development

### HTTP Request nodes instead of a Code node for Bitbucket

The original design used a single Code node with JavaScript to call the Bitbucket API. This was replaced with 4 dedicated HTTP Request nodes after discovering that n8n Cloud's external task runner sandbox blocks both `$helpers.httpRequest()` and `fetch()` inside Code nodes. Native HTTP Request nodes have no such restriction.

### OAuth2 instead of Basic Auth for Bitbucket

Switched from username + app password credentials to OAuth2 after the Basic Auth credential type caused a silent failure (the Authorization header was not being injected). OAuth2 tokens are scoped, automatically refresh, and are natively supported by n8n's credential system.

### Two-step Google Doc creation

The Google Docs node only sets the document title on creation. A separate `batchUpdate` API call (via HTTP Request node) is required to insert text content. This was discovered during a run where the document was created successfully but was completely blank.

### Slackbot crawl detection

Slack's own bot visits URLs in messages to generate link previews. This was triggering the Gate 2 resume webhook, advancing the workflow before any human had clicked Approve. A user-agent check was added to the `Build Guide Payload` Code node to detect and ignore bot requests.

---

## Workflow File

`workflow-template.json` in this repository is the sanitized export of the live workflow, with all credentials replaced by placeholder strings. To use it:

1. Replace all `YOUR_*` placeholders with real values
2. Import into n8n via **Workflows → Import from file**
3. Update each node's credential references to match your n8n credential names

The live workflow was never edited through the n8n UI — every node and connection was created and modified through Claude Code + n8n MCP tools.
