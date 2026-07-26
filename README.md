# ContextForge

Four Claude Code skills that share one live knowledge graph: scaffold the workflow, execute features against it, sweep it for bloat, and diagnose what breaks — all reading the same map of your codebase.

```
/forge-contextmap   → scaffold docs + build the graph   (start here)
/forge-orchestrate  → execute a feature against the graph
/forge-audit        → sweep the repo for over-engineering
/forge-diagnose     → find root cause, change nothing, hand off
```

`/forge-contextmap` is the entry point — `/forge-orchestrate` and `/forge-audit` hard-stop until it has built the graph. `/forge-diagnose` doesn't: it uses the graph when present and greps when not, so a bug is never blocked behind a graph build.

---

## What It Does

Claude Code starts every session cold. It reads `CLAUDE.md` (auto-loaded, ~600 tokens) to know what it's building. Generic placeholders mean generic decisions.

ContextForge replaces those placeholders with architecture extracted from your actual code, then keeps four workflows — scaffold, execute, audit, diagnose — reading from that same extracted map instead of from whatever happens to be in the session's context window.

---

## Install

All four are packaged as **skills** — a thin `SKILL.md` plus `references/` files loaded only on the
branch that needs them (progressive disclosure: a single-tree run never reads the worktree
instructions, a project with no task list never reads the tick rules).

There is **no `.claude/commands/` directory** — a skill is already a slash command. `user-invocable`
defaults to `true`, so each one appears in the `/` menu (as `/contextforge:forge-<name>` via the
plugin, or `/forge-<name>` if you copy the files). The `description` also lets Claude reach for it on
intent: "sweep this repo for dead code" gets you `/forge-audit` without typing the slash.

