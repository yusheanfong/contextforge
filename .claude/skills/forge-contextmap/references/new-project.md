## NEW PROJECT MODE

*Trigger: empty/near-empty repo, or `--new` flag*

### Step N1: Gather Project Info

Ask the user (one question at a time — wait for each answer before asking the next):

**Question 1 — Goal:** "What's the final goal of this project? Describe what you're building and who it's for."

Wait for answer. Then **enhance it**: rephrase the raw description into a polished 2–4 sentence goal statement that is clear, specific, and captures scope + audience. Show the enhanced version to the user:

```
Here's how I'd state the goal:

[ENHANCED_GOAL]

Does this capture it accurately? (or tell me what to adjust)
```

Wait for confirmation or corrections. Store the approved version as `[GOAL]`.

**Question 2 — Tech stack:** "What's the tech stack / language? (e.g. Flutter + Dart, React + TypeScript, ASP.NET Core, Django, etc.)"

Store answer as `[TECH_STACK]`. From the answer, infer two booleans:

- `[HAS_UI]` — true if the stack includes any frontend/mobile/desktop UI (Flutter, React, Vue, SwiftUI, WPF, etc.); false for pure APIs, CLIs, libraries.
- `[HAS_BACKEND]` — true if the stack includes a server, database, or backend-as-a-service (Express, Django, ASP.NET, Firebase, Supabase, Postgres, etc.).

Only if genuinely unclear from the answer, ask one follow-up: "Does this have a backend / database? Which one?" Do not ask if the stack already makes it obvious.

**Question 3 — Core features:** "List the core features you want (rough list is fine — I'll structure it)."

Wait for answer. Enhance the raw list into numbered, prioritized PRD features (`F1..Fn`, each `P0` must-have / `P1` important / `P2` nice-to-have, one line each). Show for approval:

```
Here's the structured feature list:

- **F1 (P0)** — [name]: [one-line description]
- **F2 (P0)** — ...
- **F3 (P1)** — ...

Anything to add, remove, reprioritize?
```

Wait for confirmation or corrections. Store as `[FEATURES]`.

**Question 4 — Design (only if `[HAS_UI]`):** "Describe the look/vibe you want (or give brand colors/fonts if you have them). e.g. 'clean fintech, trustworthy' or 'playful, colorful, rounded'."

Wait for answer, then **generate a complete concrete design system** — see **`references/design-brief-generation.md`**. Show the generated token set for approval; apply corrections. Store as `[DESIGN_SYSTEM]`.

If `[HAS_UI]` is false, skip this question and do not create `doc/design-brief.md`.

### Step N2: Scaffold All Files

Use the Write tool to create each file from the templates in `references/doc-templates.md` (File 1–File 13). Replace `[GOAL]`, `[TECH_STACK]`, `[FEATURES]`, `[DESIGN_SYSTEM]` with the gathered values. Do not skip any file (except the two conditional ones: `design-brief.md` only if `[HAS_UI]`, `backend-schema.md` only if `[HAS_BACKEND]`).

### Step N3: Generate Draft Engineering Plan (task list)

Based on `[GOAL]`, `[TECH_STACK]`, and `[FEATURES]`, generate a realistic phased engineering plan. Rules for the plan:

- Small tasks — each doable in one sitting (roughly one file cluster / one behavior).
- Explicit build order — task numbering `N.M` + `Depends on` lines.
- Every task links a PRD feature (`Builds: Fn`) where applicable.
- Every task carries the four standard acceptance criteria, with "works as expected" made specific to the task.

Write it to `doc/task-list.md`, then present it (together with the `doc/app-flow.md` draft) to the user:

```
Here's the draft engineering plan and app flow I created based on your goal and features.
Review them and tell me what to add, remove, or change before we finalize.
```

The task list format is defined in `references/doc-templates.md` ("Task List Format").

Wait for user feedback on the plan. Apply any changes they request. Once approved, confirm:

```
✅ /forge-contextmap setup complete!

Files created:
  CLAUDE.md
  doc/prd.md
  doc/app-flow.md
  doc/design-brief.md     [only if UI]
  doc/backend-schema.md   [only if backend]
  doc/architecture.md
  doc/domain-model.md
  doc/api-contract.md
  doc/solution-structure.md
  doc/coding-standard.md
  doc/security.md
  doc/task-list.md        ← APPROVED engineering plan, your master to-do
  doc/changelog.txt
  doc/progress.txt

Next steps:
1. Fill in [FILL IN: ...] placeholders in /doc files
2. Start coding — Claude will follow these docs automatically
3. Once you have source code, run /forge-contextmap sync to populate graph sections
```

### Step N4: Install Post-Commit Hook

After creating all files, install the post-commit hook:

Use the Bash tool to run:
```bash
mkdir -p .git/hooks
```

Then write `.git/hooks/post-commit` with this content:
```bash
#!/bin/sh
# contextmap: rebuild knowledge graph on commit
if command -v python >/dev/null 2>&1; then
    PYTHON=python
elif command -v python3 >/dev/null 2>&1; then
    PYTHON=python3
else
    exit 0
fi
if $PYTHON -m graphify --version >/dev/null 2>&1; then
    ( $PYTHON -m graphify . --update >/dev/null 2>&1 & )
fi
```

Make it executable:
```bash
chmod +x .git/hooks/post-commit
```

On Windows, the chmod may not apply — that's OK. The hook will still run in Git Bash or WSL.
