# /forge-contextmap — Graph-Powered Context Scaffold

Combines Graphify (code → knowledge graph) with ContextForge-style AI workflow docs.
Reads $ARGUMENTS to detect subcommand: `sync` or `--new`. Otherwise auto-detects mode.

Doc format version: **v2** — PRD, app flow, design brief, backend schema, engineering-plan task
list. Old-format (v1) projects are auto-detected and upgraded via **MIGRATION MODE** with all user
content preserved.

---

## STEP 1: Detect Mode

Run these checks in order and jump to the matching section:

1. If `$ARGUMENTS` contains `sync` → go to **SYNC MODE**
2. If `$ARGUMENTS` contains `--new` → go to **NEW PROJECT MODE**
3. If `doc/` exists AND contains at least one ContextForge doc (`architecture.md`,
   `task-list.md`, `solution-structure.md`, or `ui-guideline.md`) AND `CLAUDE.md` does NOT
   contain the marker `<!-- contextforge:format v2 -->` → go to **MIGRATION MODE**
   (old-format project — upgrade it)
4. Check if `graphify-out/graph.json` exists in the current directory
   - If yes → go to **SYNC MODE**
5. Count source files: `.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.dart`, `.go`, `.cs`, `.java`, `.rb`, `.rs`, `.swift`, `.kt`, `.cpp`, `.c`, `.h`
   - If any found → go to **EXISTING PROJECT MODE**
6. Otherwise → go to **NEW PROJECT MODE**

---

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

Wait for answer, then **generate a complete concrete design system** — see **DESIGN BRIEF GENERATION** at the bottom of this file. Show the generated token set for approval; apply corrections. Store as `[DESIGN_SYSTEM]`.

If `[HAS_UI]` is false, skip this question and do not create `doc/design-brief.md`.

### Step N2: Scaffold All Files

Use the Write tool to create each file below. Replace `[GOAL]`, `[TECH_STACK]`, `[FEATURES]`, `[DESIGN_SYSTEM]` with the gathered values. Do not skip any file (except the two conditional ones: `design-brief.md` only if `[HAS_UI]`, `backend-schema.md` only if `[HAS_BACKEND]`).

**File 1: `CLAUDE.md`**

```
# Project Context
<!-- contextforge:format v2 -->

## Goal
[GOAL]

## Tech Stack
[TECH_STACK]

## Doc Navigation
All project docs live in /doc:
- doc/prd.md              — Product requirements: idea overview, core features (F1..Fn), out of scope
- doc/app-flow.md         — Entry point, screen/step map, user journeys, data flow
- doc/design-brief.md     — Color tokens, typography, components, screen style   [omit this line if no UI]
- doc/backend-schema.md   — Storage, entities/tables, relations, indexes         [omit this line if no backend]
- doc/architecture.md     — Tech stack, layers, design patterns
- doc/domain-model.md     — Entities, enums, business rules
- doc/api-contract.md     — API endpoints or service interfaces
- doc/solution-structure.md — Project folder layout
- doc/coding-standard.md  — Language and framework conventions
- doc/security.md         — Auth, roles, data protection rules
- doc/task-list.md        — Engineering plan / master task list (YOUR ONLY TODO SOURCE)
- doc/changelog.txt       — Change log
- doc/progress.txt        — Current status

## Graph Sync
Sections marked <!-- graphify:auto start:... --> are auto-populated by /forge-contextmap sync.
Edit ONLY outside these markers — content inside is overwritten on sync.
When Graphify runs for the first time (after code exists), run /forge-contextmap sync to populate.

## Rules
1. Before writing code, ALWAYS read doc/architecture.md, doc/solution-structure.md, doc/coding-standard.md, and doc/prd.md (universal constraints + scope guard). Then review doc/task-list.md and graphify-out/GRAPH_REPORT.md to decide which domain docs the current task touches, and read ONLY those:
   - UI/screen/widget/component task → doc/design-brief.md + doc/app-flow.md
   - data/entity/model task → doc/domain-model.md + doc/backend-schema.md (if present)
   - API/service/endpoint task → doc/api-contract.md + doc/backend-schema.md (if present)
   - auth/permission task → doc/security.md
2. Implement ONLY the next task in doc/task-list.md whose "Depends on" tasks are all done.
3. Update doc/progress.txt after every completed task.
4. Update doc/changelog.txt after every change (format: Date | Change | Description).
5. Follow doc/solution-structure.md exactly — no structural changes.
6. UI code: use ONLY doc/design-brief.md tokens and components — no ad-hoc hex values, font sizes, spacing values, or one-off components.
7. Never invent schema fields, entities, or endpoints not defined in doc/backend-schema.md, doc/domain-model.md, or doc/api-contract.md.

## Coding Rules

### Think Before Coding
State assumptions before acting. If uncertain, ask — don't guess.
- Multiple interpretations → present them, don't silently pick one
- Simpler approach exists → say so and push back
- Confused by a requirement → name what's confusing, stop, ask

### Simplicity First
Write the minimum code that solves the stated problem. Nothing speculative.
- No features beyond what was asked
- No abstractions for single-use code
- No "future flexibility" that wasn't requested
- No error handling for impossible scenarios
- If it could be half the length, rewrite it

### Surgical Changes
Touch only what the task requires. Match existing style.
- Don't improve adjacent code, formatting, or comments
- Don't refactor things that aren't broken
- If you notice unrelated dead code, mention it — don't delete it
- Remove imports/variables/functions YOUR changes made unused; leave pre-existing dead code alone

Every changed line must trace directly to the user's request.

### Goal-Driven Execution
Define what "done" looks like before writing code.

Turn tasks into verifiable goals:
- "Add validation" → tests for invalid inputs pass
- "Fix the bug" → test reproduces it, then it passes
- "Refactor X" → tests pass before and after; nothing changed externally

For multi-step tasks, state a brief plan with a verify step per step:
1. [Step] → verify: [check]
2. [Step] → verify: [check]
```

