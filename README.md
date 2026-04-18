# ContextForge

Scaffold a complete AI-assisted coding workflow in seconds — works with any AI (Claude, ChatGPT, Gemini, Copilot, Cursor, Windsurf, and more).

Run the setup prompt and get 17 template files — a structured `/doc` folder, session prompts, changelogs, and progress tracking — ready to fill in for any project.

---

## What it creates

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

All files use `[FILL IN: ...]` placeholders — replace them with your project details and you're ready to code.

---

## Install

### Option A — Paste into any AI chat (no setup required)

Copy the raw content of [`setupcode.md`](https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/setupcode.md) and paste it into any AI chat (ChatGPT, Claude, Gemini, Copilot, etc.). The AI will create all 17 files in your project folder.

### Option B — Claude Code slash command (project-level)

```bash
mkdir -p .claude/commands && curl -o .claude/commands/setupcode.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/setupcode.md
```

Then type `/setupcode` inside Claude Code.

### Option C — Claude Code slash command (global, all projects)

```bash
mkdir -p ~/.claude/commands && curl -o ~/.claude/commands/setupcode.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/setupcode.md
```

Then type `/setupcode` in any project.

---

## Usage

1. Run the setup (any option above) to generate the `/doc` folder and prompt files
2. Fill in all `[FILL IN: ...]` placeholders across the `/doc` files
3. Paste `initialprompt.md` into your AI chat to start your first session
4. Use `subsequentprompt.md` for every session after that

```
Session 1:  paste initialprompt.md   →  AI reads all /doc files, implements Task 1
Session 2+: paste subsequentprompt.md →  AI continues from progress.txt
```

**Token efficiency:** Full `/doc` context is loaded once. Only `progress.txt` is sent on follow-up sessions — keeping token costs low.

---

## How the AI stays on track

- Reads `task-list.md` to know what to build next
- Updates `progress.txt` after each task (short, token-efficient)
- Updates `changelog.txt` after every change
- Writes detailed logs to `doc/Progress/Progress-N.txt`
- Never invents schema, fields, or endpoints not defined in your docs

---

## Compatible AI tools

| Tool | How to use |
|---|---|
| **Claude Code** | Install via Option B or C above, then `/setupcode` |
| **ChatGPT / Gemini** | Paste `setupcode.md` content into chat (Option A) |
| **Cursor / Windsurf** | Paste `setupcode.md` into AI panel, or add `.cursorrules` pointing to `/doc` |
| **GitHub Copilot** | Paste `setupcode.md` into Copilot Chat |
| **Any other AI** | Paste `setupcode.md` content — it works in any chat |

---

## License

MIT
