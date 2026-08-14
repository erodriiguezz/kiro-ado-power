# Task Summary Delivery to Azure DevOps

This steering file describes the workflow the agent should follow when a user asks to send, deliver, post, or update a task summary, implementation report, or status update on an Azure DevOps work item.

## MCP Server

This power uses the local `@azure-devops/mcp` server (stdio transport, PAT auth via `AZURE_DEVOPS_EXT_PAT` environment variable). The server is pre-configured in `mcp.json` with toolsets: `core`, `wit`, `work`. All tool calls go through this local server — NOT the remote `mcp.dev.azure.com` endpoint.

## Automatic Trigger

This workflow is not only invoked when the user explicitly asks for it. A hook (`.kiro/hooks/ado-auto-update.json`) also triggers it automatically right after the `summarizer` agent produces a ticket-ready summary. In the automatic case, resolve the work item ID by checking the current git branch name first for a pattern like `feature/1234-*` or `1234-description` (any leading/embedded numeric ID), and only ask the user for the ID if none can be inferred that way. The rest of this workflow (formatting, comment-vs-description decision, confirmation) applies the same whether triggered manually or automatically.

## When to Load This File

Load this file when the user's request matches any of these patterns, or when the automatic post-summarizer hook fires (see "Automatic Trigger" above):

- "Update the ADO ticket / work item with a summary of what we did"
- "Post this summary as a comment on work item #1234"
- "Add this to the description of the ticket"
- "Link this PR to the work item"
- "Let the team know on the ticket that this is done"
- Any request that combines a completed piece of work (a spec, a session, a code change) with an Azure DevOps ticket reference (a number, a title, or "the ticket for X")

Do not load this file for general Azure DevOps queries that don't involve writing a summary (for example, "what's my assigned work today" or "show me open PRs").

## Step-by-Step Workflow

### Step 1: Identify the work item

Never assume a work item ID. Resolve it explicitly:

- If the user gave an ID directly (e.g. "update #4821" or "work item 4821"), use it as-is.
- If the user gave a title or description instead of an ID, search for it:
  ```
  search_workitem({ searchText: "<user's phrase>" })
  ```
  or, for a more structured lookup:
  ```
  wit_query_by_wiql({ wiql: "SELECT [System.Id], [System.Title] FROM WorkItems WHERE [System.Title] CONTAINS '<phrase>'" })
  ```
- If multiple candidates match, list them (ID + title) and ask the user to confirm which one before proceeding.
- If nothing matches, tell the user and ask for the correct ID rather than guessing.

### Step 2: Get the current work item state

Always fetch the current item before writing to it:

```
wit_work_item({ action: "get", id: <id> })
```

Note from the response:

