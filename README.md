# brain — Obsidian Long-Term Memory for Claude Code

> Never re-explain your project background again.

`brain` is a Claude Code skill that connects your AI coding sessions to an Obsidian vault, giving you persistent, structured project memory across conversations.

## The Problem

Every time you start a new Claude Code session, you're talking to a blank slate. You re-explain the architecture, the current stage, what broke last time, and what to work on next — every single time.

## The Solution

`brain` maintains a structured knowledge base for each project in Obsidian:
- **`CONTEXT.md`** — architecture, tech stack, current stage, open questions
- **`DECISIONS.md`** — design decision log with rationale
- **`HANDOFF/YYYY-MM-DD.md`** — dated session notes written at conversation end

Two commands replace the entire onboarding ritual:

```
/brain resume <project>    # load context at the start of a session
/brain wrap-up <project>   # save progress at the end of a session
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

Get your key from **Obsidian → Settings → Local REST API**.

### No vault path configuration needed

`brain` automatically discovers your active vault by reading Obsidian's own config file:

```
~/Library/Application Support/obsidian/obsidian.json
```

This is the same file Obsidian uses to track all your vaults. The skill picks the vault
that was most recently open — no hardcoded paths, no manual configuration.

If you have multiple vaults, the skill will ask you to pick one the first time.

---

## Usage

### Start a session

```
/brain resume my-project
```

Claude reads your `CONTEXT.md`, `DECISIONS.md`, and the most recent `HANDOFF` note, then delivers a structured briefing:

```
## my-project — Session Briefing

Goal: Build a data pipeline that collects 30k training samples from open-access papers
Current stage: Stage 2 — XML full-text retrieval via OA API
Last session (2026-03-15):
  · Fixed XML parser edge cases for malformed JATS tags
  · API quota used 312/480 for the day — Stage 2 still running
Open questions:
  · SLURM job submission script not yet tested end-to-end

---
What would you like to work on today?
```

### End a session

```
/brain wrap-up my-project
```

Claude synthesizes the session and:
- Writes a dated `HANDOFF/YYYY-MM-DD.md` note
- Updates `CONTEXT.md` with the current stage and open questions
- Updates `_INDEX.md` with the latest project status

### Initialize a new project

```
/brain init my-new-project
```

Claude asks 3 questions (goal, local path, current stage) then scaffolds the full folder structure.

### List all projects

```
/brain
```

Displays the `_INDEX.md` project table.

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

| Project | Status | Current Stage | Last Session |
|---------|--------|---------------|-------------|
| my-project | 🟡 In Progress | Stage 2 | 2026-03-16 |
| old-project | 🔵 Archived | Done | 2026-01-10 |
```

Status emoji convention:

| Emoji | Meaning |
|-------|---------|
| 🟢 | Running / Deployed |
| 🟡 | In Progress |
| 🔴 | Blocked |
| 🔵 | Archived |

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

This skill implements the **Department Vault Brain** pattern from the Claude Code community,
inspired by [Daniel Miessler's PAI system](https://github.com/danielmiessler/PAI), adapted
for a solo developer workflow:

- **Zero configuration** — discovers your active vault from `~/Library/Application Support/obsidian/obsidian.json`, the same source of truth Obsidian itself uses (inspired by [@steipete's Obsidian skill](https://clawhub.openclaw.com))
- **Token-efficient** — only loads files relevant to the active project, not the entire vault
- **iCloud-native** — vault syncs automatically to iPhone/iPad for on-the-go project review
- **Plain text, always** — every file is Markdown: readable, searchable, version-controllable
- **Complementary to `MEMORY.md`** — `MEMORY.md` stores user preferences; Obsidian stores project-specific technical state

---

## License

MIT
