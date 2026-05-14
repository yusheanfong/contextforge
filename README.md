# ContextForge

Scaffold a complete AI-assisted coding workflow for Claude Code with a single command.

Run `/final-goal <description>` and get a comprehensive `CLAUDE.md` (auto-loaded every session) plus a structured `/doc` folder — ready to fill in for any project.

---

## What it creates

```
your-project/
├── CLAUDE.md                 ← Final goal + all AI rules (auto-loaded every session)
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

## Command

### `/final-goal <description>`

One command does everything:

1. **Enhances your goal** — rephrases your raw description into a polished 2–4 sentence goal
2. **Shows a plan** — previews what CLAUDE.md will contain and which /doc files will be created
3. **Waits for your confirmation** — type YES to proceed
4. **Creates everything** — CLAUDE.md + all 12 /doc files

```
/final-goal Build a multi-tenant SaaS task management platform
```

**What gets created (`CLAUDE.md`):**
- Enhanced final goal (2–4 polished sentences)
- Role placeholder (fill in your tech stack / AI expertise)
- Doc navigation guide — which files Claude loads every session vs. on demand
- 10 strict rules to keep Claude on track (1 customizable placeholder)
- Before/After every task checklists

---

## Install

### Option A — Project-level (this project only)

```bash
mkdir -p .claude/commands
curl -o .claude/commands/final-goal.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/final-goal.md
```

### Option B — Global (available in all projects)

```bash
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/final-goal.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/final-goal.md
```

---

## Usage

1. Run `/final-goal <description>` — Claude shows you the plan and asks for confirmation
2. Type **YES** to create CLAUDE.md and the full /doc folder
3. Fill in all `[FILL IN: ...]` placeholders  
   OR ask Claude to fill them in for you based on your project description
4. Start coding — Claude auto-loads CLAUDE.md every session

```
Session 1:  Claude auto-loads CLAUDE.md → reads task-list.md, progress.txt, architecture.md → implements Task 1
Session 2+: New session → same auto-load → continues from progress.txt
```

---

## How Claude stays on track

- Reads `task-list.md` to know what to build next
- Updates `progress.txt` after each task (short pointer — not a full history)
- Updates `changelog.txt` after every change
- Writes detailed logs to `doc/Progress/Progress-N.txt`
- Never invents schema, fields, or endpoints not defined in your docs

---

## Token efficiency

- CLAUDE.md (~600 tokens) auto-loads every session — no manual paste needed
- `architecture.md`, `task-list.md`, and `progress.txt` load every session (~300 tokens total)
- All other docs load only when the task requires them
- Per-task logs stay in `doc/Progress/` — never sent unless needed

---

## License

MIT
