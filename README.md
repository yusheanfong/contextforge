# ContextForge

A three-command suite for Claude Code that shares one live knowledge graph: scaffold the workflow, execute features against it, and sweep it for bloat — all reading the same map of your codebase.

```
/contextmap   → scaffold docs + build the graph   (start here)
/orchestrate  → execute a feature against the graph
/audit        → sweep the repo for over-engineering
```

`/contextmap` is the entry point — the other two hard-stop until it has built the graph.

---

## What It Does

Claude Code starts every session cold. It reads `CLAUDE.md` (auto-loaded, ~600 tokens) to know what it's building. Generic placeholders mean generic decisions.

`/contextmap` solves both the scaffolding problem and the accuracy problem:

- **New project** — asks your goal, enhances it, scaffolds all docs, generates a phased task plan
- **Existing project** — analyzes your codebase, extracts real architecture knowledge, fills docs with actual data, waits for your validation
- **Sync** — keeps docs in sync as code evolves, never overwriting your manual content

---

## Install

**Global** (available in all projects):

```bash
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/contextmap.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/contextmap.md
```

**Project-level** (this project only):

```bash
mkdir -p .claude/commands
curl -o .claude/commands/contextmap.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/contextmap.md
```

**For existing project analysis**, you also need Python 3.10+. `/contextmap` installs Graphify automatically when it needs it.

---

## Pipeline — Which Command When

The three commands share one graph and run in a loop:

```
Setup (once)         /contextmap                → scaffold docs + build graph + post-commit hook
Per feature          /orchestrate <feature>     → branch + graph-scoped agents + CI gates + commits
                     → review the diff → merge the branch
                     /contextmap sync           → refresh the <!-- graphify:auto --> doc fences
Periodic / on-demand /audit                     → whole-repo bloat sweep (read-only)
```

**Sync comes *after* `/orchestrate`, not before.** `/orchestrate` updates `progress.txt`,
`changelog.txt`, and `task-list.md` — but it does **not** refresh the `graphify:auto` doc fences.
The post-commit hook rebuilds `graph.json` on every commit yet never writes docs. So run
`/contextmap sync` after merging to pull the fresh graph into your docs.

| Command | Use it when | Writes | Git |
|---------|-------------|--------|-----|
| `/contextmap` | Starting out, or refreshing docs after code changes (`sync`) | `doc/*`, `graph.json`, post-commit hook | never commits |
| `/orchestrate <feature>` | Building a new feature end-to-end | code + tests, `release-readiness.md`, progress/changelog | commits per subtask on a branch (opt out with `--no-commit`) |
| `/audit [path]` | Cleaning up accumulated bloat, before a refactor or release | nothing (report-only) | never commits |

**`/audit` vs `/orchestrate`'s built-in review:** `/orchestrate` reviews a *single fresh diff* at
commit time (its over-engineering gate). `/audit` sweeps *already-landed* code across the whole repo.
Reach for `/audit` on-demand when cruft has piled up.

---

## Usage

### New Project

In an empty or near-empty directory:

```
/contextmap
```

Or force new project mode:

```
/contextmap --new
```

Flow:
1. Asks: *"What's the final goal of this project?"*
2. Enhances your raw description into a polished 2–4 sentence goal — shows it to you for approval
3. Asks: *"What's the tech stack / language?"*
4. Scaffolds all doc files with your goal as the seed
5. Generates a realistic phased task list → shows it for your approval before saving
6. Installs a post-commit hook (silently rebuilds the graph on every commit)

No Python required for new projects. Graphify activates on first sync once you have code.

### Existing Project

In a project with source code (auto-detected):

```
/contextmap
```

Flow:
1. Checks Python ≥ 3.10 (fails with clear instructions if missing)
2. Installs Graphify if not present: `pip install graphifyy`
3. Runs `python -m graphify .` on your codebase
4. Reads the knowledge graph and presents a structured understanding:
   - Architecture pattern
   - Key components (god nodes — highest-degree concepts)
   - Main subsystems (community clusters)
   - Entry points
   - Public API surface
   - Dominant coding patterns
5. Asks: *"Does this match your understanding? What should I add or correct?"*
6. Applies your corrections, then populates all doc files
7. Installs the post-commit hook

### Sync

After code changes:

```
/contextmap sync
```

Flow:
1. Rebuilds the knowledge graph incrementally
2. Refreshes only the `<!-- graphify:auto -->` fenced sections in doc files
3. Preserves everything outside the fences — always
4. Auto-drafts structural `changelog.txt` entries from added/removed graph nodes (marked as editable drafts — keep, edit, or delete)
5. Prints a summary of what changed
6. Tombstones deleted modules: `<!-- graphify:removed: ModuleName (last seen: YYYY-MM-DD) -->`

