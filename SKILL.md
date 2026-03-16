---
name: brain
description: |
  Obsidian-backed long-term project memory for Claude Code programming sessions.
  Loads/saves project context to iCloud Obsidian vault so you never have to re-explain
  project background at the start of a new conversation.

  Use this skill when:
  - Starting a new session on an existing project → /brain resume <project>
  - Ending a session and want to save progress → /brain wrap-up <project>
  - Starting a brand-new project → /brain init <project>
  - Listing all projects → /brain resume (no args)

user-invocable: true
allowed-tools: Bash, Read, Write, Edit, mcp__obsidian__obsidian_get_file_contents, mcp__obsidian__obsidian_list_files_in_dir, mcp__obsidian__obsidian_patch_content, mcp__obsidian__obsidian_get_recent_changes
argument-hint: [resume <project> | wrap-up <project> | init <project>]
---

# Brain — Obsidian Project Memory

**Vault path:** `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Obsidian/Projects/`

This skill connects Claude Code to your iCloud Obsidian vault for persistent project memory.
Each project has a `CONTEXT.md` (architecture + status), `DECISIONS.md` (design log), and
dated `HANDOFF/` notes written at end of each session.

---

## Sub-commands

Parse `$ARGUMENTS` to determine the sub-command:
- `resume [project]` — load context and brief yourself
- `wrap-up [project]` — save session progress
- `init [project]` — scaffold new project folder
- no args or unknown → show project list

---

## `resume [project]`

**Goal:** In under 60 seconds, reconstruct full project context so the user doesn't have to re-explain anything.

### Step 1 — Check Obsidian is reachable

```bash
OBSIDIAN_API_KEY=$(python3 -c "import json; d=json.load(open('/Users/chenggongsopenclaw/.claude.json')); print(d['mcpServers']['obsidian']['env']['OBSIDIAN_API_KEY'])" 2>/dev/null)
curl -sk -H "Authorization: Bearer $OBSIDIAN_API_KEY" https://127.0.0.1:27124/vault/ > /dev/null 2>&1 && echo "OK" || echo "OFFLINE"
```

If OFFLINE: fall back to reading files directly from disk (vault path is known). Do NOT abort.

### Step 2 — Read INDEX

If Obsidian is online, use `mcp__obsidian__obsidian_get_file_contents` to read `Projects/_INDEX.md`.
If offline, read directly: `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Obsidian/Projects/_INDEX.md`

If no project name given, display the index table and ask which project to load.

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
## 📂 <project> — Session Briefing

**项目目标：** <one line from CONTEXT>
**当前阶段：** <current stage/status>
**上次会话（<date>）：** <summary from latest HANDOFF, 2-3 bullets>
**待解决问题：**
- <open question 1>
- <open question 2>

---
这次想继续什么？
```

Keep it concise — the user already knows the project, this is a reminder, not a tutorial.

---

## `wrap-up [project]`

**Goal:** Capture session progress before the conversation ends. Write a HANDOFF note and update CONTEXT.

### Step 1 — Synthesize session summary

Review the current conversation and extract:
- **今天做了什么** — concrete actions taken (code written, bugs fixed, stages completed)
- **遇到的问题** — blockers, errors, unexpected findings
- **下次从哪里继续** — the exact next step, as specific as possible
- **重要发现** — any design insight worth remembering

### Step 2 — Write HANDOFF note

Create file `Projects/<project>/HANDOFF/YYYY-MM-DD.md` (use today's date: 2026-03-16 or current).

If Obsidian is online:
```bash
OBSIDIAN_API_KEY=$(python3 -c "import json; d=json.load(open('/Users/chenggongsopenclaw/.claude.json')); print(d['mcpServers']['obsidian']['env']['OBSIDIAN_API_KEY'])")
python3 - <<'PYEOF'
import os, requests, urllib3
urllib3.disable_warnings()
from datetime import date
API_KEY = os.environ.get("OBSIDIAN_API_KEY") or __import__('subprocess').check_output(
    ["python3", "-c", "import json; d=json.load(open('/Users/chenggongsopenclaw/.claude.json')); print(d['mcpServers']['obsidian']['env']['OBSIDIAN_API_KEY'])"]
).decode().strip()
BASE_URL = "https://127.0.0.1:27124"

