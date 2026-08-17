---
name: 'kiro-ado-power'
version: '1.2.1'
displayName: 'Azure DevOps Task Manager'
description: 'Read user story context (description + acceptance criteria) into your Kiro session, then push a child task with the summarizer output back to ADO when done.'
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
    'ticket',
    'sprint',
  ]
author: 'RevStar'
---

# Azure DevOps Task Manager Power

## Overview

This power connects Kiro to Azure DevOps so developers can:

1. **Pull context** — fetch a user story's description and acceptance criteria to ground the coding session
2. **Push results** — create a child task under that user story with the implementation summary when done

## Reading Context

When the user asks to pull context for a work item, use `get_work_item` to fetch:
- Title
- Description (the "what" and "why")
- Acceptance criteria
- Current state and tags

Example prompts:
- "Pull the context for user story #1234"
- "What are the acceptance criteria for #5678?"

## Writing Results

When work is complete, create a child task under the parent user story:

1. Use `create_work_item` with type `Task`
2. Set the title to a concise description of what was implemented
3. Set the description to the structured summary (changes, verification, notes) in ADO-flavored markdown
4. Use `manage_work_item_link` to link the child to the parent user story

## Automatic Delivery

A hook fires after the summarizer agent completes:
1. Resolves the work item ID from the git branch name (e.g. `feature/1234-auth-flow` → `#1234`)
2. Creates a child task under that user story with the summary
3. Confirms delivery to the developer

If no ID can be inferred from the branch, ask the user.

## Available Tools

| Tool | Purpose |
|------|---------|
| `get_work_item` | Read a user story (description, acceptance criteria) |
| `list_work_items` | List work items in a project |
| `search_work_items` | Full-text search for work items |
| `create_work_item` | Create a new work item (child task with summary) |
| `update_work_item` | Update fields on an existing work item |
| `manage_work_item_link` | Link work items (parent/child) or link to PR |
| `list_projects` | List all projects in the org |
| `create_pull_request` | Create a PR and link to work item |

## Best Practices

- **Never guess a work item ID.** Search or infer from the branch name. Confirm with the user if ambiguous.
- **Comments over description edits.** Comments preserve history. Only update the description when explicitly asked.
- **Read before write.** Always fetch the current work item state before modifying it.
- **Keep summaries scoped.** Only include work relevant to that specific ticket.
- **ADO markdown subset.** Use `##`/`###` headers, `-` bullets, `**bold**`, fenced code blocks. Avoid raw HTML.

## Steering

See `steering/task-summary-delivery.md` for detailed formatting rules, the delivery decision tree, and example tool calls.