Note: rules 1/6/7 reference `design-brief.md` / `backend-schema.md` — drop those references from the rules text too when the corresponding doc isn't created.

**File 2: `doc/prd.md`** — fully user-owned, no graph fence:

```
# Product Requirements

## Idea Overview
[GOAL]

## Core Features
<!-- Priority: P0 = must-have for launch, P1 = important, P2 = nice-to-have -->
[FEATURES — the approved F1..Fn list]

## Out of Scope
[Derive 2-4 explicit non-goals from the conversation, or write "[FILL IN: what you're explicitly NOT building]"]
```

**File 3: `doc/app-flow.md`** — draft it FROM `[GOAL]` + `[FEATURES]` (real content, not placeholders):

```
# App Flow

<!-- graphify:auto start:project:app-flow -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:app-flow -->

## Entry Point
[e.g. app launch → auth check → home screen; or for an API: request → middleware → router]

## Screen / Step Map
1. [Screen/Step] — [purpose]
2. [Screen/Step] — [purpose]
...

## User Journeys
[One journey per P0 feature:]
### F1 — [feature name]
[step] → [step] → [outcome]

## Data Flow
[Where data originates, how it moves through the app, where it's persisted]
```

**File 4: `doc/design-brief.md`** — ONLY if `[HAS_UI]`. Fill from the approved `[DESIGN_SYSTEM]`:

```
# Design Brief

Single source of truth for look & feel. Consistency over novelty — every screen uses these
tokens and components. No randomness.

<!-- graphify:auto start:project:design-brief -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:design-brief -->

## Color Tokens
| Token   | Hex     | Usage |
|---------|---------|-------|
| Primary | [#hex]  | buttons, links, active states |
| Surface | [#hex]  | backgrounds, cards |
| Text    | [#hex]  | primary text |
| Muted   | [#hex]  | secondary text, placeholders |
| Border  | [#hex]  | dividers, input borders |
| Success | [#hex]  | success states |
| Error   | [#hex]  | errors, destructive actions |

## Typography
- Family: [font]
- Scale: [e.g. 32 / 24 / 18 / 16 / 14] (display / h1 / h2 / body / caption)
- Weights: [e.g. 400 regular, 500 medium, 700 bold]

## Spacing & Radius
- Base unit: [e.g. 4px] — allowed spacing: [e.g. 4 / 8 / 12 / 16 / 24 / 32]
- Radius: [e.g. 8px cards & inputs, 999px pills]

## Reusable Components
[From DESIGN_SYSTEM, e.g.:]
- Button: primary / secondary / ghost
- Card
- Input (text, with label + error state)
- Modal
- Toast (success / error)

## Screen Style Guidance
- List screens: [e.g. cards in a single column, 16px gaps, pull-to-refresh]
- Detail screens: [e.g. hero section + sectioned content]
- Forms: [e.g. single column, labels above inputs, primary action pinned bottom]
- Empty states: [e.g. icon + one line + primary action]

## UX Rules
- Confirm before destructive actions
- Toast notifications for success/error
[FILL IN: more rules as they emerge]

## Hard Rule
Use ONLY the tokens and components defined above. No ad-hoc hex values, font sizes, spacing
values, or one-off components. Need something new? Add it HERE first, then use it.
```

**File 5: `doc/backend-schema.md`** — ONLY if `[HAS_BACKEND]`:

```
# Backend Schema

<!-- graphify:auto start:project:backend-schema -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:backend-schema -->

## Storage
[e.g. Postgres 16 via Prisma / Firestore / SQLite + Drift]

## Entities / Tables
### [table/collection name]
| Field | Type | Constraints |
|-------|------|-------------|
| [FILL IN] | | |

## Relations
[e.g. User 1—N Order; Order 1—N OrderItem]

## Indexes
[FILL IN: fields queried often]

## Auth & Ownership
See doc/security.md — roles, ownership filters, and data-protection rules live there.
```

