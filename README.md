# ContextForge

Scaffold a complete AI-assisted coding workflow for Claude Code in seconds.

Run `/setupcode` and get 15 template files — a structured `/doc` folder, session prompts, changelogs, and progress tracking — ready to fill in for any project.

---

## What it creates

```
your-project/
├── CLAUDE.md                 ← Final goal + doc navigation (created by /final-goal)
├── initialprompt.md          ← All AI rules + context
├── subsequentprompt.md       ← Continue from where you left off
├── README.txt                ← Usage guide
└── doc/
    ├── architecture.md       ← Tech stack + architecture style
    ├── solution-structure.md ← Folder/layer layout
    ├── database-schema.md    ← Tables and fields
    ├── domain-model.md       ← Enums, invariants, business rules
    ├── api-contract.md       ← REST endpoints
    ├── security.md           ← Roles and auth rules
    ├── ui-guideline.md       ← Layout and UX rules
    ├── coding-standard.md    ← Language-specific standards
    ├── task-list.md          ← Phased task breakdown
    ├── changelog.txt         ← Auto-updated after every change
    ├── progress.txt          ← Current task pointer (kept minimal)
    └── Progress/
        └── Progress-N.txt    ← Per-task detailed logs
```

All files use `[FILL IN: ...]` placeholders — replace them with your project details and you're ready to code.

---

## Commands

### `/final-goal <description>`

Creates `CLAUDE.md` at your project root. Takes your raw project description, rephrases it into a polished goal statement, and includes the full doc folder navigation guide so Claude always knows which files to load.

```
/final-goal Build a multi-tenant SaaS task management platform
```

**What it creates (`CLAUDE.md`):**
- Enhanced final goal (2–4 polished sentences)
- Reference to `initialprompt.md` for all rules
- Doc navigation guide — which files to always load vs. load on demand

### `/setupcode`

Scaffolds the full project template — 15 files including `initialprompt.md`, `subsequentprompt.md`, and the entire `/doc` folder.

> Run `/final-goal` first to define what you're building, then `/setupcode` to scaffold the template.

---

## Install

### Option A — Project-level (this project only)

```bash
mkdir -p .claude/commands
curl -o .claude/commands/setupcode.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/setupcode.md
curl -o .claude/commands/final-goal.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/final-goal.md
```

### Option B — Global (available in all projects)

```bash
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/setupcode.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/setupcode.md
curl -o ~/.claude/commands/final-goal.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/final-goal.md
```

---

## Usage

1. Run `/final-goal <description>` to create `CLAUDE.md` with your project goal
2. Run `/setupcode` to generate the `/doc` folder and prompt files
3. Fill in all `[FILL IN: ...]` placeholders across the `/doc` files
    OR
   Ask Claude to read your project prompt and fill in the files for you
4. Claude auto-loads `CLAUDE.md` every session — no manual paste needed

```
Session 1:  Claude reads CLAUDE.md + initialprompt.md  →  implements Task 1
Session 2+: paste subsequentprompt.md                  →  continues from progress.txt
```

**Token efficiency:** Docs are tiered — architecture and task list load every session; schema, API, and domain docs load only when the task requires them. Only `progress.txt` (a 3-line pointer) is sent on follow-up sessions.

---

## How Claude stays on track

- Reads `task-list.md` to know what to build next
- Updates `progress.txt` after each task (short pointer — not a full history)
- Updates `changelog.txt` after every change
- Writes detailed logs to `doc/Progress/Progress-N.txt`
- Never invents schema, fields, or endpoints not defined in your docs

---

## License

MIT
