## EXISTING PROJECT MODE

*Trigger: source files found, no graph.json yet, no old-format docs*

> **Portability contract.** Every command in this file must run identically on macOS, Linux, and
> Windows. **No heredocs, no `cp`/`mv`/`rm`, no `mkdir -p`/`chmod`, no `2>/dev/null`, no `||`
> chaining.** Multi-line Python goes into a file written with the **Write tool** and is run as
> `[PYTHON_CMD] <script>.py`; file copies use the **Read + Write tools**.

### Step E1: Python Prerequisite Check — resolves the **bootstrap** interpreter

There are **two different interpreters** in this skill; do not merge them.

| | `[BOOTSTRAP_PY]` (this step) | `[PYTHON_CMD]` (Step E2.6 onward) |
|---|---|---|
| Needs | any Python **≥ 3.10** | Python **with networkx** |
| Used for | `pip install graphifyy` in E2 | every `.forge_*.py` script |
| Resolved | before graphify exists | after E2 confirms graphify exists |

This step resolves **`[BOOTSTRAP_PY]` only**. It cannot use the `python-cmd` ladder in
`references/sync.md` S1, because that ladder verifies `import networkx` — which can only pass on an
interpreter that *already has graphify installed*. Using it here would make E2's install branch
unreachable on the machine E2 exists to serve.

Use the Bash tool. Run each as a **separate call** — `||` is not available in PowerShell 5.1. **Do
not stop at the first command that answers**; a box can have a `python3` that is too old *and* a
newer interpreter one command away. Work down the list and keep the first result that is **≥ 3.10**:

```bash
python --version
```
```bash
python3 --version
```
```bash
python3.13 --version
```
```bash
python3.12 --version
```
```bash
python3.11 --version
```

If none of those is ≥ 3.10, try graphify's own tool venv before giving up — if graphify is already
installed via uv or pipx, its interpreter is ≥ 3.10 by construction:

```bash
uv tool dir
```
```bash
pipx environment --value PIPX_LOCAL_VENVS
```

and test `<that dir>/graphifyy/bin/python --version` (macOS/Linux) or
`<that dir>\graphifyy\Scripts\python.exe --version` (Windows). A path that does not exist just
errors — harmless, move on.

Only if **every** candidate above is missing or below 3.10, print this exact error and STOP:
  ```
  ❌ Graphify requires Python 3.10 or higher.

  Found: [version found, or "Python not found"]

  Fix options:
  - Install Python 3.10+: https://www.python.org/downloads/
  - On Windows, ensure 'python' or 'python3' is on your PATH

  After fixing, run /forge-contextmap again.
  ```

Store the winner as `[BOOTSTRAP_PY]` and report which one you settled on.

> **Why the list, not just two commands.** Stock macOS ships `python3` 3.9 and **no** `python` at
> all. Stopping at the first command that answers meant printing the 3.10 error on a box that had a
> perfectly good 3.10+ interpreter — `/forge-contextmap` was simply unusable there. Windows never
> surfaced this because its `python` was already 3.10+.

### Step E2: Install Graphify

Check if Graphify is installed:
```bash
graphify --version
```

If the command is not found, try `[BOOTSTRAP_PY] -m graphify --version`. If neither resolves, install it. Choose the target by environment so we never break a managed env nor pollute the global one — `--user` is invalid inside a virtualenv, so only use it when no env is active:

- **If a virtualenv/conda/poetry env is active** — detect via `$VIRTUAL_ENV` being set, or `[BOOTSTRAP_PY] -c "import sys; print(sys.prefix != sys.base_prefix)"` printing `True` — install into it normally:
  ```bash
  [BOOTSTRAP_PY] -m pip install graphifyy
  ```
- **Otherwise** (no managed env active), use `--user` to avoid touching global/system site-packages and to avoid needing sudo:
  ```bash
  [BOOTSTRAP_PY] -m pip install --user graphifyy
  ```

