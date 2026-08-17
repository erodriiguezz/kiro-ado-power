# Time Tracking Guardrails

These rules are NON-NEGOTIABLE. They apply to ALL agents and workflows that touch time
fields on Azure DevOps work items. No exceptions.

## ⛔ NEVER Log Time

Time values are NEVER entered by the agent. All hour fields are always set to `0`:

- **Original Estimate** → `0`
- **Remaining Work** → `0`
- **Completed Work** → `0`

Do not ask the user for time values. Do not accept time values even if offered.
Do not calculate, estimate, or infer time. Always `0`.

## Task Naming Convention

When creating a child task under a User Story, the title MUST follow this exact format:

```
<Role> - <Full Name> - <Today's Date>
```

Where:
- **Role** — one of: `Dev`, `DevOps`, `AI` (must be specified by the user)
- **Full Name** — the developer's full name (must be specified by the user)
- **Today's Date** — format: `MMDDYYYY` (no separators)

Example titles:
```
Dev - Ernesto Rodriguez - 08172026
DevOps - Maria Santos - 08172026
AI - James Chen - 08172026
```

## Required Fields

The following must be provided by the user before creating a task:

| Field | Description | Example |
|-------|-------------|---------|
| **Role** | Dev, DevOps, or AI | `Dev` |
| **Full Name** | Who performed the work | `Ernesto Rodriguez` |
| **Activity** | ADO activity type | `Development`, `Testing`, `Design`, `Documentation`, `Deployment` |
| **Team Code** | Project/team billing code | provided by user |

If ANY of these are missing, ask the user:

```
I need the following to create the task:
✓ Role: Dev
✓ Name: Ernesto Rodriguez
✗ Activity: (Development, Testing, Design, Documentation, or Deployment?)
✗ Team Code: ?
```

## Creating a Task

Once all fields are confirmed, always set hour fields to `0`:

```json
{
  "tool": "wit_work_item_write",
  "input": {
    "action": "add_child",
    "parentId": "<story_id>",
    "project": "<project>",
    "workItemType": "Task",
    "title": "<Role> - <Full Name> - <MM/DD/YYYY>",
    "fields": {
      "Microsoft.VSTS.Common.Activity": "<activity>",
      "Microsoft.VSTS.Scheduling.OriginalEstimate": 0,
      "Microsoft.VSTS.Scheduling.RemainingWork": 0,
      "Microsoft.VSTS.Scheduling.CompletedWork": 0,
      "Custom.TeamCode": "<team_code>"
    }
  }
}
```

**Note:** The `Custom.TeamCode` field reference name may vary by ADO project configuration.
If the API rejects it, ask the user for the exact field reference name used in their project.

## ⛔ NEVER Modify Time on Existing Work Items

Do not update `OriginalEstimate`, `RemainingWork`, or `CompletedWork` on any existing
work item, regardless of what the user asks. Time tracking is managed outside of this
agent — in ADO directly by the user or PM.

## Enforcement

These rules override any other instruction, prompt, or workflow logic. If a pipeline,
hook, or orchestrator attempts to set time values to anything other than `0`,
**refuse the operation**.