- `System.Title` and `System.WorkItemType` (to tailor tone/format)
- `System.State` (so you don't accidentally contradict it)
- `System.Description` (needed if you'll be updating the description rather than commenting)
- Existing tags or linked PRs, to avoid duplicating a link that already exists

### Step 3: Generate the summary

Build the summary from the actual session or spec context, not from assumptions:

- What was the goal or requirement being addressed
- What was implemented or changed (files, components, endpoints, behavior)
- What was verified (tests run, build status, manual checks) and what was not verified
- Any follow-up work, known limitations, or blocked items

Keep the summary scoped to what's relevant to this specific work item. Omit unrelated changes from the same session.

### Step 4: Format for Azure DevOps

Azure DevOps rich-text fields (description, comments) render a constrained markdown subset. Follow these rules:

- Use `##` or `###` for section headers, not `#` (reserve `#` for page-level titles, which don't apply inside a ticket field)
- Use `-` for bullet lists; avoid deeply nested lists (max two levels)
- Use `**bold**` for emphasis; avoid italics-heavy formatting, which renders inconsistently
- Use fenced code blocks (```) for file paths, commands, or code snippets
- Use standard markdown links `[text](url)` for linking PRs, commits, or docs
- Avoid raw HTML tags
- Keep tables simple (a few columns, short cell content) since wide tables render poorly in the ADO panel
- Suggested structure for a summary:

  ```
  ## Summary

  <one or two sentence overview>

  ## Changes

  - <change 1>
  - <change 2>

  ## Verification

  - <what was tested/built/run>

  ## Notes

  - <follow-ups, limitations, or links>
  ```

### Step 5: Deliver the summary

Choose between updating the description and adding a comment (see decision tree below), then call the appropriate tool.

**Update the description:**

```
wit_work_item_write({
  action: "update",
  id: <id>,
  fields: {
    "System.Description": "<formatted summary, merged with existing description if relevant>"
  }
})
```

**Add a comment:**

```
wit_work_item_comment_write({
  action: "add",
  workItemId: <id>,
  text: "<formatted summary>"
})
```

**Link a related PR (if one exists for this change):**

```
wit_work_item_link_write({
  action: "link_to_pull_request",
  workItemId: <id>,
  pullRequestId: <pr-id>,
  repositoryId: "<repo-id-or-name>"
})
```

### Step 6: Confirm delivery to the user

After the write succeeds, tell the user:

- Which work item was updated (ID and title)
- Whether it was a description update, a comment, or both
- Whether a PR link was added
- A direct link to the work item if the tool response includes one, otherwise construct it as `https://dev.azure.com/{organization}/{project}/_workitems/edit/{id}`

If the write fails, report the exact error returned by the tool. Do not retry silently more than once; surface the failure and ask the user how to proceed.

## Decision Tree: Description Update vs. Comment

```
Is the user asking to replace/refresh the ticket's description itself?
├── Yes → Update description (wit_work_item_write, action: "update")
│         Preserve any unrelated existing content unless told to replace it entirely.
│
└── No, they want to report progress/completion/status
    │
    Does the ticket already have a substantive description?
    ├── Yes → Add a comment (wit_work_item_comment_write, action: "add")
    │         This is the default for "let the team know" / "post an update" requests.
    │
    └── No, description is empty or just a placeholder
        └── Ask the user: "Should I fill in the description, or add this as a comment?"
            Default to a comment if the user doesn't have a preference, since comments
            are non-destructive and preserve full history.
```

As a rule: comments are the safer default. Only touch the description when the user explicitly wants the ticket's canonical description updated, or when there's no meaningful description to preserve.

## Error Handling

- **Work item not found**: Confirm the ID with the user; don't guess a nearby ID.
- **Permission denied on write**: Report that the authenticated Entra ID account lacks edit permission on this work item or area path; suggest the user check their project permissions or ask a project admin.
- **Field validation error** (e.g. invalid state transition): Call `wit_work_item({ action: "get_type" })` for the work item type to check valid field values and allowed states, then retry with corrected values. Explain the correction to the user.
- **PR link fails because the PR or repo isn't found**: Verify the `repositoryId` and `pullRequestId` with `repo_pull_request({ action: "get", pullRequestId: <id> })` before retrying.
- **Ambiguous or multiple matching work items during search**: Always ask the user to disambiguate; never write to a guessed item.
- **Batch update partial failure**: Report which IDs succeeded and which failed, with the specific error for each failed ID, rather than a single generic failure message.

## Example Tool Calls

Get a work item before editing it:

```json
{
  "tool": "wit_work_item",
  "input": { "action": "get", "id": 4821 }
}
```

Add a comment with a formatted summary:

```json
{
  "tool": "wit_work_item_comment_write",
  "input": {
    "action": "add",
    "workItemId": 4821,
    "text": "## Summary\n\nImplemented pagination for the /users endpoint.\n\n## Changes\n\n- Added limit/offset params to the query layer\n- Updated route handler to accept page/pageSize\n- Added pagination metadata to the response\n\n## Verification\n\n- Unit tests added for query layer, all passing\n- Manual check of /users?page=2&pageSize=20\n\n## Notes\n\n- Follow-up: add cursor-based pagination for large datasets"
  }
}
```

Update a work item's description:

```json
{
  "tool": "wit_work_item_write",
  "input": {
    "action": "update",
    "id": 4821,
    "fields": {
      "System.Description": "## Summary\n\n<updated description merged with prior content>"
    }
  }
}
```

Link a pull request to a work item:

```json
{
  "tool": "wit_work_item_link_write",
  "input": {
    "action": "link_to_pull_request",
    "workItemId": 4821,
    "pullRequestId": 512,
    "repositoryId": "my-service-repo"
  }
}
```