Note: the PyPI package name is `graphifyy` (double y) — this is correct and verified, NOT a typo. Do not change it to `graphify`. The installed **command** is `graphify` (single y). After pip install, run the post-install setup:
```bash
graphify install
```

If `graphify install` fails (command not found), try:
```bash
[BOOTSTRAP_PY] -m graphify install
```

### Step E2.5: Resolve the Graphify CLI

Follow the `graphify-cli` shared block in `references/sync.md` Step S1.5 — it
probes the installed CLI once and caches `build=` / `update=` commands to
`graphify-out/.graphify_cli`. Do not hardcode an invocation here; graphify's CLI is verb-based
(`graphify update <path>`) and its flags differ across versions.

### Step E2.6: Resolve `[PYTHON_CMD]` (the verified interpreter)

Graphify is now installed, so the `python-cmd` ladder can run. Follow the `python-cmd` shared block
in `references/sync.md` Step S1 and store the result as `[PYTHON_CMD]`. Every `.forge_*.py` script
below uses it — **not `[BOOTSTRAP_PY]`**, which is only guaranteed to be ≥ 3.10 and may have no
networkx.

### Step E3: Analyze Codebase

Three commands, in this order:

1. The `build=` command resolved in E2.5 — full AST extraction.
2. The prune script from `references/sync.md` Step S2.5 (write it, run it, delete it — do not
   duplicate the script here).
3. **`graphify cluster-only .`** — append **`--no-label`** when no LLM backend is configured (the
   same condition that put `--code-only` on `build=`). This is what writes `GRAPH_REPORT.md`.

This will take time depending on codebase size. Wait for each to complete. Together they produce:
- `graphify-out/graph.json` — the knowledge graph, code-only
- `graphify-out/GRAPH_REPORT.md` — human-readable summary

If any step fails, print the error and ask the user to check their Graphify installation.

> **`build=` alone does not write `GRAPH_REPORT.md`.** `graphify extract --code-only` finishes with
> `next: run 'graphify cluster-only <path>' to generate GRAPH_REPORT.md` — and stops. E4 below and
> CLAUDE.md Rule 1 both tell readers to open that file, so without step 3 the very first session
> after scaffolding hits a missing file.
>
> **`graphify update` is NOT a substitute — do not use it here.** It short-circuits on an unchanged
> graph: `[graphify watch] No code-graph topology changes detected; outputs left untouched.` Straight
> after a fresh `extract` nothing *has* changed, so it writes no report. It only appears to work on a
> repo that already contains markdown, because then `update` adds doc nodes, topology changes, and
> the report gets written as a side effect — which is the contamination this skill exists to prevent.
> On a pure-code repo it silently does nothing.
>
> **`--no-label` is what keeps `cluster-only` safe.** Bare `cluster-only` runs LLM community naming
> and stalls on a box with no API key. With `--no-label` it re-clusters locally and leaves
> `Community N` placeholders — correct, just unnamed.
>
> **Prune runs before cluster-only**, so the report and the community ids describe the same graph the
> doc fences will.

**Scope the corpus with `.graphifyignore` before the first build.** Graphify honors a
`.graphifyignore` file at the repo root using gitignore syntax. Vendored code, generated bridges,
build output, and platform scaffolding otherwise land in the graph, inflate every doc fence, and
swamp the sync diff with churn that has nothing to do with your source. If the repo has directories
like `gauge_reader_bridge/`, `.android/`, `build/`, `node_modules/`, or `Pods/`, list them there
first — it is far cheaper than pruning them out afterwards. Check whether one exists; if not,
propose the entries you'd add and let the user confirm before creating it.

### Step E4: Parse Graph and Extract Understanding

**Do not read `graphify-out/graph.json` with the Read tool** — it is ~424 KB for a 57-file repo and
grows from there. Write this to `graphify-out/.forge_parse.py` with the **Write tool**, run it, read
its output, then delete it:

