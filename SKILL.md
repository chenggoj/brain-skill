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

**Vault path:** `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Obsidian/Projects/`

This skill connects Claude Code to your Obsidian vault for persistent project memory.
Each project has a `CONTEXT.md` (architecture + status), `DECISIONS.md` (design log), and
dated `HANDOFF/` notes written at the end of each session.

---

## Sub-commands

Parse `$ARGUMENTS` to determine the sub-command:
- `resume [project]` — load context and brief yourself
- `wrap-up [project]` — save session progress
- `init [project]` — scaffold a new project folder
- no args or unknown — show project list

---

## `resume [project]`

**Goal:** Reconstruct full project context in under 60 seconds so the user doesn't have to re-explain anything.

### Step 1 — Check Obsidian is reachable

```bash
OBSIDIAN_API_KEY=$(python3 -c "import json; d=json.load(open('/Users/chenggongsopenclaw/.claude.json')); print(d['mcpServers']['obsidian']['env']['OBSIDIAN_API_KEY'])" 2>/dev/null)
curl -sk -H "Authorization: Bearer $OBSIDIAN_API_KEY" https://127.0.0.1:27124/vault/ > /dev/null 2>&1 && echo "OK" || echo "OFFLINE"
```

If OFFLINE: fall back to reading files directly from disk. Do NOT abort.

### Step 2 — Read INDEX

If Obsidian is online, use `mcp__obsidian__obsidian_get_file_contents` to read `Projects/_INDEX.md`.
If offline, read directly from the vault path.

If no project name was given, display the index table and ask which project to load.

### Step 3 — Read project files

Read these three files (prefer Obsidian MCP when online, fall back to direct Read):

1. `Projects/<project>/CONTEXT.md` — architecture, current stage, open questions
2. `Projects/<project>/DECISIONS.md` — design decision log
3. Latest file in `Projects/<project>/HANDOFF/` — most recent session summary

To find the latest HANDOFF file:
```bash
VAULT="$HOME/Library/Mobile Documents/iCloud~md~obsidian/Documents/Obsidian"
ls "$VAULT/Projects/<project>/HANDOFF/" 2>/dev/null | sort | tail -1
```

### Step 4 — Deliver briefing

Output a structured briefing in this format:

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

### Step 1 — Synthesize session summary

Review the current conversation and extract:
- **What was done** — concrete actions taken (code written, bugs fixed, stages completed)
- **Problems encountered** — blockers, errors, unexpected findings
- **Where to continue next** — the exact next step, as specific as possible
- **Key insights** — any design insight worth preserving

### Step 2 — Write HANDOFF note

Create `Projects/<project>/HANDOFF/YYYY-MM-DD.md` (use today's date).

If Obsidian is online, write via REST API:
```bash
python3 - <<'PYEOF'
import os, requests, urllib3, json
from datetime import date
urllib3.disable_warnings()
API_KEY = json.load(open('/Users/chenggongsopenclaw/.claude.json'))['mcpServers']['obsidian']['env']['OBSIDIAN_API_KEY']
BASE_URL = "https://127.0.0.1:27124"

path = f"Projects/<project>/HANDOFF/{date.today().isoformat()}.md"
content = """<HANDOFF_CONTENT>"""

r = requests.put(f"{BASE_URL}/vault/{path}",
    headers={"Authorization": f"Bearer {API_KEY}", "Content-Type": "text/markdown"},
    data=content.encode("utf-8"), verify=False)
print(f"{'OK' if r.status_code in [200,204] else 'ERROR'} [{r.status_code}] {path}")
PYEOF
```

If Obsidian is offline: write directly to the vault path using the Write tool.

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

### Step 3 — Update CONTEXT.md

Update the "Current stage" field and "Open questions" list in `Projects/<project>/CONTEXT.md`
to reflect the session outcome. Read the file first, then use the Edit tool.

### Step 4 — Update _INDEX.md

Update the row for this project in `Projects/_INDEX.md`:
- Status emoji (🟡/🟢/🔴/🔵)
- Current stage
- Last session date

### Step 5 — Confirm

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

Ask the user 3 questions before writing:
1. What is the project goal? (one sentence)
2. Where is the project on disk? (local path)
3. What stage or phase is it currently in?

Then create:
- `Projects/<project>/CONTEXT.md` — pre-filled with the answers above
- `Projects/<project>/DECISIONS.md` — empty log template
- `Projects/<project>/HANDOFF/.gitkeep` — placeholder to create the folder

Update `Projects/_INDEX.md` to add the new project row.

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

If called with no arguments or an unrecognized argument:

Read `Projects/_INDEX.md` and display the project table. Then show:
```
Usage:
  /brain resume <project>    load context and start a session
  /brain wrap-up <project>   save progress and end a session
  /brain init <project>      initialize a new project
```

---

## Offline Fallback

If the Obsidian MCP / REST API is unreachable:
1. **Never abort** — fall back to direct file read/write using the vault path
2. Vault path: `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Obsidian/`
3. All operations work identically, just without the MCP layer
4. Inform the user: "Obsidian offline — reading/writing directly to vault files"

---

## Error Handling

| Problem | Fix |
|---------|-----|
| Obsidian connection refused | Fall back to direct disk I/O |
| Project folder not found | Suggest `/brain init <project>` |
| HANDOFF folder is empty | Skip HANDOFF step in resume; note "no previous sessions found" |
| `_INDEX.md` missing | Create it from scratch using the current project list |
