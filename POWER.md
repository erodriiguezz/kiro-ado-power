---
name: 'ado-task-manager'
displayName: 'Azure DevOps Task Manager'
description: 'Read user story context (description + acceptance criteria) into your Kiro session, then push a child task with the summarizer output back to ADO when done. Install, paste your PAT, done.'
keywords:
  [
    'azure devops',
    'ado',
    'work item',
    'task summary',
    'user story',
    'acceptance criteria',
    'child task',
    'kiro',
  ]
author: 'RevStar'
---

# Azure DevOps Task Manager Power

## Overview

This power connects Kiro CLI to Azure DevOps so developers can:

1. **Pull context** — fetch a user story's description and acceptance criteria to ground the coding session
2. **Push results** — create a child task under that user story with the implementation summary when done

No copy-paste. No browser tab-switching. The agent reads what needs to be built, helps you build it, then writes what was done back to ADO.

## Setup (2 minutes)

### Step 1: Generate a Personal Access Token

1. Go to `https://dev.azure.com/RevStarConsulting/_usersSettings/tokens`
2. Click **+ New Token**
3. Settings:
   - **Name**: `Kiro CLI`
   - **Organization**: `RevStarConsulting`
   - **Scopes**: select **Custom defined**, then check:
     - ✅ **Work Items**: Read & Write
   - **Expiration**: 90 days
4. Click **Create** and copy the token (shown only once)

### Step 2: Add to your shell

```bash
echo 'export AZURE_DEVOPS_EXT_PAT="paste-your-token-here"' >> ~/.zshrc
source ~/.zshrc
```

### Step 3: Install this power in Kiro

Done. The MCP server launches automatically on first use.

## How It Works

### Reading Context (Start of Session)

When you start working on a user story, tell the agent:

> "Pull the context for user story #1234"

The agent calls `wit_work_item` (action: `get`) to fetch:
- Title
- Description (the "what" and "why")
- Acceptance criteria
- Current state and tags

This grounds the entire session — the agent knows exactly what needs to be built and what "done" looks like.

### Writing Results (End of Session)

When work is complete, the summarizer generates a summary and the agent creates a child task:

> "Push a task summary to user story #1234"

The agent calls `wit_work_item_write` (action: `add_child`) to create a child Task with:
- Title derived from what was implemented
- Description containing the structured summary (changes, verification, notes)
- Linked to the parent user story automatically

### Automatic Delivery

A hook fires after the summarizer agent completes. It:
1. Checks the git branch for a work item ID (e.g. `feature/1234-auth-flow` → `#1234`)
2. Creates a child task under that user story with the summary
3. Confirms delivery to the developer

If no ID can be inferred from the branch, it asks.

## Available Tools

| Tool | Action | Purpose |
|------|--------|---------|
| `wit_work_item` | `get` | Read a user story (description, acceptance criteria) |
| `wit_work_item` | `get_batch` | Read multiple work items at once |
| `wit_work_item_write` | `add_child` | Create a child task under a user story |
| `wit_work_item_write` | `update` | Update fields on an existing work item |
| `wit_work_item_comment_write` | `add` | Add a comment to a work item |
| `wit_work_item_link_write` | `link_to_pull_request` | Link a PR to the work item |

## Security

- **PAT is per-developer** — actions are attributed to the individual in ADO
- **PAT is never stored in power files** — lives only in the developer's shell environment
- **Minimum scope** — only Work Items Read & Write (nothing else)
- **PAT expires** — rotate before expiration at the same tokens URL

## Token Rotation

When your PAT expires:
1. Generate a new one (same steps as setup)
2. Update the value in `~/.zshrc`
3. `source ~/.zshrc`

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "PERSONAL_ACCESS_TOKEN not set" | Ensure `AZURE_DEVOPS_EXT_PAT` is exported in `~/.zshrc` and you ran `source ~/.zshrc` |
| "Not authorized" (TF400813) | PAT scope is too narrow — regenerate with Work Items Read & Write |
| "npx not found" | Install Node.js 20+: `brew install node` |
| Tools don't appear | Restart Kiro CLI so it inherits the updated shell environment |
| Browser login popup | You're missing `-a pat` in the config — this power includes it |

## Steering

See `steering/task-summary-delivery.md` for formatting rules, the decision tree, and example tool calls.