```python
"""Summarize graph.json for the E5 understanding. Stdlib only, no networkx."""
import json
from collections import Counter, defaultdict
from pathlib import Path

data = json.loads(Path("graphify-out/graph.json").read_text(encoding="utf-8"))
nodes = {n["id"]: n for n in data.get("nodes", [])}
links = data.get("links", [])

ENTRY_HINTS = ("main", "index", "app", "program", "entrypoint")
API_HINTS = ("api", "service", "controller", "handler", "endpoint", "route")
UI_HINTS = ("screen", "widget", "view", "component", "page", "activity", "fragment")
DB_HINTS = ("model", "entity", "repository", "schema", "migration", "dao", "orm")


def src_of(n):
    s = n.get("source_file")
    return s.replace("\\", "/") if s else None


def label_of(n):
    return n.get("label", str(n.get("id")))


deg = Counter()
for e in links:
    deg[e.get("source")] += 1
    deg[e.get("target")] += 1

print("== NODE COUNT ==", len(nodes), "edges", len(links))
print("== FILE_TYPE ==", dict(Counter(n.get("file_type") for n in nodes.values())))

print("== GOD NODES (top 10) ==")
for nid, d in deg.most_common(10):
    n = nodes.get(nid)
    if n:
        print(f"  {label_of(n)} | {src_of(n)} | degree={d}")

print("== COMMUNITIES ==")
comms = defaultdict(list)
for n in nodes.values():
    comms[n.get("community")].append(n)
for cid, members in sorted(comms.items(), key=lambda kv: -len(kv[1]))[:15]:
    files = Counter(src_of(m) for m in members if src_of(m))
    top = ", ".join(f for f, _ in files.most_common(3))
    names = ", ".join(label_of(m) for m in members[:5])
    print(f"  c{cid} ({len(members)} nodes) files: {top} | {names}")

print("== ENTRY POINTS ==")
entries = {src_of(n) for n in nodes.values()
           if src_of(n) and any(h in Path(src_of(n)).stem.lower() for h in ENTRY_HINTS)}
for f in sorted(entries):
    print("  ", f)

print("== API SURFACE ==")
api = sorted({f"{label_of(n)} | {src_of(n)}" for n in nodes.values()
              if any(h in label_of(n).lower() for h in API_HINTS)
              or (src_of(n) and any(h in src_of(n).lower() for h in API_HINTS))})
for a in api[:25]:
    print("  ", a)

print("== RELATIONS (high-confidence patterns) ==")
rel = Counter((e.get("relation"), e.get("confidence")) for e in links)
for (r, c), k in rel.most_common(12):
    print(f"  {r} [{c}] x{k}")

blob = " ".join(f"{label_of(n)} {src_of(n) or ''}" for n in nodes.values()).lower()
print("== HAS_UI ==", any(h in blob for h in UI_HINTS))
print("== HAS_BACKEND ==", any(h in blob for h in DB_HINTS))
```

```bash
[PYTHON_CMD] graphify-out/.forge_parse.py
```

Read that output and derive:
- **God nodes** — the top-degree list.
- **Communities** — name each from its node labels and dominant `source_file` paths.
- **Entry points**, **API surface**, **dominant patterns** (from the relation/confidence census).
- **UI presence** (`[HAS_UI]`) and **Backend presence** (`[HAS_BACKEND]`) — the script's booleans are
  a hint, not a verdict; sanity-check them against the community and file listings.

Note: use the **`file_type`** field, not `type`. Graphify 0.9.26 leaves `type` unset on every node
(`Counter({None: 397})` on a real repo), so any logic keyed on `type` silently sees nothing.

Delete `graphify-out/.forge_parse.py` once you have the output.

Also read `graphify-out/GRAPH_REPORT.md` for the summary Graphify generated (E3's `update` step
writes it; if it is somehow absent, carry on without it rather than stopping).

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

**CLAUDE.md** — create or update. Use the SAME v2 template as `references/doc-templates.md` File 1 (including the `<!-- contextforge:format v2 -->` marker, v2 Doc Navigation, Rules 1–8, and all four Coding Rules blocks), with these differences:

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