**File 6: `doc/architecture.md`**

```
# Architecture — [GOAL summary]

## Tech Stack
[TECH_STACK]

<!-- graphify:auto start:project:architecture -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:architecture -->

## Architecture Style
[FILL IN: e.g. Clean Architecture / MVC / Layered / Feature-first]

## Project Layers
[FILL IN: list your layers, e.g. Domain / Application / Infrastructure / API]

## Naming Conventions
[FILL IN: e.g. PascalCase for classes, camelCase for variables]

## Non-Negotiable Rules
[FILL IN: e.g. No direct DB access in controllers]
```

**File 7: `doc/domain-model.md`**

```
# Domain Model

<!-- graphify:auto start:project:domain-model -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:domain-model -->

## Enums
[FILL IN: list enum names and values]

## Entity Invariants
[FILL IN: e.g. Order must belong to a Customer]

## Business Rules
[FILL IN: e.g. Cannot delete record if child records exist]
```

**File 8: `doc/api-contract.md`**

```
# API Contract

<!-- graphify:auto start:project:api-contract -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:api-contract -->

## Endpoints
[FILL IN: list endpoints grouped by resource]

## Notes
[FILL IN: e.g. JWT required on all endpoints except /auth]
```

**File 9: `doc/solution-structure.md`**

```
# Solution Structure

## Tech Stack
[TECH_STACK]

<!-- graphify:auto start:project:solution-structure -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:solution-structure -->

## Root Layout
[FILL IN: describe your folder structure]

## Layer Dependency Rules
[FILL IN: which layers depend on which]

## AI Instruction
Follow this structure exactly. Do not invent new layers or folders.
```

**File 10: `doc/coding-standard.md`**

```
# Coding Standards

## Language: [TECH_STACK]

<!-- graphify:auto start:project:coding-standard -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:coding-standard -->

## Rules
[FILL IN: e.g. Use async/await everywhere]
[FILL IN: e.g. Validate all inputs at boundary]
[FILL IN: e.g. No magic strings — use constants]
[FILL IN: e.g. Use dependency injection for all services]
```

**File 11: `doc/security.md`**

```
# Security Model

<!-- graphify:auto start:project:security -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:security -->

## Roles
[FILL IN: list roles, e.g. Admin / User / Viewer]

## Rules
[FILL IN: e.g. JWT required for all endpoints]
[FILL IN: e.g. All queries filtered by OwnerId]
[FILL IN: e.g. Soft delete only — no hard deletes]
```

**File 12: `doc/changelog.txt`**

```
# Changelog
# Format: Date | Change | Description

[TODAY'S DATE] | Project initialized | /forge-contextmap new project setup
```

Replace `[TODAY'S DATE]` with today's actual date (YYYY-MM-DD).

**File 13: `doc/progress.txt`**

```
# Progress

## Current Status
Project initialized. No tasks completed yet.

## Summary
Waiting to begin Phase 1.
```

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

The task list format:

```markdown
# Master Task List (Engineering Plan)

## Goal
[GOAL]

## How to work this list
- Build order = task numbering; a task is eligible when all its "Depends on" tasks are done.
- Implement ONLY the next eligible task.
- A task is done when every acceptance criterion is ticked.

## Phase 1 — Foundation
### Task 1.1 — [title]
- [ ] done
- Depends on: none
- Builds: [Fn or —]
- Acceptance criteria:
  - [ ] works as expected: [specific expected behavior for THIS task]
  - [ ] no errors (lint/console clean)
  - [ ] meets PRD requirement [Fn]
  - [ ] test added

### Task 1.2 — [title]
- [ ] done
- Depends on: Task 1.1
- Builds: [Fn]
- Acceptance criteria:
  - [ ] works as expected: [...]
  - [ ] no errors (lint/console clean)
  - [ ] meets PRD requirement [Fn]
  - [ ] test added

## Phase 2 — Core Features
[...]

## Phase 3 — Polish & Launch
[...]

## Notes
Update this file as requirements evolve. Keep tasks small — one sitting each.
```

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

---

## EXISTING PROJECT MODE

*Trigger: source files found, no graph.json yet, no old-format docs*

### Step E1: Python Prerequisite Check

Use the Bash tool to check Python version:

```bash
python --version 2>&1 || python3 --version 2>&1
```

Parse the version output. If Python is not found OR version is below 3.10:
- Print this exact error and STOP:
  ```
  ❌ Graphify requires Python 3.10 or higher.
  
  Found: [version found, or "Python not found"]
  
  Fix options:
  - Install Python 3.10+: https://www.python.org/downloads/
  - On Windows, ensure 'python' or 'python3' is on your PATH
  
  After fixing, run /forge-contextmap again.
  ```

