# Kiro ADO Power

A [Kiro](https://kiro.dev) power that connects your development sessions to Azure DevOps. Pull user story context into your session, then automatically push implementation summaries back as child tasks when done.

## What it does

| Capability | Description |
|-----------|-------------|
| **Read context** | Fetch a user story's description and acceptance criteria to ground your coding session |
| **Write results** | Create a child task under that user story with a structured implementation summary |
| **Auto-delivery** | A hook fires after the summarizer agent runs, linking results back to the ticket automatically |

## Quick Start

### Prerequisites

- [Kiro IDE](https://kiro.dev/downloads/) or [Kiro CLI](https://kiro.dev/cli/)
- [Node.js 16+](https://nodejs.org/) (`node --version` to check)
- An Azure DevOps organization with work items

### 1. Generate a Personal Access Token

1. Go to `https://dev.azure.com/{your-org}/_usersSettings/tokens`
2. Click **+ New Token**
3. Set:
   - **Name**: `Kiro`
   - **Organization**: your org
   - **Scopes**: Custom defined → ✅ **Work Items: Read & Write**
   - **Expiration**: 90 days
4. Click **Create** and **copy the token** (it's shown only once)

### 2. Add the token to your shell

```bash
echo 'export AZURE_DEVOPS_PAT="paste-your-token-here"' >> ~/.zshrc
source ~/.zshrc
```

### 3. Configure the power

After installing, update `mcp.json` with your org details:

```json
{
  "AZURE_DEVOPS_ORG_URL": "https://dev.azure.com/your-org-name",
  "AZURE_DEVOPS_DEFAULT_PROJECT": "Your Project Name"
}
```

### 4. Install the power

In Kiro IDE:
1. Open the **Powers panel** (⚡ icon in sidebar)
2. Click **Add Custom Power** → **Import power from GitHub**
3. Enter: `https://github.com/erodriiguezz/kiro-ado-power`
4. Click **Install**

### 5. Launch and test

```bash
kiro .
```

Then ask:
> "Pull the context for user story #12345"

If you see the work item details, you're set up. ✅

## Usage

### Pull context (start of session)

```
"Pull the context for user story #1234"
"What are the acceptance criteria for #5678?"
```

The agent fetches the work item's title, description, acceptance criteria, state, and tags — grounding the session with what needs to be built.

### Push results (end of session)

```
"Push a task summary to user story #1234"
```

Or let it happen automatically — when the summarizer agent completes, a hook:
1. Reads the work item ID from your git branch (e.g. `feature/1234-auth-flow`)
2. Creates a child task with the implementation summary
3. Links it to the parent user story

### Branch naming convention

For automatic delivery, include the work item ID in your branch name:

```
feature/1234-auth-flow
1234-payment-integration
bugfix/5678-null-check
```

## How it works

```
┌─────────────────────────────────────────────────┐
│  Developer's Kiro Session                       │
│                                                 │
│  START: "Pull context for #1234"                │
│    └─→ get_work_item → description + criteria   │
│                                                 │
│  WORK: code, test, build                        │
│                                                 │
│  END: summarizer runs → hook fires              │
│    └─→ create_work_item → child Task            │
│    └─→ manage_work_item_link → parent link      │
└─────────────────────────────────────────────────┘
```

**MCP Server**: [@tiberriver256/mcp-server-azure-devops](https://github.com/Tiberriver256/mcp-server-azure-devops) (community, 46 tools, PAT auth works correctly)

## Security

- **PAT is per-developer** — actions attributed to the individual in ADO audit logs
- **PAT is never in this repo** — lives only in the developer's shell environment
- **Minimum scope** — only Work Items Read & Write
- **PAT expires** — rotate at the same tokens URL before expiration

## Token Rotation

When your PAT expires (every 90 days):

```bash
# Generate new token at dev.azure.com/{your-org}/_usersSettings/tokens
# Then update .zshrc:
sed -i '' 's/export AZURE_DEVOPS_PAT=.*/export AZURE_DEVOPS_PAT="NEW_TOKEN_HERE"/' ~/.zshrc
source ~/.zshrc
```

Restart Kiro IDE for the change to take effect.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "MCP server not connected" | Token not in environment — verify `echo $AZURE_DEVOPS_PAT` shows a value |
| "Not authorized" (401/403) | PAT expired or scope too narrow — regenerate with Work Items Read & Write |
| Env var set but still fails | Launch Kiro from terminal (`kiro .`) so it inherits your shell env |
| `npx` not found | Install Node.js: `brew install node` |
| Tools don't appear | Restart Kiro IDE after installing the power |

## Known Issues

### "Check for updates" not detecting new versions (Kiro IDE)

The Kiro IDE's **Check for updates** button in the Powers panel may not detect new commits for custom GitHub-sourced powers. This appears to be a Kiro IDE bug — the button doesn't always run `git fetch` against the remote.

**Workaround — manual pull:**

```bash
cd ~/.kiro/powers/repos/kiro-ado-power && git pull origin main
```

Then restart your Kiro IDE session for the updated steering and hooks to take effect.

**Alternative — uninstall and reinstall:**

1. Powers panel → kiro-ado-power → Uninstall
2. Add Custom Power → Import from GitHub → `https://github.com/erodriiguezz/kiro-ado-power`

This forces a fresh clone with the latest version.

## Project Structure

```
kiro-ado-power/
├── README.md                             # This file
├── POWER.md                              # Power manifest (activation keywords, metadata)
├── mcp.json                              # MCP server configuration
├── .kiro/
│   └── hooks/
│       └── ado-auto-update.json          # Auto-creates child task after deliver agent
└── steering/
    ├── task-summary-delivery.md          # Formatting rules and delivery workflow
    └── time-tracking.md                  # Time tracking guardrails (never log hours)
```

## Contributing

1. Fork the repo
2. Create a feature branch (`git checkout -b my-change`)
3. Make changes
4. Test locally: Powers panel → Add Custom Power → Import from folder
5. Commit with a [semantic message](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`)
6. Push to your fork and open a Pull Request
