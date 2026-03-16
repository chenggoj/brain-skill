# brain — Obsidian Long-Term Memory for Claude Code

> Never re-explain your project background again.

`brain` is a Claude Code skill that connects your AI coding sessions to an iCloud-synced Obsidian vault, giving you persistent, structured project memory across conversations.

## The Problem

Every time you start a new Claude Code session, you're talking to a blank slate. You re-explain the architecture, the current stage, what broke last time, and what to work on next — every single time.

## The Solution

`brain` maintains a structured knowledge base for each project in Obsidian:
- **`CONTEXT.md`** — architecture, tech stack, current stage, open questions
- **`DECISIONS.md`** — design decision log with rationale
- **`HANDOFF/YYYY-MM-DD.md`** — dated session notes written at conversation end

Two commands replace the entire onboarding ritual:

```
/brain resume <project>   # load context at start of session
/brain wrap-up <project>  # save progress at end of session
```

---

## Installation

### Prerequisites

- [Claude Code](https://claude.ai/code) CLI
- [Obsidian](https://obsidian.md) with the [Local REST API](https://github.com/coddingtonbear/obsidian-local-rest-api) plugin enabled
- Obsidian vault accessible on disk (local or iCloud)

### Install the Skill

```bash
mkdir -p ~/.claude/skills/brain
curl -o ~/.claude/skills/brain/SKILL.md \
  https://raw.githubusercontent.com/chenggoj/brain-skill/main/SKILL.md
```

Claude Code auto-discovers skills in `~/.claude/skills/` — no registration needed.

### Configure Obsidian API Key

The skill reads your Obsidian API key from `~/.claude.json`:

```json
{
  "mcpServers": {
    "obsidian": {
      "env": {
        "OBSIDIAN_API_KEY": "your-key-here"
      }
    }
  }
}
```

Get your key from Obsidian → Settings → Local REST API.

### Set Your Vault Path

Edit `SKILL.md` and update the vault path at the top to match your setup:

```
Vault path: ~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Obsidian/Projects/
```

Common paths:
| Setup | Path |
|-------|------|
| iCloud sync (macOS) | `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/<VaultName>/` |
| Local vault | `~/Documents/Obsidian/<VaultName>/` |

---

## Usage

### Start a session

```
/brain resume natcomms_pipeline
```

Claude reads your `CONTEXT.md`, `DECISIONS.md`, and the most recent `HANDOFF` note, then delivers a structured briefing:

```
## 📂 natcomms_pipeline — Session Briefing

项目目标：从 Nature Communications 构建 ~30k SFT 数据集
当前阶段：Stage 1-2 并行（HTML scraping + OA API XML）
上次会话（2026-03-16）：
  · 修复了 XML parser 对 JATS 格式的边缘情况处理
  · OA API 配额用了 312/480，明天继续跑 Stage 2
待解决问题：
  · Stage 4 SLURM job 提交脚本尚未端到端测试

---
这次想继续什么？
```

### End a session

```
/brain wrap-up natcomms_pipeline
```

Claude synthesizes the session and writes:
- A dated `HANDOFF/YYYY-MM-DD.md` note
- Updates `CONTEXT.md` with current stage and open questions
- Updates `_INDEX.md` with the latest status

### Initialize a new project

```
/brain init my-new-project
```

Claude asks 3 questions (goal, local path, current stage) then scaffolds the full folder structure.

### List all projects

```
/brain
```

Shows the `_INDEX.md` project table.

---

## Vault Structure

```
Obsidian Vault/
└── Projects/
    ├── _INDEX.md                    ← master project index
    ├── my-project/
    │   ├── CONTEXT.md               ← architecture, status, open questions
    │   ├── DECISIONS.md             ← design decision log (newest first)
    │   └── HANDOFF/
    │       ├── 2026-03-15.md
    │       └── 2026-03-16.md
    └── another-project/
        └── ...
```

### `_INDEX.md` format

```markdown
# Projects Index

| 项目 | 状态 | 当前阶段 | 上次会话 |
|------|------|---------|---------|
| my-project | 🟡 进行中 | Stage 2 | 2026-03-16 |
| old-project | 🔵 归档 | Done | 2026-01-10 |
```

Status emoji convention:
- 🟢 Running/deployed
- 🟡 In progress
- 🔴 Blocked
- 🔵 Archived

---

## Vault Templates

Ready-to-use template files are in the [`vault-templates/`](vault-templates/) directory.
Copy them to your vault's `Projects/<your-project>/` folder to get started manually.

---

## Offline Fallback

If Obsidian is closed or the REST API is unreachable, `brain` automatically falls back to
direct disk reads/writes using the vault path. All operations work identically — you'll
just see a note: `"Obsidian offline — reading/writing directly to vault files"`.

---

## Design Philosophy

This skill implements the **Department Vault Brain** pattern from the Claude Code community
(inspired by [Daniel Miessler's PAI system](https://github.com/danielmiessler/PAI)), adapted
for a solo developer workflow:

- **Token-efficient**: Only loads files relevant to the active project, not the entire vault
- **iCloud-native**: Vault syncs automatically to iPhone/iPad for on-the-go project review
- **Complementary to `MEMORY.md`**: `MEMORY.md` stores user preferences and identity;
  Obsidian stores project-specific technical state. They don't overlap.
- **Plain text, always**: Every file is Markdown — readable, searchable, version-controllable

---

## License

MIT