Determine which command works (`python` or `python3`) and use it for all subsequent commands. Store as `[PYTHON_CMD]`.

### Step E2: Install Graphify

Check if Graphify is installed:
```bash
[PYTHON_CMD] -m graphify --version 2>&1
```

If not found, install it. Choose the target by environment so we never break a managed env nor pollute the global one — `--user` is invalid inside a virtualenv, so only use it when no env is active:

- **If a virtualenv/conda/poetry env is active** — detect via `$VIRTUAL_ENV` being set, or `[PYTHON_CMD] -c "import sys; print(sys.prefix != sys.base_prefix)"` printing `True` — install into it normally:
  ```bash
  [PYTHON_CMD] -m pip install graphifyy
  ```
- **Otherwise** (no managed env active), use `--user` to avoid touching global/system site-packages and to avoid needing sudo:
  ```bash
  [PYTHON_CMD] -m pip install --user graphifyy
  ```

Either way, `python -m graphify` resolves afterward. Note: the PyPI package name is `graphifyy` (double y) — this is correct and verified, NOT a typo. Do not change it to `graphify`. After pip install, run the post-install setup:
```bash
graphify install
```

If `graphify install` fails (command not found), try:
```bash
[PYTHON_CMD] -m graphify install
```

### Step E3: Analyze Codebase

Run the graph builder:
```bash
[PYTHON_CMD] -m graphify .
```

This will take time depending on codebase size. Wait for it to complete. It produces:
- `graphify-out/graph.json` — the knowledge graph
- `graphify-out/GRAPH_REPORT.md` — human-readable summary

If it fails, print the error and ask the user to check their Graphify installation.

### Step E4: Parse Graph and Extract Understanding

Use the Read tool to read `graphify-out/graph.json`.

From the JSON, extract:
- **God nodes**: nodes with the highest number of connections (edges). Sort nodes by degree (count of edges involving that node). Take top 10.
- **Communities**: group nodes by their `community` field. For each community, list the node labels and the most common `source_file` paths.
- **Entry points**: nodes whose `source_file` contains `main`, `index`, `app`, `program`, `entrypoint`, or similar.
- **API surface**: nodes with edges of confidence `EXTRACTED` that cross community boundaries, or nodes whose label contains `api`, `service`, `controller`, `handler`, `endpoint`, `route`.
- **High-confidence edges**: edges with confidence `EXTRACTED` (facts the parser confirmed from AST). Note dominant patterns.
- **UI presence** (`[HAS_UI]`): true if nodes look like screens/widgets/views/components/pages.
- **Backend presence** (`[HAS_BACKEND]`): true if nodes look like ORM models, SQL/migration files, controllers/handlers, or DB clients.

Also read `graphify-out/GRAPH_REPORT.md` for the summary Graphify generated.

### Step E5: Present Understanding to User

Present a structured summary. Use the information extracted in E4:

```
Here's what I understand about this codebase after analyzing it with Graphify:

**Architecture Pattern**: [inferred from community structure and god nodes — e.g. "Layered Flutter app with BLoC state management" or "Express REST API with service-repository pattern"]

**Key Components (God Nodes)**:
[list top 5-8 god nodes with their source_file paths]

**Main Subsystems** ([N] communities detected):
[for each community: name it based on the node labels, list 3-5 key nodes]

**Entry Points**:
[list detected main/index files]

**Public API Surface**:
[list exported interfaces, services, controllers, or endpoints detected]

**Dominant Patterns** (from high-confidence edges):
[list 2-4 patterns, e.g. "Dependency injection throughout", "Event-driven communication between modules"]

**Core Features (inferred)** — for doc/prd.md:
[infer F1..Fn with priorities from the subsystems/screens/endpoints detected]

---
Does this match your understanding of the project?
- Confirm if it's accurate
- Correct anything that's wrong
- Add anything important I missed (e.g. "The auth module isn't in source yet — we use Firebase Auth")
- Fix the feature list too — it becomes your PRD

I'll incorporate your corrections before populating the doc files.
```

Wait for user response. Apply any corrections to your understanding. Store the corrected feature list as `[FEATURES]`.

**If `[HAS_UI]`:** also ask the design question now (same as N1 Question 4): "Describe the look/vibe you want (or brand colors/fonts if you have them)." Then generate the concrete design system per **DESIGN BRIEF GENERATION** and get approval. Store as `[DESIGN_SYSTEM]`. If the codebase already has an obvious design system (theme file, tokens), extract from THAT first and present it — don't invent a competing one.

### Step E6: Check for Existing Docs

Check if `doc/` folder exists with existing doc files.

- If docs exist AND `CLAUDE.md` has the `<!-- contextforge:format v2 -->` marker → v2 docs already present: preserve all user-owned content, write only into auto-fenced sections, and create any v2 docs that are missing.
- If docs exist WITHOUT the marker → you should be in MIGRATION MODE (Step 1 check 3) — go there.
- If no docs exist, create the full v2 set from scratch (Step E7).

