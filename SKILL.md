---
name: brain
description: |
  Obsidian-backed long-term project memory for Claude Code programming sessions.
  Loads/saves project context to an Obsidian vault so you never have to re-explain
  project background at the start of a new conversation.

  Use this skill when:
  - Starting a new session on an existing project → /brain resume <project>
  - Ending a session and want to save progress → /brain wrap-up <project>
  - Starting a brand-new project → /brain init <project>
  - Listing all projects → /brain (no args)

user-invocable: true
allowed-tools: Bash, Read, Write, Edit, mcp__obsidian__obsidian_get_file_contents, mcp__obsidian__obsidian_list_files_in_dir, mcp__obsidian__obsidian_patch_content, mcp__obsidian__obsidian_get_recent_changes
argument-hint: [resume <project> | wrap-up <project> | init <project>]
---

# Brain — Obsidian Project Memory

This skill connects Claude Code to your Obsidian vault for persistent project memory.
Each project has a `CONTEXT.md` (architecture + status), `DECISIONS.md` (design log), and
dated `HANDOFF/` notes written at the end of each session.

---

## Step 0 — Discover Vault Path (run this first, every time)

**Never hardcode a vault path.** Always resolve it dynamically from Obsidian's own config:

```bash
python3 - <<'PYEOF'
import json, os, sys

config_path = os.path.expanduser(
    "~/Library/Application Support/obsidian/obsidian.json"
)
try:
    config = json.load(open(config_path))
except FileNotFoundError:
    print("ERROR: Obsidian config not found. Is Obsidian installed?", file=sys.stderr)
    sys.exit(1)

vaults = list(config.get("vaults", {}).values())
if not vaults:
    print("ERROR: No vaults found in Obsidian config.", file=sys.stderr)
    sys.exit(1)

# Prefer the vault that was most recently open
active = next((v for v in vaults if v.get("open")), None)

# Fall back: vault whose path contains "Projects" subfolder
if not active:
    for v in vaults:
        projects_dir = os.path.join(v["path"], "Projects")
        if os.path.isdir(projects_dir):
            active = v
            break

# Last resort: first vault
if not active:
    active = vaults[0]

vault_path = active["path"]
projects_path = os.path.join(vault_path, "Projects")
print(vault_path)
print(projects_path)
PYEOF
```

Capture the output:
- **Line 1** → `VAULT` (e.g. `/Users/you/Library/Mobile Documents/iCloud~md~obsidian/Documents/MyVault`)
- **Line 2** → `PROJECTS` (e.g. `$VAULT/Projects`)

Use `PROJECTS` as the base for all subsequent file operations in this session.

If multiple vaults are found and none is clearly active, list them and ask the user which one to use.

---

## Sub-commands

Parse `$ARGUMENTS` to determine the sub-command:
- `resume [project]` — load context and brief yourself
- `wrap-up [project]` — save session progress
- `init [project]` — scaffold a new project folder
- no args or unknown — show project list

---

## `resume [project]`

**Goal:** Reconstruct full project context in under 60 seconds.

### Step 1 — Discover vault (Step 0 above)

### Step 2 — Check Obsidian REST API

```bash
OBSIDIAN_API_KEY=$(python3 -c "
import json, os
claude_cfg = os.path.expanduser('~/.claude.json')
try:
    d = json.load(open(claude_cfg))
    print(d['mcpServers']['obsidian']['env']['OBSIDIAN_API_KEY'])
except Exception:
    pass
" 2>/dev/null)
curl -sk -H "Authorization: Bearer $OBSIDIAN_API_KEY" https://127.0.0.1:27124/vault/ \
  > /dev/null 2>&1 && echo "ONLINE" || echo "OFFLINE"
```

If OFFLINE: fall back to reading files directly from `$PROJECTS`. Do NOT abort.

### Step 3 — Read INDEX

If online, use `mcp__obsidian__obsidian_get_file_contents` to read `Projects/_INDEX.md`.
If offline, read `$PROJECTS/_INDEX.md` directly.

If no project name was given, display the index table and ask which project to load.

### Step 4 — Read project files

Read these three files (Obsidian MCP preferred when online, direct Read when offline):

1. `$PROJECTS/<project>/CONTEXT.md`
2. `$PROJECTS/<project>/DECISIONS.md`
3. Latest file in `$PROJECTS/<project>/HANDOFF/`:

```bash
ls "$PROJECTS/<project>/HANDOFF/" 2>/dev/null | sort | tail -1
```

### Step 5 — Deliver briefing

```
## <project> — Session Briefing

Goal: <one line from CONTEXT>
Current stage: <current stage/status>
Last session (<date>): <2-3 bullets from latest HANDOFF>
Open questions:
- <open question 1>
- <open question 2>

---
What would you like to work on today?
```

