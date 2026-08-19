---
name: 'kiro-ado-power'
version: '1.3.0'
displayName: 'Azure DevOps Task Manager'
description: 'Read user story context (description + acceptance criteria) into your Kiro session, then create a child task with the implementation summary when the user asks.'
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
2. **Push results** — create a child task under that user story with the implementation summary **only when the user explicitly asks**

## Reading Context

When the user asks to pull context for a work item, use `get_work_item` to fetch:
- Title
- Description (the "what" and "why")
- Acceptance criteria
- Current state and tags

Example prompts:
- "Pull the context for user story #1234"
- "What are the acceptance criteria for #5678?"

## Writing Results — USER-INITIATED ONLY

**NEVER automatically create or push tickets.** Only create a child task when the user explicitly asks:
- "Create a task for this work"
- "Push a summary to user story #1234"
- "Log this as a task under #1234"

When the user asks, create a child task under the parent user story:

1. **Collect required fields FIRST** — before calling any API:
   - Role (Dev, DevOps, or AI)
   - Developer's full name
   - Activity (Development, Testing, Design, Documentation, Deployment)
   - Team Code (billing code)
   
   If ANY are missing, ask. Do not guess. Do not proceed without them.

2. Use `create_work_item` with type `Task`
3. Title format: `<Role> - <Full Name> - <MMDDYYYY>` (today's date, no separators)
4. **ALL time fields MUST be `0`** — Original Estimate, Remaining Work, Completed Work. No exceptions.
5. Use `manage_work_item_link` to link the child to the parent user story

6. **After creation, ALWAYS print the ticket URL:**
   ```
   ✅ Task created: https://dev.azure.com/RevStarConsulting/<project>/_workitems/edit/<id>
   
   Update your hours and status directly in the ticket above.
   ```

## Branch Naming

Branch prefix convention is `feat/` (not `feature/`):
```
feat/1234-auth-flow
feat/5678-payment-integration
bugfix/9012-null-check
```

## Available Tools

| Tool | Purpose |
|------|---------|
| `get_work_item` | Read a user story (description, acceptance criteria) |
| `list_work_items` | List work items in a project |
| `search_work_items` | Full-text search for work items |
| `create_work_item` | Create a new work item (child task) — USER MUST ASK |
| `update_work_item` | Update fields on an existing work item — USER MUST ASK |
| `manage_work_item_link` | Link work items (parent/child) or link to PR |
| `list_projects` | List all projects in the org |

## Rules — NON-NEGOTIABLE

1. **NEVER auto-push tickets.** Only create work items when the user explicitly requests it.
2. **NEVER log time.** All hour fields are always `0`. Do not ask for, accept, calculate, or infer time values.
3. **ALWAYS collect required fields before creating.** Role, Name, Activity, Team Code. If missing, ask.
4. **ALWAYS print the ticket URL after creation** so the user can update hours/status manually.
5. **ALWAYS use `feat/` prefix** for feature branches (not `feature/`).
6. **Read before write.** Always fetch the current work item state before modifying it.
7. **Never guess a work item ID.** Search or infer from branch name. Confirm if ambiguous.

## Summary Format

When creating a task description, use the breakdown structure from `steering/task-summary-delivery.md`. The summary MUST follow this format:

```
## Summary
[1-3 sentences: what changed, impact]

## Changes
### [Category]
- [Plain-language description]
  - `path/to/file` (L12-L45)

## Notes / Risks
- **[Risk]** — explanation

## Testing
- [What was verified]
```

## Steering

See `steering/task-summary-delivery.md` for detailed formatting rules and `steering/time-tracking.md` for time field enforcement.
