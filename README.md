# ContextForge

A Claude Code skill that scaffolds a complete AI-assisted coding workflow in seconds.

Run `/setupcode` and get 17 template files — a structured `/doc` folder, prompts, changelogs, and progress tracking — ready to fill in for any project.

---

## What it does

When you run `/setupcode`, ContextForge creates:

```
your-project/
├── initialprompt.md          ← Paste once to give AI full context
├── subsequentprompt.md       ← Paste every session to continue work
├── README.txt                ← Usage guide
└── doc/
    ├── ai-instruction.md     ← AI behavior rules
    ├── architecture.md       ← Tech stack + architecture style
    ├── solution-structure.md ← Folder/layer layout
    ├── database-schema.md    ← Tables and fields
    ├── domain-model.md       ← Enums, invariants, business rules
    ├── api-contract.md       ← REST endpoints
    ├── security.md           ← Roles and auth rules
    ├── ui-guideline.md       ← Layout and UX rules
    ├── coding-standard.md    ← Language-specific standards
    ├── task-list.md          ← Phased task breakdown
    ├── llm-checklist.md      ← Pre-code confirmation checklist
    ├── changelog.txt         ← Auto-updated after every change
    ├── progress.txt          ← Minimal state sent each session
    └── Progress/
        └── Progress-N.txt    ← Per-task detailed logs
```

All files use `[FILL IN: ...]` placeholders — replace them with your project's details and you're ready to go.

---

## Install

### Option A – Project-level (recommended)

Copy `.claude/commands/setupcode.md` into your project's `.claude/commands/` folder:

```bash
mkdir -p .claude/commands
curl -o .claude/commands/setupcode.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/setupcode.md
```

### Option B – Global (available in all projects)

```bash
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/setupcode.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/setupcode.md
```

---

## Usage

1. Open Claude Code in your project folder
2. Type `/setupcode`
3. Fill in all `[FILL IN: ...]` placeholders in the generated `/doc` files
4. Paste `initialprompt.md` into your AI chat to start your first session
5. Use `subsequentprompt.md` for every session after that

---

## Workflow

```
Session 1:  paste initialprompt.md  →  AI reads all /doc files, implements Task 1
Session 2+: paste subsequentprompt.md  →  AI continues from progress.txt
```

**Token efficiency:** Full `/doc` context is loaded once. Only `progress.txt` is sent on every follow-up session — keeping token costs low.

---

## How the AI stays on track

- Reads `task-list.md` to know what to build next
- Updates `progress.txt` after each task (short, token-efficient)
- Updates `changelog.txt` after every change
- Writes detailed logs to `doc/Progress/Progress-N.txt`
- Never invents schema, fields, or endpoints not defined in your docs

---

## Compatible with

Any AI that supports custom slash commands or can be given a prompt file:
- Claude Code (`/setupcode` slash command)
- Any AI chat (paste the content of `initialprompt.md` / `subsequentprompt.md`)

---

## License

MIT
