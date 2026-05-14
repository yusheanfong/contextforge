# /contextmap — Graph-Powered Context Scaffold

Combines Graphify (code → knowledge graph) with ContextForge-style AI workflow docs.
Reads $ARGUMENTS to detect subcommand: `sync` or `--new`. Otherwise auto-detects mode.

---

## STEP 1: Detect Mode

Run these checks in order and jump to the matching section:

1. If `$ARGUMENTS` contains `sync` → go to **SYNC MODE**
2. If `$ARGUMENTS` contains `--new` → go to **NEW PROJECT MODE**
3. Check if `graphify-out/graph.json` exists in the current directory
   - If yes → go to **SYNC MODE**
4. Count source files: `.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.dart`, `.go`, `.cs`, `.java`, `.rb`, `.rs`, `.swift`, `.kt`, `.cpp`, `.c`, `.h`
   - If any found → go to **EXISTING PROJECT MODE**
5. Otherwise → go to **NEW PROJECT MODE**

---

## NEW PROJECT MODE

*Trigger: empty/near-empty repo, or `--new` flag*

### Step N1: Gather Project Info

Ask the user (one question at a time — wait for each answer before asking the next):

**Question 1:** "What's the final goal of this project? Describe what you're building and who it's for."

Wait for answer. Then **enhance it**: rephrase the raw description into a polished 2–4 sentence goal statement that is clear, specific, and captures scope + audience. Show the enhanced version to the user:

```
Here's how I'd state the goal:

[ENHANCED_GOAL]

Does this capture it accurately? (or tell me what to adjust)
```

Wait for confirmation or corrections. Store the approved version as `[GOAL]`.

**Question 2:** "What's the tech stack / language? (e.g. Flutter + Dart, React + TypeScript, ASP.NET Core, Django, etc.)"

Store answer as `[TECH_STACK]`.

### Step N2: Scaffold All Files

Use the Write tool to create each file below. Replace `[GOAL]` and `[TECH_STACK]` with the user's answers. Do not skip any file.

**File 1: `CLAUDE.md`**

```
# Project Context

## Goal
[GOAL]

## Tech Stack
[TECH_STACK]

## Doc Navigation
All project docs live in /doc:
- doc/architecture.md     — Tech stack, layers, design patterns
- doc/domain-model.md     — Entities, enums, business rules
- doc/api-contract.md     — API endpoints or service interfaces
- doc/solution-structure.md — Project folder layout
- doc/coding-standard.md  — Language and framework conventions
- doc/security.md         — Auth, roles, data protection rules
- doc/ui-guideline.md     — Layout and UX rules (skip if backend-only)
- doc/task-list.md        — Master task list (YOUR ONLY TODO SOURCE)
- doc/changelog.txt       — Change log
- doc/progress.txt        — Current status

## Graph Sync
Sections marked <!-- graphify:auto start:... --> are auto-populated by /contextmap sync.
Edit ONLY outside these markers — content inside is overwritten on sync.
When Graphify runs for the first time (after code exists), run /contextmap sync to populate.

## Rules
1. Read all /doc files before writing any code.
2. Implement ONLY the next incomplete task from doc/task-list.md.
3. Update doc/progress.txt after every completed task.
4. Update doc/changelog.txt after every change (format: Date | Change | Description).
5. Follow doc/solution-structure.md exactly — no structural changes.
```

**File 2: `doc/architecture.md`**

```
# Architecture — [GOAL summary]

## Tech Stack
[TECH_STACK]

<!-- graphify:auto start:project:architecture -->
_No graph data yet. Run /contextmap sync after adding source code to auto-populate this section._
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

**File 3: `doc/domain-model.md`**

```
# Domain Model

<!-- graphify:auto start:project:domain-model -->
_No graph data yet. Run /contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:domain-model -->

## Enums
[FILL IN: list enum names and values]

## Entity Invariants
[FILL IN: e.g. Order must belong to a Customer]

## Business Rules
[FILL IN: e.g. Cannot delete record if child records exist]
```

**File 4: `doc/api-contract.md`**

```
# API Contract

<!-- graphify:auto start:project:api-contract -->
_No graph data yet. Run /contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:api-contract -->

## Endpoints
[FILL IN: list endpoints grouped by resource]

## Notes
[FILL IN: e.g. JWT required on all endpoints except /auth]
```

**File 5: `doc/solution-structure.md`**

```
# Solution Structure

## Tech Stack
[TECH_STACK]

<!-- graphify:auto start:project:solution-structure -->
_No graph data yet. Run /contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:solution-structure -->

## Root Layout
[FILL IN: describe your folder structure]

## Layer Dependency Rules
[FILL IN: which layers depend on which]

## AI Instruction
Follow this structure exactly. Do not invent new layers or folders.
```

**File 6: `doc/coding-standard.md`**

```
# Coding Standards

## Language: [TECH_STACK]

<!-- graphify:auto start:project:coding-standard -->
_No graph data yet. Run /contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:coding-standard -->