---

## /orchestrate — Execute With the Map

`/contextmap` builds the map. `/orchestrate` uses it to *execute* a whole feature — a hierarchical
multi-agent pipeline (Coordinator → Worker → Critic → Synthesis) where each worker subagent sees
**only the docs and graph nodes its piece of the work touches** — context isolation derived from
`graphify-out/graph.json`, not hand-curated.

```
/orchestrate add input validation to the checkout endpoint
```

Flow:
1. **Hard stop** if no `graphify-out/graph.json` — run `/contextmap` first
2. **Clarify only if ambiguous** — asks about scope/criteria only for a genuinely unclear request; otherwise states its assumptions and runs hands-off
3. **Branch** — creates `feature/<slug>` and works there
4. **Decompose** the task into subtasks (goal, success criterion, dependencies, gate set)
5. **Graph slice per subtask** — reads `graph.json`, finds the touched nodes' community + neighborhood, and pulls only the matching `doc/*` files (plus the three universal docs)
6. **Dispatch workers** — 2+ independent subtasks each run in an isolated **git worktree** (real parallelism, overlapping edits surface as a visible merge conflict); lone/dependent subtasks run single-tree
7. **CI gates** — adaptive and honest: lint, tests, coverage, dependency-vuln, secrets (hard block), SAST, over-engineering. Runs what your project has, skips + *honestly reports* what it doesn't — never a fake green check
8. **Bounded review loop** — spec, then quality, then an over-engineering pass; max 3 iterations per subtask
9. **Commit per subtask** on the branch once its gates pass
10. **Synthesize** — clean summary, updates `progress.txt` / `changelog.txt`, ticks the task, and writes `doc/release-readiness.md` (the CD steps it can't run locally: deploy, canary, monitoring, rollback, DAST)

**This command commits** — a feature branch with per-subtask commits. It never tags and never merges
to `main`; **you own the release and the merge.** Pass **`--no-commit`** to run the full pipeline and
all gates with zero git writes — changes are left in the working tree for you to review and commit
yourself. `/orchestrate` never rebuilds the graph (that's `/contextmap sync` or the post-commit
hook), so commit or sync first if you have uncommitted structural changes.

**Install** (alongside contextmap):

```bash
curl -o ~/.claude/commands/orchestrate.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/orchestrate.md
```

Requires Python 3.10+ (to read the graph) and a project that has already run `/contextmap`.

---

## /audit — Sweep for Over-Engineering

`/orchestrate` reviews the diff it just wrote. `/audit` sweeps code that **already landed** —
the entire repo (or a scoped path) for accumulated bloat. **Report-only: never edits, never commits.**

```
/audit                    # whole repo
/audit src/checkout       # scope to a path prefix
```

Like `/orchestrate`, it uses `graphify-out/graph.json` to point at candidates instead of blind-reading
every file — then confirms each against the real source before flagging it (*the graph points, the
code decides*).

Flow:
1. **Hard stop** if no `graphify-out/graph.json` — run `/contextmap` first
2. **Graph scan** — surfaces three candidate buckets: orphans / near-dead nodes, duplicate labels across files, and god nodes (over-centralization)
3. **Source confirmation** — opens each candidate's file and evaluates it against the minimal-code ladder (dead? duplicate? stdlib does it? one-liner?), dropping anything the code proves legitimate
4. **Report** — a grouped delete/simplify list with a rung + rationale per finding, plus god-node structural notes framed as *decompose?* questions, and a summary count

Pass **`--graph-only`** to skip source confirmation (faster, less precise) — every finding is then
tagged `[unconfirmed]`. Tests and files needed for current behavior are never delete-listed.

**Install** (alongside contextmap):

```bash
curl -o ~/.claude/commands/audit.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/audit.md
```

Requires Python 3.10+ (to read the graph) and a project that has already run `/contextmap`.

---

## What Gets Created

```
your-project/
├── CLAUDE.md                    ← Auto-loaded every session (goal + architecture summary + rules)
├── .git/hooks/post-commit       ← Silently rebuilds graph on every commit
├── graphify-out/
│   ├── graph.json               ← Knowledge graph (Graphify-managed, don't edit)
│   ├── graph.prev.json          ← Last-sync snapshot (powers the auto-changelog diff)
│   └── GRAPH_REPORT.md          ← Human-readable graph summary
└── doc/
    ├── architecture.md          ← Tech stack, detected patterns, design rules
    ├── domain-model.md          ← Entities, relationships, business rules
    ├── api-contract.md          ← API surface, services, interfaces
    ├── solution-structure.md    ← Folder layout, dependency flow
    ├── coding-standard.md       ← Detected patterns + your standards
    ├── security.md              ← Auth components + your security rules
    ├── ui-guideline.md          ← UI components + your UX rules
    ├── task-list.md             ← Master task list (100% yours — never graph-populated)
    ├── changelog.txt            ← Updated after every change
    ├── progress.txt             ← Current status (kept short)
    ├── release-readiness.md     ← Written by /orchestrate — CD steps to run on your platform
    └── Progress/
        └── Progress-N.txt       ← Per-task detailed logs
```

---

## How It Works

### The Knowledge Graph (Graphify)

[Graphify](https://github.com/safishamsi/graphify) is an open-source Python tool that parses your codebase via tree-sitter AST analysis (29+ languages) and builds a queryable knowledge graph:

| Concept | What it means |
|---------|---------------|
| **Node** | A concept: function, class, module, interface, file |
| **Edge** | A relationship: calls, imports, extends, implements |
| **Edge confidence** | `EXTRACTED` (AST-confirmed) / `INFERRED` / `AMBIGUOUS` |
| **Community** | A cluster of related nodes (Leiden algorithm) |
| **God node** | A highest-degree concept — something everything else depends on |

`/contextmap` reads `graphify-out/graph.json` and maps this data into your doc files.

### The Fence Protocol

Graph-managed content is wrapped in HTML comment fences:

```markdown
<!-- graphify:auto start:src/auth/auth_service:architecture -->
Content here is extracted from your code.
Overwritten on every /contextmap sync.
<!-- graphify:auto end:src/auth/auth_service:architecture -->
```

**Everything outside the fences is yours.** Claude never touches it during sync.

- Add notes, rules, and context anywhere outside the fences — they survive every sync
- Graph sections refresh automatically when you run `/contextmap sync`
- Fence keys are fully-qualified paths (`source_file:section`) — no collision between modules with the same short name

### The Post-Commit Hook

`.git/hooks/post-commit` runs `python -m graphify . --update` silently in the background after every commit. This keeps `graphify-out/graph.json` current without blocking your workflow.

The hook only rebuilds the graph — it never writes to doc files. Run `/contextmap sync` explicitly when you want docs refreshed.

### Session Flow

```
You write code → git commit
  └─→ post-commit hook: graphify --update (background, silent)

/contextmap sync
  └─→ reads updated graph.json
  └─→ refreshes <!-- graphify:auto --> sections
  └─→ drafts changelog entries from added/removed nodes (you review)
  └─→ your content outside fences: unchanged

Claude Code session start
  └─→ CLAUDE.md auto-loads (~600 tokens)
  └─→ Claude knows your architecture, key components, rules
  └─→ First message: already in context, no setup needed
```

---

## Why It Works

### Real data beats placeholders

Standard templates fill `architecture.md` with `[FILL IN: your tech stack]`. You do the work, Claude uses what you wrote — which may be incomplete or stale after a week of coding.

`/contextmap` extracts architecture from your actual code. It finds your central abstractions, maps module dependencies, and detects coding patterns. Docs reflect reality.

### Structured context = better decisions

Claude makes better decisions with accurate, structured context:
- Won't suggest patterns that contradict your existing architecture
- Won't invent schema fields that don't exist in your domain model
- Won't suggest refactors that break your layer rules

### Fenced sections prevent drift

Docs that require manual updates go stale. The fence protocol solves this: graph sections are always current, your manual sections are always preserved. Neither overwrites the other.

### God nodes = what matters most

Not all code is equally important. God nodes are the concepts with the most connections — what everything else depends on. These surface in `CLAUDE.md`, giving Claude immediate awareness of your architecture's center of gravity before you type anything.

---

## Keeping Claude on Track

`/contextmap` writes these rules into `CLAUDE.md`:

1. Before writing code, always read the universal docs (`architecture.md`, `solution-structure.md`, `coding-standard.md`), then read only the domain docs the task touches (decided from `task-list.md` + `GRAPH_REPORT.md`) — not every `/doc` file
2. Implement ONLY the next incomplete task from `task-list.md`
3. Update `progress.txt` after every completed task
4. Update `changelog.txt` after every change (`Date | Change | Description`)
5. Follow `solution-structure.md` exactly — no structural changes
6. Never invent schema, fields, or endpoints not defined in the docs

```
Session N:   auto-load CLAUDE.md → read task-list → implement → update progress + changelog → commit
Session N+1: fresh session → same auto-load → continue from progress.txt
```

---

## Requirements

- Claude Code (any version)
- Python 3.10+ — needed for existing-project analysis, `/contextmap sync`, and both `/orchestrate` and `/audit` (they read the graph). Not required for new-project scaffolding.

---

## License

MIT
