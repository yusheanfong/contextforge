## EXISTING PROJECT MODE

*Trigger: source files found, no graph.json yet, no old-format docs*

> **Portability contract.** Every command in this file must run identically on macOS, Linux, and
> Windows. **No heredocs, no `cp`/`mv`/`rm`, no `mkdir -p`/`chmod`, no `2>/dev/null`, no `||`
> chaining.** Multi-line Python goes into a file written with the **Write tool** and is run as
> `[PYTHON_CMD] <script>.py`; file copies use the **Read + Write tools**.

### Step E1: Python Prerequisite Check

Use the Bash tool to check Python version. Run these as **two separate calls** — `||` is not
available in PowerShell 5.1, and the first one that succeeds wins:

```bash
python --version
```
```bash
python3 --version
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
graphify --version
```

If the command is not found, try `[PYTHON_CMD] -m graphify --version`. If neither resolves, install it. Choose the target by environment so we never break a managed env nor pollute the global one — `--user` is invalid inside a virtualenv, so only use it when no env is active:

- **If a virtualenv/conda/poetry env is active** — detect via `$VIRTUAL_ENV` being set, or `[PYTHON_CMD] -c "import sys; print(sys.prefix != sys.base_prefix)"` printing `True` — install into it normally:
  ```bash
  [PYTHON_CMD] -m pip install graphifyy
  ```
- **Otherwise** (no managed env active), use `--user` to avoid touching global/system site-packages and to avoid needing sudo:
  ```bash
  [PYTHON_CMD] -m pip install --user graphifyy
  ```

Note: the PyPI package name is `graphifyy` (double y) — this is correct and verified, NOT a typo. Do not change it to `graphify`. The installed **command** is `graphify` (single y). After pip install, run the post-install setup:
```bash
graphify install
```

If `graphify install` fails (command not found), try:
```bash
[PYTHON_CMD] -m graphify install
```

### Step E2.5: Resolve the Graphify CLI

Follow the `graphify-cli` shared block in `references/sync.md` Step S1.5 — it
probes the installed CLI once and caches `build=` / `update=` commands to
`graphify-out/.graphify_cli`. Do not hardcode an invocation here; graphify's CLI is verb-based
(`graphify update <path>`) and its flags differ across versions.

### Step E3: Analyze Codebase

Run the `build=` command resolved in E2.5.

This will take time depending on codebase size. Wait for it to complete. It produces:
- `graphify-out/graph.json` — the knowledge graph
- `graphify-out/GRAPH_REPORT.md` — human-readable summary

If it fails, print the error and ask the user to check their Graphify installation.

**Scope the corpus with `.graphifyignore` before the first build.** Graphify honors a
`.graphifyignore` file at the repo root using gitignore syntax. Vendored code, generated bridges,
build output, and platform scaffolding otherwise land in the graph, inflate every doc fence, and
swamp the sync diff with churn that has nothing to do with your source. If the repo has directories
like `gauge_reader_bridge/`, `.android/`, `build/`, `node_modules/`, or `Pods/`, list them there
first — it is far cheaper than pruning them out afterwards. Check whether one exists; if not,
propose the entries you'd add and let the user confirm before creating it.

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

**If `[HAS_UI]`:** also ask the design question now (same as `references/new-project.md` N1 Question 4): "Describe the look/vibe you want (or brand colors/fonts if you have them)." Then generate the concrete design system per **`references/design-brief-generation.md`** and get approval. Store as `[DESIGN_SYSTEM]`. If the codebase already has an obvious design system (theme file, tokens), extract from THAT first and present it — don't invent a competing one.

### Step E6: Check for Existing Docs

Check if `doc/` folder exists with existing doc files.

- If docs exist AND `CLAUDE.md` has the `<!-- contextforge:format v2 -->` marker → v2 docs already present: preserve all user-owned content, write only into auto-fenced sections, and create any v2 docs that are missing.
- If docs exist WITHOUT the marker → you should be in MIGRATION MODE (SKILL.md STEP 1 check 3) — read `references/migration.md` and follow it.
- If no docs exist, create the full v2 set from scratch (Step E7).

### Step E7: Create/Update All Doc Files

For each doc file, use the fence format to mark graph-generated content. All content outside fences is either user-provided (preserved) or a placeholder prompt (for new files).

**CLAUDE.md** — create or update. Use the SAME v2 template as `references/doc-templates.md` File 1 (including the `<!-- contextforge:format v2 -->` marker, v2 Doc Navigation, Rules 1–7, and all four Coding Rules blocks), with these differences:

- `## Goal` — Ask user: "What's the goal of this project?" if not already known from E5. Use their answer.
- `## Tech Stack` — inferred from graph + user confirmation.
- Add this section after Tech Stack. The `### Notes` heading is not optional — it is where any
  hand-written description must live, because fence content is regenerated wholesale on every sync
  (see `references/fence-format.md`):
  ```
  ## Key Architecture (from Graphify)
  <!-- graphify:auto start:project:claude-summary -->
  **God nodes**: [top 5 god node labels]
  **Communities**: [community names/descriptions]
  **Entry points**: [main files]
  <!-- graphify:auto end:project:claude-summary -->

  ### Notes
  <!-- User-owned. Sync never touches anything below the end marker.
       Put architecture prose the graph can't derive here — what a module is FOR,
       why a decision was made, non-obvious data flow. -->
  [FILL IN]
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

**doc/design-brief.md** — ONLY if `[HAS_UI]`. Same template as `references/doc-templates.md` File 4, filled from `[DESIGN_SYSTEM]`, with the fence populated:

```markdown
<!-- graphify:auto start:project:design-brief -->
## Detected UI Components

[list UI-related nodes: screens, widgets, views, components, pages]
[note any design pattern detected, e.g. BLoC, Redux, MVVM]
[note any existing theme/token files detected]
<!-- graphify:auto end:project:design-brief -->
```

**doc/backend-schema.md** — ONLY if `[HAS_BACKEND]`. Same template as `references/doc-templates.md` File 5, with the fence populated:

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

If `doc/task-list.md` does not exist, create it in the engineering-plan format (see the task-list format in `references/doc-templates.md` — Goal, "How to work this list", `### Task N.M` blocks with `done` / `Depends on` / `Builds` / `Acceptance criteria`), with Phase 1 inferred from the codebase state.

If `doc/task-list.md` already exists, `/forge-contextmap` never rewrites its content — it's
user-authored. (Note: `/forge-orchestrate` is the one exception — on completion it ticks a task's
`[ ]` → `[x]`, or appends a `[x]` entry under `## Completed (orchestrated)` when the shipped feature
wasn't on the list. That is status bookkeeping, not a contextmap rewrite.)

Create `doc/changelog.txt`, `doc/progress.txt` only if they don't exist (templates in `references/doc-templates.md`, Files 12–13).

### Step E8: Install Post-Commit Hook

Same as Step N4 in `references/new-project.md`.

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