## Rules
[FILL IN: e.g. Use async/await everywhere]
[FILL IN: e.g. Validate all inputs at boundary]
[FILL IN: e.g. No magic strings — use constants]
[FILL IN: e.g. Use dependency injection for all services]
```

**File 7: `doc/security.md`**

```
# Security Model

<!-- graphify:auto start:project:security -->
_No graph data yet. Run /contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:security -->

## Roles
[FILL IN: list roles, e.g. Admin / User / Viewer]

## Rules
[FILL IN: e.g. JWT required for all endpoints]
[FILL IN: e.g. All queries filtered by OwnerId]
[FILL IN: e.g. Soft delete only — no hard deletes]
```

**File 8: `doc/ui-guideline.md`**

```
# UI Guidelines

<!-- graphify:auto start:project:ui-guideline -->
_No graph data yet. Run /contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:ui-guideline -->

## Layout
[FILL IN: e.g. Left sidebar / Top nav / Main content area]

## Views
[FILL IN: e.g. List / Detail / Modal]

## UX Rules
[FILL IN: e.g. Confirm before destructive actions]
[FILL IN: e.g. Toast notifications for success/error]
```

**File 9: `doc/changelog.txt`**

```
# Changelog
# Format: Date | Change | Description

[TODAY'S DATE] | Project initialized | /contextmap new project setup
```

Replace `[TODAY'S DATE]` with today's actual date (YYYY-MM-DD).

**File 10: `doc/progress.txt`**

```
# Progress

## Current Status
Project initialized. No tasks completed yet.

## Summary
Waiting to begin Phase 1.
```

**File 11: `doc/Progress/Progress-N.txt`**

```
# Task Progress Template
# Copy this file and rename to Progress-1.txt, Progress-2.txt, etc.

## Task
[Task name from task-list.md]

## Status
[ ] In Progress  [ ] Completed

## Files Modified
-

## Notes
```

### Step N3: Generate Draft Task List

Based on `[GOAL]` and `[TECH_STACK]`, generate a realistic phased task list. Write it to `doc/task-list.md` and then present it to the user:

```
Here's the draft task list I created based on your goal. Review it and tell me what to add, remove, or change before we finalize.
```

The task list format:

```markdown
# Master Task List

## Goal
[GOAL]

## Phase 1 — Foundation
[ ] [Task derived from goal and stack]
[ ] [Task]
[ ] [Task]

## Phase 2 — Core Features
[ ] [Task]
[ ] [Task]
[ ] [Task]

## Phase 3 — Polish & Launch
[ ] [Task]
[ ] [Task]

## Notes
Update this file as requirements evolve. Only implement the NEXT incomplete task.
```

Wait for user feedback on the task list. Apply any changes they request. Once approved, confirm:

```
✅ /contextmap setup complete!

Files created:
  CLAUDE.md
  doc/architecture.md
  doc/domain-model.md
  doc/api-contract.md
  doc/solution-structure.md
  doc/coding-standard.md
  doc/security.md
  doc/ui-guideline.md
  doc/task-list.md        ← APPROVED, your master to-do
  doc/changelog.txt
  doc/progress.txt
  doc/Progress/Progress-N.txt (template)

Next steps:
1. Fill in [FILL IN: ...] placeholders in /doc files
2. Start coding — Claude will follow these docs automatically
3. Once you have source code, run /contextmap sync to populate graph sections
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
    $PYTHON -m graphify . --update 2>/dev/null &
fi
```

Make it executable:
```bash
chmod +x .git/hooks/post-commit
```

On Windows, the chmod may not apply — that's OK. The hook will still run in Git Bash or WSL.

---

## EXISTING PROJECT MODE

*Trigger: source files found, no graph.json yet*

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
  
  After fixing, run /contextmap again.
  ```

Determine which command works (`python` or `python3`) and use it for all subsequent commands. Store as `[PYTHON_CMD]`.

### Step E2: Install Graphify

Check if Graphify is installed:
```bash
[PYTHON_CMD] -m graphify --version 2>&1
```

If not found, install it:
```bash
[PYTHON_CMD] -m pip install graphifyy
```

Note: the PyPI package name is `graphifyy` (double y). After pip install, run the post-install setup:
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

---
Does this match your understanding of the project?
- Confirm if it's accurate
- Correct anything that's wrong
- Add anything important I missed (e.g. "The auth module isn't in source yet — we use Firebase Auth")

I'll incorporate your corrections before populating the doc files.
```

Wait for user response. Apply any corrections to your understanding.

### Step E6: Check for Existing Docs

Check if `doc/` folder exists with existing doc files. If docs already exist (partial or full), preserve their user-owned content — only write into auto-fenced sections.

If no docs exist, create all doc files from scratch (see Step E7).

### Step E7: Create/Update All Doc Files

For each doc file, use the fence format to mark graph-generated content. All content outside fences is either user-provided (preserved) or a placeholder prompt (for new files).

**CLAUDE.md** — create or update:

```markdown
# Project Context

## Goal
[Ask user: "What's the goal of this project?" if not already known. Use their answer here.]

## Tech Stack
[inferred from graph + user confirmation]

## Key Architecture (from Graphify)
<!-- graphify:auto start:project:claude-summary -->
**God nodes**: [top 5 god node labels]
**Communities**: [community names/descriptions]
**Entry points**: [main files]
<!-- graphify:auto end:project:claude-summary -->

## Doc Navigation
All project docs live in /doc:
- doc/architecture.md     — Tech stack, layers, design patterns
- doc/domain-model.md     — Entities, enums, business rules
- doc/api-contract.md     — API endpoints or service interfaces
- doc/solution-structure.md — Project folder layout
- doc/coding-standard.md  — Language and framework conventions
- doc/security.md         — Auth, roles, data protection rules
- doc/ui-guideline.md     — Layout and UX rules
- doc/task-list.md        — Master task list (YOUR ONLY TODO SOURCE)
- doc/changelog.txt       — Change log
- doc/progress.txt        — Current status

## Graph Sync Warning
Sections marked <!-- graphify:auto --> are overwritten on /contextmap sync.
Edit ONLY outside these markers.

## Rules
1. Read all /doc files before writing any code.
2. Implement ONLY the next incomplete task from doc/task-list.md.
3. Update doc/progress.txt after every completed task.
4. Update doc/changelog.txt after every change (format: Date | Change | Description).
5. Follow doc/solution-structure.md exactly.
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

**doc/ui-guideline.md**:

```markdown
# UI Guidelines

<!-- graphify:auto start:project:ui-guideline -->
## Detected UI Components

[list UI-related nodes: screens, widgets, views, components, pages]
[note any design pattern detected, e.g. BLoC, Redux, MVVM]
<!-- graphify:auto end:project:ui-guideline -->

## Layout
[FILL IN — user-owned]

## UX Rules
[FILL IN — user-owned]
```

If `doc/task-list.md` does not exist, create it:

```markdown
# Master Task List

## Phase 1 — [infer from codebase state]
[ ] [FILL IN]

## Notes
Update this file as requirements evolve. Implement ONLY the next incomplete task.
```

If `doc/task-list.md` already exists, DO NOT modify it — it's 100% user-owned.

Create `doc/changelog.txt`, `doc/progress.txt`, `doc/Progress/Progress-N.txt` only if they don't exist (same templates as NEW PROJECT MODE).

### Step E8: Install Post-Commit Hook

Same as Step N4 in New Project Mode.

### Step E9: Confirm

Print:

```
✅ /contextmap setup complete!

Graph analyzed: graphify-out/graph.json
Docs populated with graph data (sections marked <!-- graphify:auto -->)

Files created/updated:
  CLAUDE.md
  doc/architecture.md
  doc/domain-model.md
  doc/api-contract.md
  doc/solution-structure.md
  doc/coding-standard.md
  doc/security.md
  doc/ui-guideline.md
  [doc/task-list.md — created if it didn't exist]
  doc/changelog.txt
  doc/progress.txt

Graph sections will auto-update whenever you run /contextmap sync.
User-owned content (outside <!-- graphify:auto --> markers) is never touched.

Next: Fill in [FILL IN] placeholders in /doc files, then start coding.
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

### Step S4: Fence-Aware Merge

For each doc file that has `<!-- graphify:auto start:... -->` markers:

1. Read the entire file
2. Find all fence pairs: `<!-- graphify:auto start:KEY -->` ... `<!-- graphify:auto end:KEY -->`
3. For each fence pair:
   - Generate new content based on the updated graph data
   - Replace ONLY the content between the start and end markers
   - Keep the markers themselves intact
   - Keep all content outside the markers exactly as-is
4. Write the updated file back

If a fence key references a `source_file` path that no longer exists in `graph.json`:
- Replace the fence content with:
  ```
  <!-- graphify:removed: [node label] (last seen: [TODAY'S DATE]) -->
  ```
- Keep the outer fence markers so the user can see it and manually clean up

### Step S5: Report Changes

Print a summary of what changed:

```
✅ /contextmap sync complete!

Graph rebuilt: [N] nodes, [M] edges
Docs refreshed:
  doc/architecture.md   — [N] sections updated
  doc/domain-model.md   — [N] sections updated
  [...]

Removed modules:
  [list any modules that were in fences but no longer in graph]

User content: untouched (all content outside <!-- graphify:auto --> preserved)
```

---

## FENCE FORMAT REFERENCE

```
<!-- graphify:auto start:QUALIFIED_KEY:SECTION -->
Content here is managed by /contextmap sync.
DO NOT manually edit — changes will be overwritten.
<!-- graphify:auto end:QUALIFIED_KEY:SECTION -->
```

**Key format**: `source_file_path:section_name` for module-specific content, or `project:section_name` for project-level summaries.

Examples:
- `<!-- graphify:auto start:src/auth/auth_service:architecture -->`
- `<!-- graphify:auto start:project:domain-model -->`

**Rules**:
- Content inside fences: overwritten on every sync
- Content outside fences: never touched
- Fully-qualified keys prevent collision between modules with the same short name
- Removed module stubs: `<!-- graphify:removed: ModuleName (last seen: YYYY-MM-DD) -->`