### Step E7: Create/Update All Doc Files

For each doc file, use the fence format to mark graph-generated content. All content outside fences is either user-provided (preserved) or a placeholder prompt (for new files).

**CLAUDE.md** — create or update. Use the SAME v2 template as NEW PROJECT MODE File 1 (including the `<!-- contextforge:format v2 -->` marker, v2 Doc Navigation, Rules 1–7, and all four Coding Rules blocks), with these differences:

- `## Goal` — Ask user: "What's the goal of this project?" if not already known from E5. Use their answer.
- `## Tech Stack` — inferred from graph + user confirmation.
- Add this section after Tech Stack:
  ```
  ## Key Architecture (from Graphify)
  <!-- graphify:auto start:project:claude-summary -->
  **God nodes**: [top 5 god node labels]
  **Communities**: [community names/descriptions]
  **Entry points**: [main files]
  <!-- graphify:auto end:project:claude-summary -->
  ```
- Omit the design-brief / backend-schema navigation lines and rule references when `[HAS_UI]` / `[HAS_BACKEND]` are false.

**doc/prd.md** — from the E5-confirmed understanding (user-owned, no fence):

```markdown
# Product Requirements

## Idea Overview
[goal from user / E5 exchange]

## Core Features
<!-- Priority: P0 = must-have for launch, P1 = important, P2 = nice-to-have -->
[FEATURES — confirmed in E5]

## Out of Scope
[FILL IN: what you're explicitly NOT building]
```

**doc/app-flow.md**:

```markdown
# App Flow

<!-- graphify:auto start:project:app-flow -->
## Detected Flow

### Entry Points
[detected main/index files]

### Screens / Routes / Steps
[detected screen, route, page, or handler nodes with source paths]
<!-- graphify:auto end:project:app-flow -->

## Screen / Step Map
[FILL IN or draft from detected screens — numbered pipeline]

## User Journeys
[Draft one per P0 feature from FEATURES]

## Data Flow
[FILL IN — user-owned]
```

**doc/design-brief.md** — ONLY if `[HAS_UI]`. Same template as NEW PROJECT MODE File 4, filled from `[DESIGN_SYSTEM]`, with the fence populated:

```markdown
<!-- graphify:auto start:project:design-brief -->
## Detected UI Components

[list UI-related nodes: screens, widgets, views, components, pages]
[note any design pattern detected, e.g. BLoC, Redux, MVVM]
[note any existing theme/token files detected]
<!-- graphify:auto end:project:design-brief -->
```

**doc/backend-schema.md** — ONLY if `[HAS_BACKEND]`. Same template as NEW PROJECT MODE File 5, with the fence populated:

```markdown
<!-- graphify:auto start:project:backend-schema -->
## Detected Schema

### Entities / Models
[detected ORM model / entity nodes with fields where extractable]

### Data Access
[detected repositories, DAOs, DB clients, migration files]
<!-- graphify:auto end:project:backend-schema -->
```

**doc/architecture.md**:

```markdown
# Architecture

## Tech Stack
[from graph + user confirmation]

<!-- graphify:auto start:project:architecture -->
## Detected Architecture

### God Nodes (Central Concepts)
[list top god nodes with source paths]

### Module Relationships
[describe dominant dependency patterns from high-confidence edges]

### Key Abstractions
[list key interfaces/base classes detected]
<!-- graphify:auto end:project:architecture -->

## Architecture Style
[FILL IN or confirm from graph]

## Non-Negotiable Rules
[FILL IN]
```

**doc/domain-model.md**:

```markdown
# Domain Model

<!-- graphify:auto start:project:domain-model -->
## Detected Entities and Concepts

### Core Entities (from graph communities)
[for each community: list entities, their fields if detectable]

### Relationships
[list key entity relationships from graph edges]

### Enums and Types
[list detected enum types and values]
<!-- graphify:auto end:project:domain-model -->

## Business Rules
[FILL IN — user-owned, never overwritten]
```

**doc/api-contract.md**:

```markdown
# API Contract

<!-- graphify:auto start:project:api-contract -->
## Detected API Surface

### Services / Controllers
[list detected service/controller nodes with their source paths]

### Key Interfaces
[list public interfaces detected from EXTRACTED edges]

### External Dependencies
[list detected external API calls or integrations]
<!-- graphify:auto end:project:api-contract -->

## Endpoint Definitions
[FILL IN — user-owned, never overwritten]
```

**doc/solution-structure.md**:

```markdown
# Solution Structure

<!-- graphify:auto start:project:solution-structure -->
## Detected Folder Structure

[list the main source folders and their dominant node types]

## Community Map
[map each community to its folder(s)]

## Dependency Flow
[describe the dependency direction inferred from graph edges]
<!-- graphify:auto end:project:solution-structure -->

## Layer Rules
[FILL IN — user-owned]
```