Keep it concise — the user already knows the project, this is a reminder, not a tutorial.

---

## `wrap-up [project]`

**Goal:** Capture session progress before the conversation ends.

### Step 1 — Discover vault (Step 0 above)

### Step 2 — Synthesize session summary

Review the current conversation and extract:
- **What was done** — concrete actions taken (code written, bugs fixed, stages completed)
- **Problems encountered** — blockers, errors, unexpected findings
- **Where to continue next** — the exact next step, as specific as possible
- **Key insights** — any design insight worth preserving

### Step 3 — Write HANDOFF note

Create `$PROJECTS/<project>/HANDOFF/YYYY-MM-DD.md` (use today's date).

If Obsidian is online, write via REST API:
```bash
python3 - <<'PYEOF'
import json, os, requests, urllib3
from datetime import date
urllib3.disable_warnings()

claude_cfg = os.path.expanduser("~/.claude.json")
API_KEY = json.load(open(claude_cfg))["mcpServers"]["obsidian"]["env"]["OBSIDIAN_API_KEY"]
BASE_URL = "https://127.0.0.1:27124"

path = f"Projects/<project>/HANDOFF/{date.today().isoformat()}.md"
content = """<HANDOFF_CONTENT>"""

r = requests.put(
    f"{BASE_URL}/vault/{path}",
    headers={"Authorization": f"Bearer {API_KEY}", "Content-Type": "text/markdown"},
    data=content.encode("utf-8"),
    verify=False,
)
print(f"{'OK' if r.status_code in [200, 204] else 'ERROR'} [{r.status_code}] {path}")
PYEOF
```

If Obsidian is offline: write directly to `$PROJECTS/<project>/HANDOFF/YYYY-MM-DD.md` using the Write tool.

**HANDOFF note template:**
```markdown
# <project> — Session Handoff <YYYY-MM-DD>

## What was done
- <specific action 1>
- <specific action 2>

## Problems encountered
- <blocker or finding>

## Where to continue next
<One concrete next step — be specific>

## Key insights / design decisions
- <insight worth remembering>
```

### Step 4 — Update CONTEXT.md

Update "Current stage" and "Open questions" in `$PROJECTS/<project>/CONTEXT.md`.
Read the file first, then use the Edit tool.

### Step 5 — Update _INDEX.md

Update the row for this project in `$PROJECTS/_INDEX.md`:
- Status emoji (🟡/🟢/🔴/🔵)
- Current stage
- Last session date

### Step 6 — Confirm

```
Wrap-up complete for <project>:
  HANDOFF  → Projects/<project>/HANDOFF/<date>.md
  CONTEXT  → updated
  INDEX    → updated

Start your next session with: /brain resume <project>
```

---

## `init <project>`

**Goal:** Scaffold the Obsidian folder structure for a brand-new project.

### Step 1 — Discover vault (Step 0 above)

Ask the user 3 questions before writing:
1. What is the project goal? (one sentence)
2. Where is the project on disk? (local path)
3. What stage or phase is it currently in?

Then create:
- `$PROJECTS/<project>/CONTEXT.md` — pre-filled with the answers above
- `$PROJECTS/<project>/DECISIONS.md` — empty log template
- `$PROJECTS/<project>/HANDOFF/` — create folder with a placeholder file

Update `$PROJECTS/_INDEX.md` to add the new project row.

Confirm:
```
Initialized brain for <project>:
  Projects/<project>/CONTEXT.md
  Projects/<project>/DECISIONS.md
  Projects/<project>/HANDOFF/

When done working, run: /brain wrap-up <project>
```

---

## No-args / List Projects

### Step 1 — Discover vault (Step 0 above)

Read `$PROJECTS/_INDEX.md` and display the project table. Then show:
```
Usage:
  /brain resume <project>    load context and start a session
  /brain wrap-up <project>   save progress and end a session
  /brain init <project>      initialize a new project
```

---

## Offline Fallback

If the Obsidian MCP / REST API is unreachable:
1. **Never abort** — fall back to direct file read/write using `$PROJECTS`
2. All operations work identically, just without the MCP layer
3. Inform the user: "Obsidian offline — reading/writing directly to vault files"

---

## Error Handling

| Problem | Fix |
|---------|-----|
| `obsidian.json` not found | Obsidian not installed — ask user to install it |
| No vaults in config | Ask user to open Obsidian and create a vault first |
| Multiple vaults, none clearly active | List them, ask user to pick one |
| Obsidian REST API connection refused | Fall back to direct disk I/O |
| Project folder not found | Suggest `/brain init <project>` |
| HANDOFF folder is empty | Skip HANDOFF step in resume; note "no previous sessions found" |
| `_INDEX.md` missing | Create it from scratch using the current project list |