path = f"Projects/<project>/HANDOFF/{date.today().isoformat()}.md"
content = """<HANDOFF_CONTENT>"""

r = requests.put(f"{BASE_URL}/vault/{path}",
    headers={"Authorization": f"Bearer {API_KEY}", "Content-Type": "text/markdown"},
    data=content.encode("utf-8"), verify=False)
print(f"{'✓' if r.status_code in [200,204] else '✗'} [{r.status_code}] {path}")
PYEOF
```

If Obsidian is offline: write directly to the vault path using Write tool.

**HANDOFF note template:**
```markdown
# <project> — Session Handoff <YYYY-MM-DD>

## 今天做了什么
- <specific action 1>
- <specific action 2>

## 遇到的问题
- <blocker or finding>

## 下次从哪里继续
<one concrete next step — be specific, e.g. "从 stage2.py line 142 的 XML 解析器开始">

## 重要发现 / 设计决策
- <insight worth remembering>
```

### Step 3 — Update CONTEXT.md

Update the "当前状态 > 进行中阶段" field and "开放问题" list in `Projects/<project>/CONTEXT.md`
to reflect the session's outcome. Use Edit tool (read first, then edit).

### Step 4 — Update _INDEX.md

Update the row for this project in `Projects/_INDEX.md`:
- 状态 (🟡/🟢/🔴/🔵)
- 当前阶段
- 上次会话 date

### Step 5 — Confirm

```
✅ Brain wrap-up complete for <project>:
   · HANDOFF → Projects/<project>/HANDOFF/<date>.md
   · CONTEXT.md updated
   · _INDEX.md updated

下次开始新对话时运行：/brain resume <project>
```

---

## `init <project>`

**Goal:** Scaffold the Obsidian folder structure for a brand-new project.

Ask the user 3 questions before writing:
1. 项目目标是什么？（一句话）
2. 项目路径在哪里？（本地路径）
3. 目前处于什么阶段？

Then create:
- `Projects/<project>/CONTEXT.md` — pre-filled with answers above
- `Projects/<project>/DECISIONS.md` — empty log template
- `Projects/<project>/HANDOFF/` — empty folder (create a `.gitkeep` placeholder)

Update `Projects/_INDEX.md` to add the new project row.

Confirm:
```
✅ Brain initialized for <project>:
   · Projects/<project>/CONTEXT.md
   · Projects/<project>/DECISIONS.md
   · Projects/<project>/HANDOFF/

开始工作后，会话结束时运行：/brain wrap-up <project>
```

---

## No-args / List Projects

If called with no arguments or unrecognized argument:

Read `Projects/_INDEX.md` and display the project table. Then show:
```
用法：
  /brain resume <项目名>   — 加载上下文开始会话
  /brain wrap-up <项目名>  — 保存进度结束会话
  /brain init <项目名>     — 初始化新项目
```

---

## Offline Fallback

If Obsidian MCP / REST API is unreachable:
1. **Never abort** — fall back to direct file read/write using vault path
2. Vault path: `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Obsidian/`
3. All operations work identically, just without the MCP layer
4. Note to user: "Obsidian offline — reading/writing directly to vault files"

---

## Error Handling

| Problem | Fix |
|---------|-----|
| Obsidian connection refused | Fall back to direct disk I/O |
| Project folder not found | Suggest `/brain init <project>` |
| HANDOFF folder empty | Skip HANDOFF step in resume, note "no previous sessions" |
| `_INDEX.md` missing | Create it from scratch using current project list |