**doc/coding-standard.md**:

```markdown
# Coding Standards

<!-- graphify:auto start:project:coding-standard -->
## Detected Patterns

[list coding patterns inferred from EXTRACTED edges, e.g.:
- "Dependency injection: constructor injection used throughout"
- "Async/await: all I/O operations are async"
- "Repository pattern: data access abstracted behind interfaces"]
<!-- graphify:auto end:project:coding-standard -->

## Standards
[FILL IN — user-owned]
```

**doc/security.md**:

```markdown
# Security Model

<!-- graphify:auto start:project:security -->
## Detected Security Components

[list auth-related nodes: token handlers, middleware, auth services, etc.]
[list permission/role-related nodes if detected]
<!-- graphify:auto end:project:security -->

## Roles
[FILL IN — user-owned]

## Rules
[FILL IN — user-owned]
```

If `doc/task-list.md` does not exist, create it in the engineering-plan format (same as NEW PROJECT MODE Step N3 — Goal, "How to work this list", `### Task N.M` blocks with `done` / `Depends on` / `Builds` / `Acceptance criteria`), with Phase 1 inferred from the codebase state.

If `doc/task-list.md` already exists, `/forge-contextmap` never rewrites its content — it's
user-authored. (Note: `/forge-orchestrate` is the one exception — on completion it ticks a task's
`[ ]` → `[x]`, or appends a `[x]` entry under `## Completed (orchestrated)` when the shipped feature
wasn't on the list. That is status bookkeeping, not a contextmap rewrite.)

Create `doc/changelog.txt`, `doc/progress.txt` only if they don't exist (same templates as NEW PROJECT MODE).

### Step E8: Install Post-Commit Hook

Same as Step N4 in New Project Mode.

### Step E9: Confirm

Print:

```
✅ /forge-contextmap setup complete!

Graph analyzed: graphify-out/graph.json
Docs populated with graph data (sections marked <!-- graphify:auto -->)

Files created/updated:
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
  [doc/task-list.md — created if it didn't exist]
  doc/changelog.txt
  doc/progress.txt

Graph sections will auto-update whenever you run /forge-contextmap sync.
User-owned content (outside <!-- graphify:auto --> markers) is never touched.

Next: Fill in [FILL IN] placeholders in /doc files, then start coding.
```

---

## MIGRATION MODE

*Trigger: old-format `doc/` present (ContextForge docs exist, but CLAUDE.md lacks the
`<!-- contextforge:format v2 -->` marker)*

Upgrades a v1 project to the v2 doc format. **Guarantee: no user-authored line is ever dropped or
reworded — content is moved, never rewritten.**

### Step M1: Inventory

Read every file in `doc/` plus `CLAUDE.md`. Note which v1 docs exist, which have
`<!-- graphify:auto -->` fences, and where user content lives (outside fences). Also detect
`[HAS_UI]` / `[HAS_BACKEND]` — from the graph if `graphify-out/graph.json` exists, otherwise from
source-file extensions and doc contents.

Announce:

```
Old-format ContextForge docs detected — upgrading to v2.
New in v2: doc/prd.md, doc/app-flow.md, doc/design-brief.md (absorbs ui-guideline.md),
doc/backend-schema.md (if backend), and an engineering-plan task-list format.
All your existing content is preserved verbatim.
```

### Step M2: Create `doc/prd.md`

- `## Idea Overview` — seed from the `## Goal` section of the existing `CLAUDE.md` (or
  task-list.md's `## Goal`). If neither exists, ask the user for the goal (enhance + approve, as in
  N1 Question 1).
- `## Core Features` — infer F1..Fn with priorities from the existing task list, docs, and graph
  (if present). Present the inferred list for confirmation/correction before writing (same exchange
  style as N1 Question 3).
- `## Out of Scope` — `[FILL IN]` placeholder.

### Step M3: Create `doc/app-flow.md`

Use the EXISTING PROJECT MODE template. If `graphify-out/graph.json` exists, populate the
`project:app-flow` fence from detected entry points/screens/routes; otherwise leave the fence with
the "no graph data yet" placeholder. Draft `## User Journeys` from the confirmed P0 features.

### Step M4: Create `doc/design-brief.md` (absorbs `ui-guideline.md`)

Only if `[HAS_UI]` OR `doc/ui-guideline.md` exists:

1. Create `doc/design-brief.md` from the standard template.
2. Move **ALL** content of `doc/ui-guideline.md` (including its fence and fenced content, if any)
   verbatim into a `## UX Rules (migrated from ui-guideline.md)` section — every line preserved.
3. Get concrete tokens: if the codebase has a theme/token file, extract values from it; otherwise
   ask the vibe question (N1 Question 4) and generate per **DESIGN BRIEF GENERATION**; approve.
4. Ask the user: "ui-guideline.md's content now lives in design-brief.md — OK to delete
   doc/ui-guideline.md?" **Delete only after explicit yes.** If no, leave it and add a one-line
   pointer at its top: `> Superseded by doc/design-brief.md — new content goes there.`

### Step M5: Create `doc/backend-schema.md`

Only if `[HAS_BACKEND]`: standard template; populate the fence from the graph if present.

### Step M6: Upgrade `doc/task-list.md` in place

Convert flat checkbox lines to the engineering-plan format. Conversion rules — mechanical, no
rewording:

- Keep the existing `## Goal` and `## Phase N — ...` headings as-is.
- Each `- [ ] <text>` line under Phase N becomes (numbering by order within the phase):
  ```markdown
  ### Task N.M — <text, verbatim>
  - [ ] done
  - Depends on: none   <!-- FILL IN if this task needs an earlier one -->
  - Builds: [Fn if it clearly maps to a PRD feature, else —]
  - Acceptance criteria:
    - [ ] works as expected: <text, verbatim>
    - [ ] no errors (lint/console clean)
    - [ ] meets PRD requirement [Fn or —]
    - [ ] test added
  ```
- Each `- [x] <text>` (already completed) becomes:
  ```markdown
  ### Task N.M — <text, verbatim>
  - [x] done (migrated as completed — criteria not retro-verified)
  - Depends on: none
  ```
  Do NOT fabricate ticked acceptance criteria for work you didn't verify.
- Default every `Depends on` to `none` — do not invent dependencies; the FILL-IN comment invites
  the user to add real ones.
- Keep any non-checkbox lines (notes, sections like `## Completed (orchestrated)`) exactly where
  they are.
- Add the `## How to work this list` block (from the N3 template) after `## Goal`.

Show the converted file to the user before writing it. Count lines: every v1 task line must appear
verbatim in the v2 version.

### Step M7: Upgrade `CLAUDE.md`

- Add the `<!-- contextforge:format v2 -->` marker under the title.
- Replace the `## Doc Navigation` and `## Rules` sections with the v2 versions (NEW PROJECT MODE
  File 1), keeping the conditional lines consistent with `[HAS_UI]`/`[HAS_BACKEND]`.
- Keep `## Goal`, `## Tech Stack`, `## Key Architecture`, `## Graph Sync`, `## Coding Rules`, and
  ANY sections the user added themselves — preserved verbatim, untouched.

### Step M8: Report

```
✅ Migration to v2 complete!

Created:
  doc/prd.md              (features confirmed by you)
  doc/app-flow.md
  doc/design-brief.md     [if created — absorbed ui-guideline.md]
  doc/backend-schema.md   [if created]

Upgraded in place:
  doc/task-list.md        → engineering-plan format ([N] tasks converted, [M] kept completed)
  CLAUDE.md               → v2 navigation + rules, format marker stamped

Deleted (with your OK):
  doc/ui-guideline.md     [only if approved]

Guarantee: every task line and every user-authored doc line was preserved verbatim.
Next: review the "Depends on: none" defaults in doc/task-list.md and fill in real dependencies,
then run /forge-contextmap sync (if you have a graph) to refresh the new fences.
```

---

## SYNC MODE

*Trigger: `graphify-out/graph.json` exists, or `sync` argument*

### Step S1: Python Check

Same as Step E1. Detect `[PYTHON_CMD]`.

### Step S2: Rebuild Graph

Run incremental graph update:
```bash
[PYTHON_CMD] -m graphify . --update
```

If `--update` flag is not supported by the installed version, fall back to:
```bash
[PYTHON_CMD] -m graphify .
```

Wait for completion.

### Step S3: Parse Updated Graph

Re-read `graphify-out/graph.json`. Extract the same fields as Step E4.

### Step S3.5: Diff Graph and Draft Changelog

Auto-draft structural changelog entries from what actually changed, so the log captures real mutations instead of vague summaries. Use the Read tool on `graphify-out/graph.prev.json` (the baseline saved at the end of the last sync):

1. **If `graphify-out/graph.prev.json` does not exist** (first sync): skip the diff — you'll create the baseline in step 4 below. Note "changelog baseline created — no diff yet."
2. **If it exists**: build a node-identity key for each node as `label + "@" + source_file`, then compute:
   - `added` = nodes in current `graph.json` but not in `graph.prev.json`
   - `removed` = nodes in `graph.prev.json` but not in current `graph.json`
   - For file-level context, also run (best-effort — ignore failure): `git diff --name-only HEAD~1 HEAD 2>/dev/null`
3. **Append a draft block** to `doc/changelog.txt`. This file is user-owned, so mark the block clearly as an editable draft. Use the existing `Date | Change | Description` line style:
   ```
   [TODAY'S DATE] | Auto-draft (review/edit) | [Added] <label> (<source_file>) ; [Removed] <label> (<source_file>)
   ```
   List the most significant added/removed nodes (cap ~10 each to stay readable). If nothing structural changed, write one line noting "no structural changes detected."
4. **Update the baseline** for next time — use the Bash tool:
   ```bash
   cp graphify-out/graph.json graphify-out/graph.prev.json
   ```

Keep the `removed` list in memory — Step S4 uses it for tombstones.

### Step S4: Fence-Aware Merge

For each doc file that has `<!-- graphify:auto start:... -->` markers (v2 set:
`CLAUDE.md` claude-summary, `architecture.md`, `domain-model.md`, `api-contract.md`,
`solution-structure.md`, `coding-standard.md`, `security.md`, `app-flow.md`, `design-brief.md`,
`backend-schema.md` — whichever exist):

1. Read the entire file
2. Find all fence pairs: `<!-- graphify:auto start:KEY -->` ... `<!-- graphify:auto end:KEY -->`
3. For each fence pair:
   - Generate new content based on the updated graph data
   - Replace ONLY the content between the start and end markers
   - Keep the markers themselves intact
   - Keep all content outside the markers exactly as-is
4. **Tombstones**: if a node from the S3.5 `removed` list appeared in this fence's PREVIOUS
   content, append at the end of the new fence content:
   ```
   <!-- graphify:removed: <label> (last seen: YYYY-MM-DD) -->
   ```
   Also carry over any `graphify:removed` lines already present in the old fence content. Cap at 10
   tombstones per fence — drop the oldest beyond that. (Tombstones tell the next reader a module
   was deliberately deleted, not accidentally lost from the docs.)
5. Write the updated file back

### Step S5: Report Changes

Print a summary of what changed:

```
✅ /forge-contextmap sync complete!

Graph rebuilt: [N] nodes, [M] edges
Docs refreshed:
  doc/architecture.md   — [N] sections updated
  doc/domain-model.md   — [N] sections updated
  [...]

Changelog draft (doc/changelog.txt):
  [+N added, -M removed] structural changes drafted — review/edit the auto-draft block
Tombstones: [N] removed modules marked <!-- graphify:removed --> in doc fences

User content: untouched (all content outside <!-- graphify:auto --> preserved)
```

---

## DESIGN BRIEF GENERATION *(shared by New / Existing / Migration modes)*

Goal: turn one vibe answer into a COMPLETE, CONCRETE design system — so every future UI task uses
the same values. No randomness: real hex codes, a real font name, a fixed scale.

From the user's vibe/brand answer, generate ALL of:

- **Color tokens** — at minimum Primary, Surface, Text, Muted, Border, Success, Error — each a
  concrete hex value that fits the vibe. If the user gave brand colors, build around them. Ensure
  Text-on-Surface contrast is readable (aim WCAG AA).
- **Typography** — one font family (prefer widely available: Inter, system-ui stack, Roboto, SF
  Pro, or the user's brand font), a numeric scale (e.g. 32/24/18/16/14), and weights.
- **Spacing & radius** — a base unit (e.g. 4px) with an allowed set, and radius values.
- **Reusable components** — the starter inventory appropriate to the app type (typically: Button
  primary/secondary/ghost, Card, Input, Modal, Toast; add List Item, Avatar, Badge, etc. as the
  features demand).
- **Screen style guidance** — one line each for list screens, detail screens, forms, empty states.

Present the whole set in a compact block and ask for approval:

```
Proposed design system (from "[their vibe answer]"):

Colors:   Primary #.. · Surface #.. · Text #.. · Muted #.. · Border #.. · Success #.. · Error #..
Type:     [font] — 32/24/18/16/14 · weights 400/500/700
Spacing:  4px base (4/8/12/16/24/32) · Radius 8px
Components: Button (1°/2°/ghost), Card, Input, Modal, Toast[, ...]

Approve, or tell me what to change (any value is adjustable).
```

Apply corrections, then write the approved values into `doc/design-brief.md`. If the codebase
already defines a theme/token file, EXTRACT from it instead of generating — the code is the truth;
present what you found for confirmation.

---

## FENCE FORMAT REFERENCE

```
<!-- graphify:auto start:QUALIFIED_KEY:SECTION -->
Content here is managed by /forge-contextmap sync.
DO NOT manually edit — changes will be overwritten.
<!-- graphify:auto end:QUALIFIED_KEY:SECTION -->
```

**Key format**: `project:section_name` for project-level summaries.

v2 fence keys: `project:claude-summary`, `project:architecture`, `project:domain-model`,
`project:api-contract`, `project:solution-structure`, `project:coding-standard`,
`project:security`, `project:app-flow`, `project:design-brief`, `project:backend-schema`.

Example:
- `<!-- graphify:auto start:project:domain-model -->`

**Rules**:
- Content inside fences: overwritten on every sync
- Content outside fences: never touched
- `doc/prd.md` and `doc/task-list.md` have NO fences — 100% user-owned (orchestrate's
  status ticks are the sole exception for task-list.md)
