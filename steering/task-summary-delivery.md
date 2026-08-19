# Task Summary Delivery to Azure DevOps

This steering file describes the workflow the agent should follow when a user explicitly asks to create a task, deliver a summary, or update a work item in Azure DevOps.

## MCP Server

This power uses `@tiberriver256/mcp-server-azure-devops` (stdio transport, PAT auth via `AZURE_DEVOPS_PAT` environment variable). The server is pre-configured in `mcp.json`.

## ⛔ NO AUTOMATIC DELIVERY

There is NO automatic hook. Tasks are ONLY created when the user explicitly asks. If the user says "create a task", "push summary to ADO", "log this work" — proceed. Otherwise, do nothing related to ADO writes.

## When to Use This File

Load this file when the user's request matches:

- "Create a task under user story #1234"
- "Push a summary to the ticket"
- "Log this work to ADO"
- "Update the ticket with what we did"

Do NOT use this for general ADO queries ("what's assigned to me", "show open PRs").

## Step-by-Step Workflow

### Step 1: Collect Required Fields

Before ANY API call, confirm these with the user:

| Field | Required | Example |
|-------|----------|---------|
| Parent work item ID | Yes | `#136662` (from user or branch name `feat/136662-*`) |
| Role | Yes | `Dev`, `DevOps`, `AI` |
| Full Name | Yes | `Ernesto Rodriguez` |
| Activity | Yes | `Development`, `Testing`, `Design`, `Documentation`, `Deployment` |
| Team Code | Yes | provided by user |

If ANY field is missing, ask:
```
I need the following to create the task:
✓ Parent: #136662
✓ Role: Dev
✓ Name: Ernesto Rodriguez
✗ Activity: (Development, Testing, Design, Documentation, or Deployment?)
✗ Team Code: ?
```

Do NOT proceed until all fields are confirmed.

### Step 2: Identify the parent work item

- If user gave an ID, use it directly.
- If no ID given, check branch name for `feat/<id>-*` pattern.
- If ambiguous, ask the user. Never guess.

### Step 3: Fetch the parent work item

```
get_work_item({ id: <parent_id>, project: "<project>" })
```

Confirm it exists and note its project and title.

### Step 4: Generate the summary

Use the **breakdown format** for the task description:

```markdown
## Summary
[1-3 sentences: what changed in plain language, user/system impact first]

## Changes
### [Feature Area / Category]
- [Plain-language description of what changed and why]
  - `path/to/file.ts` (L12–L45)

### [Next Category]
- [Next change]

## Notes / Risks
- **[Risk or caveat]** — explanation

## Testing
- [What was verified and how]
```

### Step 5: Create the child task

```json
{
  "tool": "create_work_item",
  "input": {
    "project": "<project>",
    "type": "Task",
    "title": "<Role> - <Full Name> - <MMDDYYYY>",
    "description": "<formatted summary from Step 4>",
    "additionalFields": {
      "Microsoft.VSTS.Common.Activity": "<activity>",
      "Microsoft.VSTS.Scheduling.OriginalEstimate": 0,
      "Microsoft.VSTS.Scheduling.RemainingWork": 0,
      "Microsoft.VSTS.Scheduling.CompletedWork": 0,
      "Custom.TeamCode": "<team_code>"
    }
  }
}
```

**Time fields are ALWAYS `0`.** No exceptions. See `steering/time-tracking.md`.

### Step 6: Link to parent

```json
{
  "tool": "manage_work_item_link",
  "input": {
    "sourceId": "<new_task_id>",
    "targetId": "<parent_id>",
    "linkType": "System.LinkTypes.Hierarchy-Reverse",
    "operation": "add",
    "project": "<project>"
  }
}
```

### Step 7: Confirm and print URL

After successful creation, ALWAYS output:

```
✅ Task created: Dev - Ernesto Rodriguez - 08182026
   ID: #<new_id>
   Parent: #<parent_id> (<parent_title>)
   
   🔗 https://dev.azure.com/RevStarConsulting/<project>/_workitems/edit/<new_id>
   
   Update your hours and status directly in the ticket above.
```

## Error Handling

- **Work item not found**: Confirm the ID with the user.
- **Permission denied on write**: Report that the PAT lacks edit permission.
- **Field validation error**: Check valid field values, correct, and retry once. Explain to user.
- **Missing required fields**: Ask the user. Never guess or use defaults for Role, Name, Activity, or Team Code.

## Date Format

Task title dates use `MMDDYYYY` — no separators, no slashes, no dashes.
- ✅ `08182026`
- ❌ `08/18/2026`
- ❌ `08-18-2026`
- ❌ `2026-08-18`