**Coming from a pre-skill install?** These skills do not upgrade your old
`~/.claude/commands/forge-*.md` files — install from the block below, then delete the stale
commands: see [Upgrading from a pre-skill install](#upgrading-from-a-pre-skill-install).

### Install as a plugin (recommended)

This repo is also a Claude Code **plugin marketplace**. Two commands in your terminal:

```bash
claude plugin marketplace add https://github.com/yusheanfong/contextforge.git
claude plugin install contextforge@contextforge
```

Or the same two inside a Claude Code session, as `/plugin marketplace add …` and
`/plugin install contextforge@contextforge`.

All four skills arrive together, in every project, and stay in sync with this repo — see
[Updating](#updating).

The shorthand `yusheanfong/contextforge` also works in place of the URL, but it clones over **SSH**
by default, so it fails without a GitHub SSH key. Use the full HTTPS URL above, or set
`CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1` before using the shorthand.

**Names are namespaced.** Plugin skills carry the plugin name, so they appear as:

```
/contextforge:forge-contextmap
/contextforge:forge-orchestrate
/contextforge:forge-audit
/contextforge:forge-diagnose
```

Bare `/forge-audit` won't autocomplete after a plugin install. Intent still works — "sweep this repo
for dead code" reaches `/forge-audit` without typing anything. Where this README and the skills
themselves say "run `/forge-contextmap` first", they mean `/contextforge:forge-contextmap`.

### Manual install (copy the skills)

No plugin system involved — the four skill directories are plain files.

**Global** — available in all projects:

```bash
git clone --depth 1 https://github.com/yusheanfong/contextforge /tmp/contextforge
mkdir -p ~/.claude/skills
cp -R /tmp/contextforge/.claude/skills/. ~/.claude/skills/
rm -rf /tmp/contextforge
```

**Project-level** — this project only:

```bash
git clone --depth 1 https://github.com/yusheanfong/contextforge /tmp/contextforge
mkdir -p .claude/skills
cp -R /tmp/contextforge/.claude/skills/. .claude/skills/
rm -rf /tmp/contextforge
```

> Pick one or the other. For skills, **personal (`~/.claude/skills/`) overrides project
> (`.claude/skills/`)** when both define the same name — the reverse of how `settings.json`
> precedence works. Install globally *or* per-project, not both, or the project copy is dead weight.

Copied skills keep their bare `/forge-*` names — no `contextforge:` prefix.

### Install one only

Manual install only; the plugin ships all four. Supported — every shared block is duplicated inside
each skill dir precisely so one skill works standalone (see [For Maintainers](#for-maintainers)):

```bash
git clone --depth 1 https://github.com/yusheanfong/contextforge /tmp/contextforge
mkdir -p ~/.claude/skills
cp -R /tmp/contextforge/.claude/skills/forge-orchestrate ~/.claude/skills/
rm -rf /tmp/contextforge
```

Start with **`forge-contextmap`** regardless — the others depend on the graph it builds.

### Updating

**Plugin install:** the plugin declares no pinned `version`, so every commit pushed here counts as a
new version and Claude Code's background refresh picks improvements up on its own. To pull them
immediately:

```bash
claude plugin marketplace update contextforge
claude plugin update contextforge@contextforge
```

Use the qualified `contextforge@contextforge` for the second command. The bare plugin name fails
with `Failed to update plugin "contextforge": Plugin "contextforge" not found`, even though
`claude plugin details contextforge` resolves it. Restart Claude Code to apply the update.

The in-session equivalents are `/plugin marketplace update contextforge` and `/plugin update`.

**Manual install:** re-run the install block; `cp -R` overwrites in place.

### Switching from a manual install to the plugin

Delete the copied skills after installing the plugin, or the same four show up twice — once bare,
once as `contextforge:*`:

```bash
rm -rf ~/.claude/skills/forge-*      # global copy
rm -rf .claude/skills/forge-*        # per-project copy, run inside that project
```

### Upgrading from a pre-skill install

Older versions shipped all four as single files in `~/.claude/commands/` — `forge-contextmap.md`,
`forge-orchestrate.md`, `forge-audit.md`, `forge-diagnose.md`. **Nothing auto-upgrades.** Neither the
plugin nor a copied skills tree removes them, and stale copies of the same `/forge-*` names are
confusing at best.

Install by either method above, then clear the old commands out:

```bash
rm -f ~/.claude/commands/forge-*.md
```

Installs from before the `forge-` rename also have `contextmap.md`, `orchestrate.md`, and `audit.md`
— remove those by name only. A bare glob would hit commands you wrote yourself.

### Uninstall

**Plugin install:**

```bash
claude plugin uninstall contextforge
claude plugin marketplace remove contextforge
```

**Manual install:**

```bash
rm -rf ~/.claude/skills/forge-*
rm -f ~/.claude/commands/forge-*.md   # only if you ever used a pre-skill install
```

### Requirements

- Claude Code. Any version for the manual install — the skills themselves use no version-gated
  features. The **plugin** install additionally needs a build with plugin support: run
  `claude plugin --help` and check that `marketplace` is listed. Verified on 2.1.220.
- Python 3.10+ — for existing-project analysis, `/forge-contextmap sync`, `/forge-orchestrate`, and
  `/forge-audit` (all read the graph). **Not** required for new-project scaffolding.
  `/forge-contextmap` installs Graphify automatically when it needs it.
- `/forge-orchestrate` and `/forge-audit` additionally need a project that has already run
  `/forge-contextmap` — they read its graph and never build it themselves.

---

## The Loop

The four skills share one graph and run in a loop:

```
Setup (once)          /forge-contextmap                   → scaffold docs + build graph + post-commit hook
                      then read doc/task-list.md          → your build order, phase by phase

Per task (the loop)   /forge-orchestrate                  → no args: takes the next eligible task off
                                                            the plan and uses ITS acceptance criteria
                      → review the diff → merge the branch
                      /forge-contextmap sync              → refresh doc fences + print the bloat signal

Ad-hoc work           /forge-orchestrate <feature>        → same pipeline, spec typed by you

When something breaks /forge-diagnose <issue>             → root cause (no code changes) + handoff file
                      /forge-orchestrate doc/diagnosis-*  → execute that handoff, ambiguity scan skipped

Periodic / on-demand  /forge-audit                        → whole-repo bloat sweep (read-only)
```

| Command | Use it when | Writes | Git |
|---------|-------------|--------|-----|
| `/forge-contextmap` | Starting out, or refreshing docs after code changes (`sync`) | `doc/*`, `graph.json`, post-commit hook | never commits |
| `/forge-orchestrate <feature>` | Building a new feature end-to-end | code + tests, `release-readiness.md`, progress/changelog | commits per subtask on a branch (opt out with `--no-commit`) |
| `/forge-audit [path]` | Cleaning up accumulated bloat, before a refactor or release | nothing (report-only) | never commits |
| `/forge-diagnose <issue>` | Something is broken and the fix will run in another session | `doc/diagnosis-<slug>.md` only | never commits |

**Bare `/forge-orchestrate` is the normal way to run it.** With no arguments it reads
`doc/task-list.md`, picks the first unchecked `### Task N.M` whose `Depends on` are all done, and
uses that task's own acceptance criteria as its success criteria — you don't retype the spec, and
the plan stays the single source of truth for what's next. Type a description only for work that
isn't on the plan.

**Sync comes *after* `/forge-orchestrate`, not before.** `/forge-orchestrate` updates `progress.txt`,
`changelog.txt`, and `task-list.md` — but it does **not** refresh the `graphify:auto` doc fences.
The post-commit hook rebuilds `graph.json` on every commit yet never writes docs. So run
`/forge-contextmap sync` after merging to pull the fresh graph into your docs.

**Diagnosis feeds execution directly.** `/forge-diagnose` writes `doc/diagnosis-<slug>.md`; pass that
*path* to `/forge-orchestrate` rather than re-typing the problem — see
[Then hand it straight to the pipeline](#forge-diagnose--find-the-root-cause-hand-it-off).

**Sync surfaces the audit trigger.** Since sync already has the graph loaded, it prints orphan /
duplicate-label / god-node / dead-file counts. Those are pointers, not findings — `/forge-audit` is
what confirms them against real source.

**`/forge-audit` vs `/forge-orchestrate`'s built-in review:** `/forge-orchestrate` reviews a *single fresh diff* at
commit time (its over-engineering gate). `/forge-audit` sweeps *already-landed* code across the whole repo.
Reach for `/forge-audit` on-demand when cruft has piled up.

---

## The Four Skills

### /forge-contextmap — Scaffold the Docs, Build the Map

`/forge-contextmap` solves both the scaffolding problem and the accuracy problem:

- **New project** — asks your goal + features + design vibe, enhances them, scaffolds the full v2 doc set (PRD, app flow, design brief, backend schema), generates a phased engineering plan
- **Existing project** — analyzes your codebase, extracts real architecture knowledge, fills docs with actual data, waits for your validation
- **Migration** — detects an old-format (v1) `doc/` set and upgrades it to v2 in place, preserving every user-authored line verbatim
- **Sync** — keeps docs in sync as code evolves, never overwriting your manual content

#### New project

In an empty or near-empty directory:

```
/forge-contextmap
```

Or force new project mode:

```
/forge-contextmap --new
```

Flow:
1. Asks: *"What's the final goal of this project?"*
2. Enhances your raw description into a polished 2–4 sentence goal — shows it to you for approval
3. Asks: *"What's the tech stack / language?"* (infers whether you have a UI and/or backend)
4. Asks: *"List the core features"* → structures them into a prioritized PRD (`F1 (P0)`, `F2 (P1)`, …) for your approval
5. If there's a UI, asks one design question (*"describe the look/vibe or brand colors"*) → generates a **complete concrete design system** — hex color tokens, font + type scale, spacing/radius, reusable component inventory — for your approval. No randomness: every future screen uses these exact values.
6. Scaffolds all doc files (PRD, app flow, design brief, backend schema if applicable, + the architecture/domain/api/structure/standards/security set)
7. Generates a phased **engineering plan** (`doc/task-list.md`): small tasks with build order, `Depends on`, a PRD feature link, and 4 acceptance criteria each (works as expected / no errors / meets requirement / test added) → shows it for approval before saving
8. Installs a post-commit hook (silently rebuilds the graph on every commit)

No Python required for new projects. Graphify activates on first sync once you have code.

#### Existing project

In a project with source code (auto-detected):

```
/forge-contextmap
```

Flow:
1. Checks Python ≥ 3.10 (fails with clear instructions if missing)
2. Installs Graphify if not present: `pip install graphifyy`
3. Probes the installed graphify CLI once (caching it to `graphify-out/.graphify_cli`), then runs it on your codebase
   - The CLI is verb-based (`graphify update .`) and its flags differ across versions, so the invocation is resolved at runtime, never hardcoded
4. Reads the knowledge graph and presents a structured understanding:
   - Architecture pattern
   - Key components (god nodes — highest-degree concepts)
   - Main subsystems (community clusters)
   - Entry points
   - Public API surface
   - Dominant coding patterns
5. Asks: *"Does this match your understanding? What should I add or correct?"* — including an inferred core-feature list that becomes your PRD
6. If UI code is detected: extracts your existing theme/tokens (or asks the vibe question) → concrete design brief
7. Applies your corrections, then populates all doc files (backend schema included when backend code is detected)
8. Installs the post-commit hook

#### Scope the corpus first: `.graphifyignore`

Graphify honors a `.graphifyignore` at the repo root, gitignore syntax. Vendored code, generated
bridges, build output, and platform scaffolding otherwise become graph nodes — they inflate every
doc fence and swamp the sync diff with churn unrelated to your source.

```
gauge_reader_bridge/
.android/
build/
node_modules/
Pods/
```

Cheaper to exclude up front than to prune afterwards. `/forge-contextmap` checks for this file
before the first build and proposes entries if it is missing.

#### Migration (old-format projects)

Already ran an older ContextForge on a project? Just run:

```
/forge-contextmap
```

It detects a v1 `doc/` set (no `<!-- contextforge:format v2 -->` marker in `CLAUDE.md`) and upgrades in place:

- Creates `prd.md` (features inferred + confirmed by you), `app-flow.md`, `backend-schema.md` (if backend)
- `design-brief.md` absorbs **all** of `ui-guideline.md` verbatim into a *UX Rules (migrated)* section — the old file is deleted only after you approve
- `task-list.md` converts to the engineering-plan format — every task line and checkbox state preserved verbatim; completed tasks are marked *migrated as completed*, and no acceptance criteria are fabricated as ticked
- `CLAUDE.md` gets the v2 navigation/rules + format marker; everything you wrote stays untouched

**Guarantee: no user-authored line is dropped or reworded — content is moved, never rewritten.**

#### Sync

After code changes:

```
/forge-contextmap sync
```

Flow:
1. Rebuilds the knowledge graph incrementally
2. Refreshes only the `<!-- graphify:auto -->` fenced sections in doc files
3. Preserves everything outside the fences — always
4. Auto-drafts structural `changelog.txt` entries from added/removed graph nodes (marked as editable drafts — keep, edit, or delete)
5. Prints a summary of what changed
6. Tombstones deleted modules: `<!-- graphify:removed: ModuleName (last seen: YYYY-MM-DD) -->`

---

### /forge-orchestrate — Execute With the Map

`/forge-contextmap` builds the map. `/forge-orchestrate` uses it to *execute* a whole feature — a hierarchical
multi-agent pipeline (Coordinator → Worker → Critic → Synthesis) where each worker subagent sees
**only the docs and graph nodes its piece of the work touches** — context isolation derived from
`graphify-out/graph.json`, not hand-curated.

```
/forge-orchestrate add input validation to the checkout endpoint
```

Flow:
1. **Hard stop** if no `graphify-out/graph.json` — run `/forge-contextmap` first
2. **Clarify only if ambiguous** — asks about scope/criteria only for a genuinely unclear request; otherwise states its assumptions and runs hands-off
3. **Branch** — creates `feature/<slug>` and works there
4. **Decompose** the task into subtasks (goal, success criterion, dependencies, gate set). Run with no arguments and it picks the next eligible task from the engineering plan — dependency-aware — and uses that task's own acceptance criteria as the success criteria
5. **Graph slice per subtask** — reads `graph.json`, finds the touched nodes' community + neighborhood, and pulls only the matching `doc/*` files (plus the universal docs: architecture, structure, standards, and the PRD as scope guard). UI subtasks get `design-brief.md` + `app-flow.md`; data/API subtasks get `backend-schema.md` when it exists
6. **Dispatch workers** — 2+ independent subtasks each run in an isolated **git worktree** (real parallelism, overlapping edits surface as a visible merge conflict); lone/dependent subtasks run single-tree
7. **CI gates** — adaptive and honest: lint, tests, coverage, dependency-vuln, secrets (hard block), SAST, over-engineering. Runs what your project has, skips + *honestly reports* what it doesn't — never a fake green check
8. **Bounded review loop** — spec (against the PRD feature the task `Builds:`), then quality, then an over-engineering pass, then **design compliance** on UI subtasks (every color/font/spacing in the diff must come from `design-brief.md` tokens — ad-hoc hex values fail review); max 3 iterations per subtask
9. **Commit per subtask** on the branch once its gates pass
10. **Synthesize** — clean summary, updates `progress.txt` / `changelog.txt`, ticks the task, and writes `doc/release-readiness.md` (the CD steps it can't run locally: deploy, canary, monitoring, rollback, DAST)

**This command commits** — a feature branch with per-subtask commits. It never tags and never merges
to `main`; **you own the release and the merge.** Pass **`--no-commit`** to run the full pipeline and
all gates with zero git writes — changes are left in the working tree for you to review and commit
yourself. `/forge-orchestrate` never rebuilds the graph (that's `/forge-contextmap sync` or the post-commit
hook), so commit or sync first if you have uncommitted structural changes.

---

### /forge-audit — Sweep for Over-Engineering

`/forge-orchestrate` reviews the diff it just wrote. `/forge-audit` sweeps code that **already landed** —
the entire repo (or a scoped path) for accumulated bloat. **Report-only: never edits, never commits.**

```
/forge-audit                    # whole repo
/forge-audit src/checkout       # scope to a path prefix
```

Like `/forge-orchestrate`, it uses `graphify-out/graph.json` to point at candidates instead of blind-reading
every file — then confirms each against the real source before flagging it (*the graph points, the
code decides*).

Flow:
1. **Hard stop** if no `graphify-out/graph.json` — run `/forge-contextmap` first
2. **Graph scan** — surfaces four candidate buckets: orphans / near-dead nodes, duplicate labels across files, god nodes (over-centralization), and dead files (no edge leaves the file — a case `degree <= 1` structurally cannot catch, since a module scores degree from its own symbols)
3. **Source confirmation** — opens each candidate's file and evaluates it against the minimal-code ladder (dead? duplicate? stdlib does it? one-liner?), dropping anything the code proves legitimate
4. **Report** — a grouped delete/simplify list with a rung + rationale per finding, plus god-node structural notes framed as *decompose?* questions, and a summary count

Pass **`--graph-only`** to skip source confirmation (faster, less precise) — every finding is then
tagged `[unconfirmed]`. Tests and files needed for current behavior are never delete-listed.

---

### /forge-diagnose — Find the Root Cause, Hand It Off

A **discussion-driven diagnosis** session for when something is broken. It investigates actively —
runs the program, runs your tests, hits endpoints, reads logs — but **never changes a line of
code**. It ends by writing a handoff prompt: a self-contained instruction the *next* session pastes
in and executes without re-exploring the codebase. Unlike `/forge-orchestrate` and `/forge-audit`, it
does **not** require the graph — it uses it when present and greps when not.

```
/forge-diagnose checkout returns 500 on an empty cart
```

Two problems this solves. First, **premature fixing** — editing before the root cause is understood.
Second, **lost investigation** — the exploration lives only in that session's context, so fixing it
in a fresh session means re-reading the same files from zero.

Flow:
1. **Interrogate before exploring** — error text, expected vs actual, repro steps, when it started, which environment — asked *before* a single source file is read
2. **Reproduce** — runs your existing tests/commands and `git log`/`diff`. Reproduced means the root cause can be `[Certain]`; not reproduced means `[Likely]` at best, and it says so
3. **Locate & trace** — reads `doc/architecture.md` and the graph to scope the search, then traces *backward* from the symptom to where the bad value originates
4. **Rule out** — records every eliminated hypothesis with the evidence that killed it
5. **Discussion checkpoint** — presents the root cause and waits for you to confirm, redirect, or go deeper
6. **Handoff** — writes `doc/diagnosis-<slug>.md`

The handoff carries the root cause with `path` → `symbol` evidence, the reproduction command, the
ruled-out branches ("do not re-investigate these"), the proposed fix, blast radius, and the
verification test. Locations are anchored on symbol names, not line numbers, so they survive the
drift between sessions.

**Then hand it straight to the pipeline:**

```
/forge-orchestrate doc/diagnosis-<slug>.md
```

Pass the *path*, not a re-typed description. Orchestrate reads the handoff's proposed fix as the
task, its verification commands as the success criterion, its blast radius as the files-touched
hint, and its ruled-out branches as "do not re-investigate" — then **skips its own ambiguity scan**,
because the diagnosis already answered it. It branches as `fix/<slug>` instead of `feature/<slug>`.
Re-typing the task instead throws every one of those away.

For a one-file surgical fix, apply the handoff's §7 directly and run its §9 — no pipeline needed.

**"Never changes code" is a contract, not a sandbox.** `Edit` is left out of `allowed-tools`, but
the handoff has to be written, so `Write` is present — the guarantee that nothing but
`doc/diagnosis-*.md` gets written is behavioral.

---

### Flags

| Flag | Command | Effect |
|---|---|---|
| `--no-commit` | `/forge-orchestrate` | Full pipeline and every gate, **zero git writes** — no branch, no commits, changes left in the working tree for you. Also forces single-tree dispatch (no branches → no worktrees) |
| `--new` | `/forge-contextmap` | Force the new-project interview even when source files exist |
| `--graph-only` | `/forge-audit` | Skip source confirmation — faster, less precise, every finding tagged `[unconfirmed]` |

`/forge-contextmap sync` is a subcommand, not a flag. `/forge-audit` also takes a bare path prefix
(`/forge-audit src/checkout`) to scope the sweep.

### When it stops vs. runs hands-off

The pipeline is autonomous by design — it prints its decomposition and keeps going rather than
asking for approval at every step. It interrupts you in exactly these cases:

- **No `graphify-out/graph.json`** — `/forge-orchestrate` and `/forge-audit` hard-stop and tell you
  to run `/forge-contextmap`. They never auto-run it, and never rebuild the graph themselves.
  `/forge-diagnose` doesn't stop — it greps instead.
- **Genuinely ambiguous request** — unclear scope, missing acceptance criteria, or unclear affected
  area gets a real question with options. A *clear* request gets a one-line stated assumption and no
  question. Skipped entirely when the task came from a diagnosis handoff.
- **Dirty working tree** — asks before branching: stash or abort. It will never silently discard or
  commit your pending work.
- **Secret found in the diff** — hard block. No commit, no retry loop (retrying can't un-leak a
  secret), exact match location surfaced, waits for you.
- **No test runner detected** — warns loudly and asks once, because the success criterion can't be
  verified. Records `unverified — no test tooling` in the results either way. Never a fake green.
- **A gate fails 3×** — stops and reports. Bounded loop, never infinite.

And two things it never does, regardless: **tag a release**, or **merge to `main`**. It leaves you
on the feature branch. You own the merge.

---

## What Gets Created

```
your-project/
├── CLAUDE.md                    ← Auto-loaded every session (goal + architecture summary + rules + v2 format marker)
├── .git/hooks/post-commit       ← Silently rebuilds graph on every commit
├── graphify-out/
│   ├── graph.json               ← Knowledge graph (Graphify-managed, don't edit)
│   ├── graph.prev.json          ← Last-sync snapshot (powers the auto-changelog diff)
│   └── GRAPH_REPORT.md          ← Human-readable graph summary
└── doc/
    ├── prd.md                   ← Product requirements: idea overview, core features F1..Fn, out of scope (100% yours)
    ├── app-flow.md              ← Entry point, screen/step map, user journeys, data flow
    ├── design-brief.md          ← Color tokens, typography, components, screen style (UI projects)
    ├── backend-schema.md        ← Storage, entities/tables, relations, indexes (backend projects)
    ├── architecture.md          ← Tech stack, detected patterns, design rules
    ├── domain-model.md          ← Entities, relationships, business rules
    ├── api-contract.md          ← API surface, services, interfaces
    ├── solution-structure.md    ← Folder layout, dependency flow
    ├── coding-standard.md       ← Detected patterns + your standards
    ├── security.md              ← Auth components + your security rules
    ├── task-list.md             ← Engineering plan (100% yours — never graph-populated)
    ├── changelog.txt            ← Updated after every change
    ├── progress.txt             ← Current status (kept short)
    ├── release-readiness.md     ← Written by /forge-orchestrate — CD steps to run on your platform
    └── diagnosis-<slug>.md      ← Written by /forge-diagnose — root cause + handoff for the fix session
```

### The CLAUDE.md rules — keeping Claude on track

`/forge-contextmap` writes these rules into `CLAUDE.md`:

1. Before writing code, always read the universal docs (`architecture.md`, `solution-structure.md`, `coding-standard.md`, `prd.md`), then read only the domain docs the task touches (UI → `design-brief.md` + `app-flow.md`; data → `domain-model.md` + `backend-schema.md`; API → `api-contract.md`; auth → `security.md`) — not every `/doc` file
2. Implement ONLY the next task in `task-list.md` whose dependencies are all done
3. Update `progress.txt` after every completed task
4. Update `changelog.txt` after every change (`Date | Change | Description`)
5. Follow `solution-structure.md` exactly — no structural changes
6. UI code uses ONLY `design-brief.md` tokens and components — no ad-hoc hex values or one-off components
7. Never invent schema, fields, or endpoints not defined in the docs

```
Session N:   auto-load CLAUDE.md → read task-list → implement → update progress + changelog → commit
Session N+1: fresh session → same auto-load → continue from progress.txt
```

### The engineering plan format (`doc/task-list.md`)

Small tasks, explicit build order, verifiable done:

```markdown
## Phase 1 — Foundation
### Task 1.2 — Login screen
- [ ] done
- Depends on: Task 1.1
- Builds: F2
- Acceptance criteria:
  - [ ] works as expected: user can log in and reach home
  - [ ] no errors (lint/console clean)
  - [ ] meets PRD requirement F2
  - [ ] test added
```

`/forge-orchestrate` picks the next task whose dependencies are done, uses the acceptance criteria
as its success criteria, and on completion ticks `done` plus only the criteria its gates actually
verified (tests gate → *test added*, lint gate → *no errors*, …) — never a fake check.

---

## How It Works

### The knowledge graph (Graphify)

[Graphify](https://github.com/safishamsi/graphify) is an open-source Python tool that parses your codebase via tree-sitter AST analysis (29+ languages) and builds a queryable knowledge graph:

| Concept | What it means |
|---------|---------------|
| **Node** | A concept: function, class, module, interface, file |
| **Edge** | A relationship: calls, imports, extends, implements |
| **Edge confidence** | `EXTRACTED` (AST-confirmed) / `INFERRED` / `AMBIGUOUS` |
| **Community** | A cluster of related nodes (Leiden algorithm) |
| **God node** | A highest-degree concept — something everything else depends on |

`/forge-contextmap` reads `graphify-out/graph.json` and maps this data into your doc files.

### The fence protocol

Graph-managed content is wrapped in HTML comment fences:

```markdown
<!-- graphify:auto start:src/auth/auth_service:architecture -->
Content here is extracted from your code.
Overwritten on every /forge-contextmap sync.
<!-- graphify:auto end:src/auth/auth_service:architecture -->
```

**Everything outside the fences is yours.** Claude never touches it during sync.

- Add notes, rules, and context anywhere outside the fences — they survive every sync
- Graph sections refresh automatically when you run `/forge-contextmap sync`
- Fence keys are fully-qualified paths (`source_file:section`) — no collision between modules with the same short name

### The post-commit hook

`.git/hooks/post-commit` runs `graphify update .` in the background after every commit. This keeps `graphify-out/graph.json` current without blocking your workflow.

Output goes to **`graphify-out/.hook.log`**, not `/dev/null`. If the graph ever looks stale, that log is the first place to look — an earlier version of this hook discarded stderr, so when its command was wrong the graph silently stopped updating and nothing said so.

The hook only rebuilds the graph — it never writes to doc files. Run `/forge-contextmap sync` explicitly when you want docs refreshed.

### Session flow

```
You write code → git commit
  └─→ post-commit hook: graphify update . (background, logged to graphify-out/.hook.log)

/forge-contextmap sync
  └─→ reads updated graph.json
  └─→ refreshes <!-- graphify:auto --> sections
  └─→ drafts changelog entries from added/removed nodes (you review)
  └─→ your content outside fences: unchanged

Claude Code session start
  └─→ CLAUDE.md auto-loads (~600 tokens)
  └─→ Claude knows your architecture, key components, rules
  └─→ First message: already in context, no setup needed
```

### Why real data beats placeholders

Standard templates fill `architecture.md` with `[FILL IN: your tech stack]`. You do the work, Claude uses what you wrote — which may be incomplete or stale after a week of coding.

`/forge-contextmap` extracts architecture from your actual code. It finds your central abstractions, maps module dependencies, and detects coding patterns. Docs reflect reality.

### Why structured context means better decisions

Claude makes better decisions with accurate, structured context:
- Won't suggest patterns that contradict your existing architecture
- Won't invent schema fields that don't exist in your domain model
- Won't suggest refactors that break your layer rules

### Why fenced sections prevent drift

Docs that require manual updates go stale. The fence protocol solves this: graph sections are always current, your manual sections are always preserved. Neither overwrites the other.

### Why god nodes matter most

Not all code is equally important. God nodes are the concepts with the most connections — what everything else depends on. These surface in `CLAUDE.md`, giving Claude immediate awareness of your architecture's center of gravity before you type anything.

---

## For Maintainers

*(Plugin packaging, then shared blocks. Requirements are under [Install](#requirements).)*

### Plugin packaging

The repo root doubles as both the marketplace and the single plugin it publishes:

| File | Role |
|---|---|
| `.claude-plugin/marketplace.json` | Marketplace `contextforge`, one entry whose `source` is `"./"` — the repo root *is* the plugin. Metadata only; no component paths, or Claude Code errors with `conflicting manifests` |
| `.claude-plugin/plugin.json` | Plugin manifest. `"skills": ["./.claude/skills/"]` points at the existing skills tree, so the working tree still loads them as project-level skills for live editing — no `skills/` copy to keep in sync |

**`version` is deliberately absent from both files.** Claude Code then falls back to the git commit
SHA, so every pushed commit registers as a new version and installs pick it up. Adding a `version`
string pins users until you bump it. `claude plugin validate .` warns about the missing field —
that warning is expected.

Verify a change to either manifest with `claude plugin validate .` (schema) followed by
`claude plugin details contextforge` (component inventory — must list all four skills; `validate`
alone passes even when the skills path resolves to nothing).

### Shared blocks

Each skill installs standalone, so a skill can never read a file from a sibling skill's directory —
that would dangle for anyone who installed only one. A handful of blocks are therefore **duplicated
on purpose**, each wrapped in `<!-- forge:shared-block <name> -->` (or `# forge:shared-block <name>`
inside Python). Edit one copy, update the rest:

| Block | Copies |
|---|---|
| `graph-hard-stop` | `skills/forge-orchestrate/SKILL.md` 0a · `skills/forge-audit/SKILL.md` 0b |
| `python-cmd` | `skills/forge-orchestrate/SKILL.md` 0b · `skills/forge-audit/SKILL.md` 0c · `skills/forge-contextmap/references/sync.md` S1 |
| `graphify-cli` | `skills/forge-contextmap/references/sync.md` S1.5 — **one copy, not duplicated.** Both consumers live in `forge-contextmap`, so `references/existing-project.md` E2.5 just points at it. Listed here because a second skill needing the CLI must copy it rather than reach across directories. |
| `graph-loader` | `skills/forge-orchestrate/references/graph-slice.md` · `skills/forge-audit/SKILL.md` Phase 1 · `skills/forge-contextmap/references/sync.md` S3.6 |
| `bloat-buckets` | `skills/forge-audit/SKILL.md` Phase 1 · `skills/forge-contextmap/references/sync.md` S3.6 — thresholds must match, or sync's counts disagree with audit's |
| `minimal-ladder` | `skills/forge-orchestrate/SKILL.md` 3c (**payload copy**) · same file 5b.3 (review copy) · `skills/forge-audit/SKILL.md` Phase 2 |
| `source-doc-map` | `skills/forge-orchestrate/SKILL.md` 3b · `skills/forge-contextmap/references/doc-templates.md` CLAUDE.md rule 1 |

Two constraints that are not negotiable:

- **The `minimal-ladder` payload copy in orchestrate 3c must stay literal text.** It is injected into
  worker subagent prompts, and a subagent sees only its dispatch payload — never the skill file. A
  pointer there would silently ship workers with no ladder.
- **The `source-doc-map` marker in `doc-templates.md` sits *outside* the template fence.** Everything
  inside that fence is copied verbatim into the user's generated `CLAUDE.md`; a marker in there
  would leak into every scaffolded project.

Verify after editing any of them:

```bash
grep -rn "forge:shared-block" .claude          # every marker — copies come in pairs
grep -rn "\.\./[a-z-]*/references" .claude     # must be empty — no cross-skill reads
```

(A bare `../` search is useless here: `worktree-mode.md` legitimately contains `../<repo>-st<i>`
sibling checkout paths.)

---

## License

MIT
