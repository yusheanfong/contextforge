# ContextForge

Scaffold a complete AI-assisted coding workflow in seconds — works with any AI (Claude, ChatGPT, Gemini, Copilot, Cursor, Windsurf, and more).

Run the setup prompt and get 15 template files — a structured `/doc` folder, session prompts, changelogs, and progress tracking — ready to fill in for any project.

---

## What it creates

```
your-project/
├── initialprompt.md          ← All AI rules + context (paste once to start)
├── subsequentprompt.md       ← Paste every follow-up session to continue
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

### `/setupcode`

Scaffolds the full project template — 15 files including `initialprompt.md`, `subsequentprompt.md`, and the entire `/doc` folder.

### `/final-goal <description>`

Creates a `CLAUDE.md` at your project root. Takes your raw project description, rephrases it into a polished goal statement, and includes the full doc folder navigation guide so the AI always knows which files to load.

```
/final-goal Build a multi-tenant SaaS task management platform
```

**What it creates (`CLAUDE.md`):**
- Enhanced final goal (2–4 polished sentences)
- Reference to `initialprompt.md` for all AI rules
- Doc navigation guide — which files to always load vs. load on demand

> Run `/final-goal` first to define what you're building, then `/setupcode` to scaffold the full template.

---

## Install

### Option A — Paste into any AI chat (no setup required)

Copy the raw content of [`setupcode.md`](https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/setupcode.md) and paste it into any AI chat (ChatGPT, Claude, Gemini, Copilot, etc.). The AI will create all 15 files in your project folder.

### Option B — Claude Code slash commands (project-level)

```bash
mkdir -p .claude/commands
curl -o .claude/commands/setupcode.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/setupcode.md
curl -o .claude/commands/final-goal.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/final-goal.md
```

Then type `/final-goal <your project description>` and `/setupcode` inside Claude Code.

### Option C — Claude Code slash commands (global, all projects)

```bash
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/setupcode.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/setupcode.md
curl -o ~/.claude/commands/final-goal.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/final-goal.md
```

Then type `/final-goal` and `/setupcode` in any project.

---

## Usage

1. Run `/final-goal <description>` to create `CLAUDE.md` with your project goal
2. Run `/setupcode` to generate the `/doc` folder and prompt files
3. Fill in all `[FILL IN: ...]` placeholders across the `/doc` files
    OR
   Give Claude (or your model of choice) the prompt of your project, and tell it to structure a plan for you and fill in the files
4. Tell your model to read `initialprompt.md` to start your first session
5. Use `subsequentprompt.md` for every session after that

```
Session 1:  paste initialprompt.md    →  AI reads /doc files, implements Task 1
Session 2+: paste subsequentprompt.md →  AI continues from progress.txt
```

**Token efficiency:** Docs are tiered — architecture and task list load every session; schema, API, and domain docs load only when the task requires them. Only `progress.txt` (a 3-line pointer) is sent on follow-up sessions.

---

## How the AI stays on track

- Reads `task-list.md` to know what to build next
- Updates `progress.txt` after each task (short pointer — not a full history)
- Updates `changelog.txt` after every change
- Writes detailed logs to `doc/Progress/Progress-N.txt`
- Never invents schema, fields, or endpoints not defined in your docs

---

## Auto-load for specific tools

All rules live in `initialprompt.md`. For tools that support auto-loading, create a one-liner file pointing to it:

| Tool | File to create | Content |
|---|---|---|
| Claude Code | `CLAUDE.md` | `Read and follow all rules in initialprompt.md before any action.` |
| Cursor | `.cursorrules` | same |
| Windsurf | `.windsurfrules` | same |
| GitHub Copilot | `.github/copilot-instructions.md` | same |

---

## Compatible AI tools

| Tool | How to use |
|---|---|
| **Claude Code** | Install via Option B or C above, then `/setupcode` |
| **ChatGPT / Gemini** | Paste `setupcode.md` content into chat (Option A) |
| **Cursor / Windsurf** | Paste `setupcode.md` into AI panel |
| **GitHub Copilot** | Paste `setupcode.md` into Copilot Chat |
| **Any other AI** | Paste `setupcode.md` content — it works in any chat |

---

## License

MIT
