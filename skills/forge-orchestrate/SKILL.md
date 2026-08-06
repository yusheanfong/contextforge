---
name: forge-orchestrate
description: CI/CD-gated autonomous feature pipeline (/forge-orchestrate). Use when building a feature end-to-end against the knowledge graph — decompose, branch, dispatch graph-scoped worker subagents, run lint/test/secrets/over-engineering gates, commit per subtask. The `codex` subcommand plans through a Claude+Codex planning council, keeps review on Claude, and hands implementation to the Codex CLI. Also use to execute a /forge-diagnose handoff. Triggers include "/forge-orchestrate", "/forge-orchestrate codex", "orchestrate this feature", "build the next task", "run the pipeline", "execute the diagnosis", "plan on Claude execute on Codex", "planning council".
argument-hint: "[codex] [feature | doc/diagnosis-*.md] [--no-commit]"
---

# /forge-orchestrate — CI/CD-Gated Autonomous Feature Pipeline

Take one feature request and run it end-to-end: parse intent → decompose → branch → dispatch
graph-scoped worker subagents → run CI quality gates → commit per subtask → report. Asks
questions **only when the request is genuinely ambiguous**; otherwise runs hands-off.

This is a hierarchical multi-agent pipeline (Coordinator → Worker → Critic → Synthesis) where each
worker is fed ONLY the doc files and graph nodes its subtask touches. Context isolation is derived
from `graphify-out/graph.json` (built by `/forge-contextmap`), not hand-curated.

Companion to `/forge-contextmap`: contextmap builds the map, `/forge-orchestrate` uses it to execute.
Reads `$ARGUMENTS` as the feature request, or as the path to a `/forge-diagnose` handoff.

**This command commits** (feature branch + per-subtask commits). It never tags and never merges to
`main` — you own releases. Pass **`--no-commit`** to run the full pipeline + gates with zero git
writes (you review the working tree and commit yourself).

**`codex` subcommand** — `/forge-orchestrate codex <feature>` plans through a two-agent **planning
council** and hands implementation to the Codex CLI. Claude decomposes, then one read-only Codex call
both critiques that decomposition and returns its own cut of the same task, and Claude decides — all
before a line of code is written, because a plan flaw is cheapest to fix while it is still a plan.
Claude stays coordinator, synthesizer, reviewer and
final decision-maker. Everything else is identical, `--no-commit` still composes, and the main
session still owns every git write.

**Roles (mapped from the multi-agent blueprint):** Coordinator = intent-parse + decompose · Engineer
= worker subagent that edits code + writes tests · QA/Critic = the CI gate runner + review loop ·
Synthesis = final report + release-readiness.

**Global state object** — track these across phases and surface them in the final report:
`branch`, `subtasks[]` (goal, criterion, deps, independent, gate_set, status, agent_id, commit_sha,
`slice` = the key words its graph slice was produced from, so a stale slice is visible after a
revision), `gate_results`, `files_changed`, `commits[]`, `all_gates_pass`.

Under `[BACKEND] = codex`, also `council`: `run_id`, `verdict`, `unresolved[]`. The
council's single call is never resumed, so it holds no session id — every stored session id belongs
to a subtask's `agent_id`.

**Reference files** — read each only when its branch applies, not up front:

| File | Read when |
|---|---|
| [`references/graph-slice.md`](references/graph-slice.md) | Phase 3a — always (the slice script lives there) |
| [`references/codex-backend.md`](references/codex-backend.md) | `[BACKEND] = codex` — read right after 0d, before the council |
| [`references/task-list-formats.md`](references/task-list-formats.md) | `doc/task-list.md` exists (Phase 0d + Phase 6) |
| [`references/worktree-mode.md`](references/worktree-mode.md) | a batch has 2+ independent subtasks AND `[NO_COMMIT]` is false |
| [`references/release-readiness.md`](references/release-readiness.md) | Phase 6 |

---

## PHASE 0: Intent Parsing & Clarification *(Coordinator / Router)*

### 0a. Graph prerequisite — hard-stop

<!-- forge:shared-block graph-hard-stop variant:orchestrate -->
`/forge-orchestrate` is graph-driven. Use the Read tool (or a file check) for `graphify-out/graph.json`
in the current directory.

If it does NOT exist, print this exact message and STOP — do nothing else (do not auto-run
`/forge-contextmap`; the user builds the graph manually):

```
❌ /forge-orchestrate is graph-driven and needs graphify-out/graph.json.
Run /forge-contextmap first to build the knowledge graph, then re-run /forge-orchestrate.
```
<!-- /forge:shared-block graph-hard-stop -->

### 0b. Resolve the Python interpreter as `[PYTHON_CMD]`

<!-- forge:shared-block python-cmd variant:orchestrate -->
Graphify is normally installed with **uv** or **pipx**, which put it in its own virtualenv. That
venv's python has `networkx`; your system `python3` almost certainly does not. Resolving the wrong
one is the single most common way these scripts die. Work down this list and stop at the first
interpreter that passes step 5.

1. If `graphify-out/.graphify_python` exists, read it — that one line is an interpreter path graphify
   already validated. Use it verbatim. (Recent graphify CLI versions do **not** write this file, so
   expect it to be missing and keep going.)
2. **uv** — ask uv where its tools live:
   ```bash
   uv tool dir
   ```
   Then try, in order, `<that dir>/graphifyy/bin/python` (macOS/Linux) and
   `<that dir>\graphifyy\Scripts\python.exe` (Windows). A path that does not exist just errors —
   harmless, move on.
3. **pipx** — same idea:
   ```bash
   pipx environment --value PIPX_LOCAL_VENVS
   ```
   Try `<that dir>/graphifyy/bin/python` and `<that dir>\graphifyy\Scripts\python.exe`.
4. Fall back to the bare interpreter. Run these as **two separate calls** — `||` is not available in
   PowerShell 5.1, and the first that succeeds wins:
   ```bash
   python --version
   ```
   ```bash
   python3 --version
   ```
5. **Verify the candidate before accepting it** — the Phase 3a slice needs them:
   ```bash
   [PYTHON_CMD] -c "import networkx, json"
   ```
   If it fails, this is the wrong interpreter — go back and try the next candidate. If **every**
   candidate fails, print:
   ```
   ❌ /forge-orchestrate found no Python with networkx. Graphify installs one in its own venv —
      try: uv tool install graphifyy   (or: pipx install graphifyy)
   ```
   and STOP. Always report which interpreter you settled on.
<!-- /forge:shared-block python-cmd -->

### 0c. Graph freshness note (print, don't block)

`/forge-orchestrate` reads whatever `graph.json` currently exists — it never rebuilds it. The post-commit
hook refreshes the graph on every `git commit`, so committed edits are already reflected.

Check for uncommitted changes (best-effort — ignore failure):
```bash
git status --porcelain
```
If there are uncommitted changes that look structural (new/renamed source files), print once:
```
ℹ️ Uncommitted structural changes won't be in the graph slice — run /forge-contextmap sync or commit
   first for the most accurate routing. (Workers still read live code, so this only affects which
   files they're pointed at, not correctness.)
```
Do not force a sync. Continue.

### 0d. Resolve the request

First, parse the **backend subcommand**: if `$ARGUMENTS` starts with the bare word **`codex`**, set
`[BACKEND] = codex` and strip that word. Otherwise `[BACKEND] = claude`. It is a subcommand, not a
flag — same shape as `/forge-contextmap sync` — and it must be parsed first so that both
`/forge-orchestrate codex --no-commit <feature>` and `/forge-orchestrate codex doc/diagnosis-x.md`
resolve correctly. Under `[BACKEND] = codex`, read [`references/codex-backend.md`](references/codex-backend.md) now and run its C1
preflight before the council; a missing `codex` binary stops the run before anything is written, and
its other checks warn and continue.

Announce the parse in one line — `Backend: codex — Claude and Codex plan as a council, Codex
implements.` — and continue without blocking. A feature request whose own first word is "codex" would otherwise be
silently mis-split, and this line makes that visible in time to correct it.

Next, parse flags: if what remains contains **`--no-commit`**, set `[NO_COMMIT] = true`
and strip the flag from the text. Otherwise `[NO_COMMIT] = false`. Under `[NO_COMMIT]` the pipeline
runs every phase and gate but makes NO branch and NO commits — it leaves changes in the working
tree for the user to review and commit.

Then resolve the request, in this order:

1. **Diagnosis handoff** — if `$ARGUMENTS` (flags stripped) names or resolves to a
   `doc/diagnosis-*.md`, read that file. It is a `/forge-diagnose` handoff: a root cause already
   established with evidence, and the investigation already done. Map it in:
   - `[TASK]` ← §7 *Proposed fix*, framed by §1 *Root cause*
   - `[TASK_CRITERIA]` ← §9 *Verification* — its before/after commands ARE the success criterion
   - `[BLAST_RADIUS]` ← §8 — carry into Phase 1 as the files-touched hint for decomposition
   - `[RULED_OUT]` ← §5/§6 — carry into every worker payload as "do not re-investigate these"
   - Set `[FROM_DIAGNOSIS] = true`. **Skip 0e entirely** — the diagnosis session already resolved
     the ambiguity, and re-asking is exactly the waste the handoff exists to prevent. If the
     handoff's §6 *Open questions* section is non-empty, surface those to the user once before
     proceeding; do not re-derive them.
2. **Explicit request** — otherwise, if `$ARGUMENTS` (flags stripped) is non-empty, that is the
   request.
3. **Next task from the plan** — if `$ARGUMENTS` is empty:
   - If `doc/task-list.md` exists → read [`references/task-list-formats.md`](references/task-list-formats.md) and follow its
     *Phase 0d* section to pick the next eligible task. Wait for confirmation.
   - Otherwise ask: `"What feature should I orchestrate?"` and wait.

Store the request as `[TASK]`. If it came from a v2 task block, also store `[TASK_CRITERIA]`
(its acceptance-criteria lines) and `[TASK_FEATURE]` (its `Builds:` PRD feature ref).

### 0e. Ambiguity scan — the "ask only if unclear" rule

**Skip this step entirely when `[FROM_DIAGNOSIS]` is true.**

Evaluate `[TASK]` for genuine ambiguity: unclear **scope** (how much is in/out), missing
**acceptance criteria** (what "done" looks like), or unclear **affected area** (which part of the
codebase). Cross-check against the graph slice (Phase 3 tooling) and docs.

- **If genuinely ambiguous** → use `AskUserQuestion` with specific options. **Never guess on real
  ambiguity.** Wait for the answer.
- **If clear enough to proceed** → state your assumptions in one line and continue. Do NOT
  manufacture questions for a clear request.

### 0f. Derive the feature branch name

**If `[NO_COMMIT]` is true, skip this step** — no branch is created; jump to Phase 1 and ignore the
branch references throughout.

Slugify `[TASK]` into `feature/<slug>` (e.g. "create a feature where the login button works" →
`feature/login-button-works`). Store as `[BRANCH]`.

**If `[FROM_DIAGNOSIS]` is true**, use `fix/<diagnosis-slug>` instead — reuse the slug from the
`doc/diagnosis-<slug>.md` filename so the branch, the diagnosis, and the fix all share one name.

- If you are asking clarifying questions in 0e anyway, fold in a branch-name confirmation.
- If no questions were needed, announce: `Branch: [BRANCH] — proceeding (reply to rename).` and
  continue without blocking.

---

## PHASE 1: Decompose / Micro-plan *(Coordinator = PM agent)*

**Under `[BACKEND] = codex`, start here as normal, then run the council in 1b.** 1a's rules produce
the decomposition; the council's single Codex call then critiques it and returns Codex's own cut of
the same task, and 1b.4 is where you decide between them. Do not dispatch that call before the
decomposition exists — it is the call's main input.

### 1a. Decomposition rules

Break `[TASK]` into an ordered list of subtasks. For each subtask record:

- **goal** — one line
- **success criterion** — a verifiable check (test passes, output matches, file exists).
  **If `[TASK_CRITERIA]` exists** (task came from a v2 task-list block, or from a diagnosis
  handoff's §9), derive success criteria FROM those lines — don't invent parallel ones. For a
  task-list task: the "works as expected" criterion becomes the behavioral check; "test added" is
  already mandated by Phase 4; "no errors" maps to the lint gate; "meets PRD requirement [Fn]"
  makes `doc/prd.md`'s feature Fn part of the spec-compliance check in 5b.1.
- **depends on** — which earlier subtasks must finish first (or `none`)
- **independent** — `yes` if it can run in parallel with its siblings, else `no`
- **gate set** — which CI gates apply (default: all detected; note any to skip, e.g. docs-only edit)

**If `[BLAST_RADIUS]` exists** (diagnosis handoff), use it as the files-touched hint — those call
sites are known dependents of the change and usually belong in the same subtask or an explicit
follow-up one, not scattered.

Initialize the global state object with these subtasks.

Print the decomposition for transparency, then **proceed immediately — no approval stop**
(clarify-then-run). Under `[BACKEND] = codex` this print happens at 1b.4, once the plan is final;
the last line becomes the council's:

```
Plan for [TASK] (branch [BRANCH]):

Subtask 1: [goal]
  success: [criterion]   depends on: none   independent: yes   gates: lint,test,audit
Subtask 2: [goal]
  success: [criterion]   depends on: 1      independent: no    gates: lint,test
...

Running now. I'll stop only if a request is ambiguous, the planning council ends unresolved, or a
gate fails 3×.
```

### 1b. Planning council *(codex backend only — skip entirely when `[BACKEND] = claude`)*

Before any code is written, Codex reviews your decomposition. Follow
[`references/codex-backend.md`](references/codex-backend.md) §C3 — it carries the schema, the invocation shape, the payload and
the budget rule. **One Codex call, read-only, and there is no second one.** The dispatch is
background (§C2); a planning call can exceed a 10-minute foreground cap, and a killed call is the
whole budget.

1. **1b.0 — set up.** Pick the `run_id` and give this run its own directory,
   `graphify-out/.orchestrate_council_<run_id>/` — never purge another run's, which may be live
   (C3). Write the schema there. Produce a **task-level** graph slice: read
   [`references/graph-slice.md`](references/graph-slice.md) and run its script with `[TASK]`'s key words.
2. **1b.1 — your decomposition.** Apply 1a's rules. Don't print it yet — 1b.4 prints the final one.
3. **1b.2 — per-subtask slices** ([`references/graph-slice.md`](references/graph-slice.md) again). These are **advisory
   planning inputs**: Phase 3a re-runs them after the branch exists.
4. **1b.3 — the call.** Fresh session. The payload carries your decomposition and its per-subtask
   `FILES`, the task and its criteria, `[TASK_FEATURE]`, the diagnosis facts (`[BLAST_RADIUS]`,
   `[RULED_OUT]`), any assumption 0e resolved, and the graph/doc/`CLAUDE.md` *paths*. It asks for two
   things in one object: a `verdict` + `findings` on your plan, and Codex's **own** `subtasks[]` for
   the same task. Validate the result semantically, not just against the schema.
5. **1b.4 — decide, then print.** Incorporate or reject each finding **with the repository evidence
   that justifies it**. Read Codex's `subtasks[]` as evidence about the shape of the work — never
   adopt it wholesale, never discard it unread, and where you keep your own slice, say why.
   **Re-slice every subtask you revised** — Phase 3b routes docs off the slice's `FILES`, so a stale
   one mis-routes them. Reconcile `execution_order` against the `depends on` fields here, in the
   decomposition, never as a silent override at dispatch time; it is returned against Codex's own
   subtask numbers, so map it onto yours before using it to break ties. Then print the plan block
   above.

**Then the approval policy:**

- **`verified`, or `flawed` with every finding incorporated** → continue to Phase 2 hands-off. No
  stop. The normal path.
- **Any finding you did not incorporate, or a degraded call** → print the final plan, Codex's
  outstanding objection and the concrete risk it names, then **ask before executing**. A rejection
  backed by evidence still stops here: with no verifying round, nothing checked it. Never describe
  this as consensus. This is a pause, not an abort: 6z keeps the council artifacts.

---

## PHASE 2: Branch Setup *(enables auto-commit)*

**If `[NO_COMMIT]` is true, skip this entire phase** — no branch, no dirty-tree prompt; workers
edit the current tree in place and the user commits later.

1. **Dirty-tree check** (the one legitimate blocking question):
   ```bash
   git status --porcelain
   ```
   If the tree is dirty, ASK before proceeding — stash (`git stash push -u`) or abort. Do not
   silently discard or commit the user's pending work.

   **Under `[BACKEND] = codex`, ignore this run's own scratch here.** The council (1b) writes
   payloads, schemas and results into `graphify-out/` before this check runs, and that directory is
   not guaranteed to be gitignored — so on a repo that tracks it, an untouched tree looks dirty and
   you would ask the user to stash your own files. Match on the **path prefix**, not exact names:
   `git status --porcelain` can collapse the whole untracked directory into one `?? graphify-out/`
   line. If the user chooses to stash, exclude the run's scratch from it (or regenerate it after) —
   `git stash push -u` would otherwise take the council state with it.

2. **Resolve the base branch as `[BASE]`** — before the branch is created, because it is the branch's
   start point:

   <!-- forge:shared-block base-branch -->
   Resolve the repository's base branch as `[BASE]`. It is `main` on some repos and `master` on
   others — detect it, never assume. Run these as **separate Bash calls** (`||` is not available in
   PowerShell 5.1) and take the first that answers:

   ```bash
   git symbolic-ref --quiet refs/remotes/origin/HEAD
   ```

   Prints `refs/remotes/origin/<name>` — strip the prefix to get the name. A silent non-zero exit
   means there is no remote HEAD to read; move on.

   ```bash
   git rev-parse --verify --quiet refs/heads/main
   ```

   ```bash
   git rev-parse --verify --quiet refs/heads/master
   ```

   `--quiet` is what makes a missing ref silent without `2>/dev/null`, which the portability
   contract bans — read the exit code, not the output.

   A probe only counts as answering if the name it yields exists as a **local** branch — the first
   probe reads a remote ref, which can name a branch that was never checked out here or was deleted
   locally. Confirm before accepting the name, and if the confirmation fails, keep going down the
   list rather than stopping:

   ```bash
   git rev-parse --verify --quiet refs/heads/[BASE]
   ```

   If the first probe found nothing and **both** `main` and `master` exist locally, ask which one
   with AskUserQuestion. Guessing here merges the work into the wrong branch.
   <!-- /forge:shared-block base-branch -->

   If nothing resolves, print `ℹ️ Base branch not resolved — branching from HEAD and skipping the
   merge-conflict pre-check.`, then take the HEAD fallback in step 3 and skip step 4 entirely. Never
   block a feature on this.

3. **Create + checkout the branch, explicitly from `[BASE]`.** Probe first — `-b` fails hard when the
   branch already exists, and re-running the same feature is normal:

   ```bash
   git rev-parse --verify --quiet refs/heads/[BRANCH]
   ```

   **Exit 0 — the branch already exists.** Check it out and note it. This is a rerun and its existing
   commits are the point, so the start point is not reconsidered:

   ```bash
   git checkout [BRANCH]
   ```

   **Non-zero — new branch.** First check whether HEAD is carrying committed work that branching
   from `[BASE]` would leave behind:

   ```bash
   git log --oneline [BASE]..HEAD
   ```

   Empty is the normal case — you are on `[BASE]`, or level with it. Create the branch without
   asking:

   ```bash
   git checkout -b [BRANCH] [BASE]
   ```

   Not empty means HEAD has commits `[BASE]` doesn't, and there is no safe default: you are either
   sitting on a stale branch by accident, or deliberately stacking this feature on another one. Ask
   which, then use `git checkout -b [BRANCH] [BASE]` or bare `git checkout -b [BRANCH]` to match the
   answer.

   The start point is explicit on purpose. Bare `git checkout -b [BRANCH]` branches from HEAD, so
   running `/forge-orchestrate` from an unrelated branch quietly drags that branch's commits into the
   feature branch — and from there into `main` when `/forge-merge` lands it. They are real commits,
   so nothing looks broken until they show up somewhere nobody reviewed them for.

4. **Merge-conflict pre-check** vs `[BASE]` (checklist: "automated merge conflict detection").
   Best-effort throughout — a repo with no remote fails the fetch, which is fine:

   ```bash
   git fetch
   ```

   ```bash
   git merge-base --is-ancestor origin/[BASE] HEAD
   ```

   If the fetch failed or `origin/[BASE]` does not exist, fall back to the local ref:

   ```bash
   git merge-base --is-ancestor [BASE] HEAD
   ```

   A non-zero exit means this branch is missing commits `[BASE]` already has. On a branch freshly
   created in step 3 that can only come from the `origin/[BASE]` form — you have fetched work that
   is not in your local `[BASE]` yet. On a rerun it means `[BASE]` moved while the branch was in
   flight. Either way a conflict is likelier, so print a one-line warning. Do not block.

---

## PHASE 3: Per-Subtask Context Assembly *(RAG — the core)*

For each subtask, build an **isolated context payload**. This payload is the ENTIRE context the
worker will see — it never inherits this session's history.

### 3a. Graph slice

Read [`references/graph-slice.md`](references/graph-slice.md) and follow it exactly. It writes the slice script once, runs it per
subtask with the subtask's key terms, and returns the touched source files + node labels + key
edges. `NO_MATCH` means the graph has no node for these terms — fall back to the universal docs only
(3b) and tell the worker the graph had no specific match.

**This phase is authoritative even when 1b already sliced.** The council's slices were computed
before Phase 2 chose a branch, and Phase 2 can check out an existing feature branch carrying prior
commits or branch from `[BASE]` rather than HEAD — after which worktrees inherit that branch. A
pre-branch slice can therefore describe a tree nobody is editing, and 3b derives the doc set straight
from `FILES`, so a stale slice mis-routes docs silently. Re-run the slice here for every subtask and
overwrite the advisory result.

### 3b. Doc slice

**Pass paths, not text.** Workers have the Read tool, and the payload is routing — the same
pointers-not-content rule the graph slice already follows. Do NOT read these docs and paste them in;
a pasted doc set is ~2k *output* tokens per dispatch on a small project and more on a large one,
re-emitted on every retry.

List `doc/*.md` **once per run** (one glob, cached) so you know what actually exists. Then, from the
`FILES` the slice returned, name the matching domain docs plus the universal ones — paths only, and
only paths the glob confirmed. Naming a doc the project never scaffolded costs the worker a wasted
turn on a failed Read.

<!-- forge:shared-block source-doc-map variant:dispatch -->
- **Always:** `doc/architecture.md`, `doc/solution-structure.md`, `doc/coding-standard.md`, and
  `doc/prd.md` if it exists (small — it's the scope guard for spec compliance)
- UI/screen/widget/view/component sources → `doc/design-brief.md` + `doc/app-flow.md`
  (v1 fallback: `doc/ui-guideline.md` if the project hasn't migrated)
- controller/service/handler/endpoint/route/api sources → `doc/api-contract.md`
  + `doc/backend-schema.md` if it exists
- entity/model/enum/domain sources → `doc/domain-model.md` + `doc/backend-schema.md` if it exists
- auth/token/permission/role sources → `doc/security.md`
<!-- /forge:shared-block source-doc-map -->

**One exception — paste `doc/design-brief.md` inline for UI subtasks.** It is about a page, and 5b's
design check is the only one that hard-fails on a miss; that's cheaper than a re-loop.

**What this means for the main session:** you never Read `architecture.md`, `solution-structure.md`
or `coding-standard.md` at all, and you read `prd.md` only if Phase 5b's reviewer can't (it can —
it reads its own). The one remaining on-demand read is Phase 0e's ambiguity scan, which fires only
on a genuinely ambiguous request and once per run, not once per subtask.

### 3c. Instruction

Assemble the worker prompt from: the subtask **goal** + **success criterion** + the graph slice
(3a) + the doc paths (3b) + these constraints, verbatim. The read gate leads the block — it is a
blocking precondition, not a footnote, and a worker that skims will still hit it first:

<!-- forge:shared-block minimal-ladder variant:payload -->
```
FIRST — before writing any code, read these. They are the binding constraints for this subtask
and you have not seen them:
  [every path 3b selected AND the glob confirmed, one per line, each with a short what-it-is —
   e.g. "doc/coding-standard.md  — language + framework conventions". Emit nothing for a doc
   this project doesn't have.]
Read doc/prd.md only if this subtask names a `Builds: Fn` reference above.
Do not start editing until you have read them.

- Edit only the files needed for THIS subtask.
- Write/update tests and run them; report results.
- Keep lint/style clean — match the project's existing conventions.
- UI subtasks: use ONLY doc/design-brief.md tokens and components — no ad-hoc hex values, font
  sizes, spacing values, or one-off components. Need something new? Report it; don't invent it.
- Do NOT git commit and do NOT touch unrelated code.
- If you lack context to proceed, stop and report NEEDS_CONTEXT with what's missing.
- Return: what you changed (files + summary) and test results.

Engineering discipline (write lean) — before writing each piece of code, walk this ladder
top-down and stop at the first rung that applies:
  1. "Does this need to exist?" → skip it (YAGNI)
  2. "Already in this codebase?" → reuse it, don't rewrite
  3. "Stdlib does it?" → use it
  4. "Native platform feature?" → use it
  5. "Installed dependency?" → use it
  6. "One line?" → one line
  7. Only then: the minimum that works
GUARD: the ladder never applies to the tests Phase 4 mandates or to any file needed to satisfy the
success criterion. Those are always required — never skip them as "YAGNI."
```
<!-- /forge:shared-block minimal-ladder -->

> The ladder above is `forge:shared-block minimal-ladder`, in its **payload copy**, and it has to
> stay literal text: a worker sees its dispatch payload, never this file, so it can never become a
> pointer. The ladder specifically — rungs 2–5 are the "don't reinvent what exists" half, and
> `CLAUDE.md`'s *Simplicity First* does not cover them.
>
> The surrounding discipline bullets used to be restated here too and no longer are. Workers are
> `general-purpose` subagents, and [every subagent except `Explore` and `Plan` loads the project's
> `CLAUDE.md`](https://code.claude.com/docs/en/sub-agents) — so a ContextForge project's *Coding
> Rules* (think-before-coding, surgical changes, goal-driven execution) already reach the worker
> once, for free. Restating them cost ~90 output tokens on every dispatch and every retry. Two of
> them also duplicated bullets already in this same payload.

**If `[RULED_OUT]` exists** (diagnosis handoff), append it to the payload as: "Already ruled out by
the diagnosis — do not re-investigate: [list]."

---

## PHASE 4: Execution *(Worker = Engineer agent)*

**If `[BACKEND] = codex`**, the payload assembled in Phase 3 is used verbatim — that reuse is the
point — but it goes to `codex exec` instead of the Agent tool. Follow [`references/codex-backend.md`](references/codex-backend.md)
§C2, §C4 and §C5 for dispatch, and ignore only the Agent-tool paragraph and the model-selection
paragraph below. The dispatch-mode section and everything from Phase 5 on still apply.

Use the Agent tool. **Default `subagent_type` is `general-purpose`** (the documented catch-all);
use `Explore`/`Plan` only for read-only research subtasks.

Model selection: cheap/fast model for mechanical 1–2 file edits with a clear spec; a more
capable model for multi-file integration or design judgment.

Each worker receives ONLY its payload from Phase 3 — not this session's history. Workers edit
code + write tests + run them, and do **NOT** commit. Committing happens in Phase 5 after gates pass.

**Record every dispatched worker's agent ID/name in its subtask state (`agent_id`).** 5c resumes
that live agent instead of respawning it, and without the ID there is nothing to resume. This
applies to both dispatch modes below. Under `[BACKEND] = codex`, `agent_id` holds the session id
`codex exec` prints on stdout.

### Dispatch mode — worktrees vs. single tree

Decide per batch of sibling subtasks:

- **Worktree mode** — a batch has **2+ independent (parallel) subtasks** AND `[NO_COMMIT]` is false.
  Read [`references/worktree-mode.md`](references/worktree-mode.md) and follow it for dispatch, commit (5d), merge-back (5e), and
  Phase 6 cleanup. This is the preferred path for real parallelism.
- **Single-tree mode** — a lone subtask, a sequence of dependent subtasks, or `[NO_COMMIT]` (no
  branches → no worktrees). Workers edit the current checkout directly. Everything you need is
  below; do not read `worktree-mode.md`.

#### Single-tree mode

- **Independent subtasks** (only one, or `[NO_COMMIT]`): dispatch in the current checkout; if more
  than one runs in parallel here, the serialize + collision rule in Phase 5d applies.
- **Dependent subtasks**: dispatch sequentially. Pass the prior worker's COMPACTED summary forward
  (a few lines: what changed + relevant outputs) — never its full transcript.

---

## PHASE 5: CI Gate / Validation *(Critic = QA agent)*

This is where the executable CI subset of the CI/CD checklist lives. After a worker returns, run
the **adaptive gate runner**, then the review checks, then commit on pass.

### 5a. Adaptive gate runner

Detect available tooling from the project manifests (`package.json`, `pyproject.toml`,
`requirements.txt`, `go.mod`, `*.csproj`, etc.), **or** from a convention directory plus a runner on
PATH — a `tests/` or `spec/` directory with `pytest` / `go test` / `jest` available is working test
tooling even with no manifest in the repo. **A missing manifest is not evidence of missing tooling**;
check PATH before recording a skip. Then **run what exists, skip + honestly report what doesn't**.
Never print a passing check for a gate you didn't actually run. Run in this order and record each
result as `pass` / `fail` / `skipped (reason)`:

**A worker's, a reviewer's or Codex's statement about test results is a review input, never a gate
result.** The only counts that may reach a commit message, `doc/progress.txt`, `doc/changelog.txt` or
the Phase 6 report are the ones printed by the command *this session* ran here. If a reviewer reports
failures you have not seen, run the test command yourself before recording anything — and if your run
does not reproduce them, that is **not** a gate failure: it does not enter 5c and does not spend one
of the three iterations. Record what you observed and report the discrepancy. The codex backend makes
this easy to get wrong: more agents report results, so there is more to relay.

1. **Lint / style** — e.g. `eslint`, `ruff check`, `flake8`, `dotnet format --verify-no-changes`,
   `go vet`. (checklist: "linting and code style enforcement")
2. **Tests** — e.g. `npm test`, `pytest`, `go test ./...`, `dotnet test`. MUST pass before commit.
   (checklist: "tests run before allowing merge", "functional tests")
   - **A collection/import error is not a missing runner.** In a manifest-less repo `pytest` often
     dies with `ModuleNotFoundError: No module named 'src'` because nothing put the repo root on
     `sys.path`. That is a `fail` to report, not a `skipped` — the runner exists and ran. Retry once
     with `PYTHONPATH=.` (or note that the repo needs a root `conftest.py`) before concluding
     anything about the code under test.
   - **If NO test runner is detected at all** — after checking PATH, not just the manifests: the
     subtask's success criterion cannot be verified.
     Do NOT record a `pass` and do NOT silently commit. Warn loudly and ask once:
     ```
     ⚠️ No test runner detected — I can't verify "[subtask success criterion]" actually works.
        Proceed and commit unverified, or stop here?
     ```
     Record this gate as `unverified — no test tooling` in the results table regardless of choice.
3. **Coverage** — add the runner's coverage flag if it supports one; enforce a threshold only if
   the repo configures one. (checklist: "test coverage reporting")
4. **Dependency-vuln scan** — `npm audit`, `pip-audit`, `dotnet list package --vulnerable`.
   (checklist: "dependency vulnerability scanning / monitoring")
5. **Secrets scan** — `gitleaks detect` if installed; else a regex fallback over the diff
   (`git diff` for API keys, tokens, private keys). (checklist: "secrets scanning in code")
   - **A secrets finding is a HARD BLOCK:** never commit, do NOT enter the retry loop (retrying
     does not un-leak a secret), and surface the exact match location to the user immediately.
     Stop the subtask and wait for the user to remove the secret.
6. **SAST** — `semgrep --error` if installed; else `skipped (semgrep not installed)`. (checklist:
   "SAST")

### 5b. Review checks — ONE read-only reviewer subagent

All four checks run in a **single reviewer dispatch**, and the reviewer runs `git diff` **itself**.
The diff never enters this session's context or its output. That matters because main-session
tokens are recurring — paid again on every later turn — while a subagent's are discarded when it
returns. Splitting the checks defeats it: any one of them left here drags the whole diff back in.

Assemble the reviewer payload from:

- the subtask **goal** + **success criterion**
- **the diff command**, with the right cwd for the dispatch mode (see below) and scoped to the files
  the worker reported changing. Under `[BACKEND] = codex` that scope comes from the git snapshot
  delta plus C5's separately confirmed ignored paths in [`references/codex-backend.md`](references/codex-backend.md), not from the
  worker's self-report — everywhere below that says "the files the worker reported", read "the
  reconciled list plus its ignored no-diff paths". The reviewer itself is unchanged: still a
  read-only **Claude** subagent, because the executor must not grade its own diff.
- the minimal-code ladder — paste the block below into the payload. A subagent sees only its
  dispatch payload, never this file:
  <!-- forge:shared-block minimal-ladder variant:review -->
  1. Does this need to exist? → shouldn't (YAGNI)
  2. Already in this codebase? → should have reused it
  3. Stdlib does it? → should have used it
  4. Native platform feature? → should have used it
  5. Installed dependency? → should have used it
  6. One line? → should be one line
  7. Only then: the minimum that works
  <!-- /forge:shared-block minimal-ladder -->
- **Scope guard, verbatim:** never delete-list the tests Phase 4 mandates, or any file needed to
  satisfy the subtask's success criterion. Minimality never overrides the "workers must write
  tests" rule.
- the four checks, each one line, the last two conditional:
  1. **Spec compliance** — does the diff do exactly what the subtask asked, nothing missing and
     nothing extra? If the subtask carries a `Builds: Fn` reference, add: "read `doc/prd.md` and
     check the diff against feature Fn." If `[FROM_DIAGNOSIS]` is true, add instead: "read
     `doc/diagnosis-<slug>.md` and check against its §7 *Proposed fix*, including any design choice
     §7 explicitly left to this session."
  2. **Quality** — only if spec passes — is it well-built (tests real, no obvious smell)?
  3. **Over-engineering** *(always)* — walk the ladder top-down over the diff, stop at the first
     rung that applies, and return a **delete-list**: speculative abstractions, unrequested
     flexibility, reinvented stdlib/deps, dead scaffolding.
  4. **Design compliance** *(only when the subtask is UI and `doc/design-brief.md` exists)* — "read
     `doc/design-brief.md`. Every color, font size, spacing and radius in the diff must be one of
     its tokens, and components must be its reusable ones."

**Return contract** — this is all that enters this session, so keep it tight:

```
VERDICT: PASS | FAIL
FAILURES:    one line each, or "none"
DELETE-LIST: path:line — reason, or "clean"
NO-DIFF REVIEWED: comma-separated paths, or "none"
```

`FAILURES` lines must name the **concrete offending values**, not just which check failed. 5b.4's
escalation depends on it: you can only tell the user "add `#3B82F6` to `design-brief.md` first,
don't hardcode it" if the reviewer reported the hex. A bare "design check failed" kills that path —
and if the value is genuinely needed, adding it to the brief is the fix, never hardcoding.

#### Where the reviewer runs `git diff`

**Gitignored paths — every dispatch mode.** In the same checkout the reviewer will inspect, run
`git check-ignore -q <path>` for every scoped path and read its exit code directly. Branch on exit
`0` specifically: it means ignored; exit `1` means not ignored; exit `128` is an error, so stop and
report it rather than dispatching a review with an unverified scope. In worktree mode, use
`git -C <worktree abs path> check-ignore -q <path>` instead.

When any scoped path is ignored, `git diff -- <path>` returns nothing and a reviewer that trusts it
can return `VERDICT: PASS` without reviewing that file. Name ignored paths separately in the payload
as `NO-DIFF PATHS (gitignored)`, exclude them from the diff-only path list, and tell the reviewer to
read each one directly and apply all four checks to the current whole file. The reviewer must list
every such path on the return contract's `NO-DIFF REVIEWED` line; that disclosure is not a
failure. A whole-file review checks the current file, not the change, and the report must not imply
that a diff was reviewed. If every scoped path is ignored, omit the diff command entirely; never run
`git diff --` with an empty path list.

**Worktree mode** — the subtask's edits live in its own checkout, and a subagent starts in THIS
session's cwd, not there. A bare `git diff` would come back empty and the reviewer would return
`VERDICT: PASS` on nothing. Give it the absolute worktree path and have it use the `-C` form, the
same way 5d stages:

```bash
git -C <worktree abs path> diff -- [exact files this worker reported]
```

No baseline problem here: the checkout is isolated and 5b runs before that worktree commits, so its
working-tree diff is exactly this subtask's work.

**Single-tree, commit mode** — 5d commits each subtask as it passes, so HEAD advances and a plain
`git diff -- [files]` is already scoped to the current subtask.

**Single-tree, `[NO_COMMIT]`** — nothing commits, so HEAD never moves and the working tree
accumulates every subtask's changes. File scoping alone does not save this; 5d.3's collision rule
exists precisely because two subtasks touching one file is expected. For any file in scope that an
**earlier** subtask in this run also reported, append to the payload:

```
Out of scope — <path> also carries changes from an earlier subtask in this run
("<that subtask's goal>"). Review only the hunks belonging to THIS subtask, and never
delete-list the earlier work.
```

Omit that line when there is no overlap.

### 5c. Conditional routing loop (bounded)

If any **gate** (5a) or **review check** (5b) FAILS, re-dispatch the SAME worker with the specific
failures as an error log, and re-run 5a–5b. A non-empty delete-list counts as a review failure —
feed it back as the error log ("delete these, keep tests + success-criterion files") through this
same loop; do not open a separate one.

**Re-dispatch means *resume*, not respawn.** Send the failures to the agent that is already alive —
in Claude Code, `SendMessage` with the `agent_id` recorded in Phase 4. Its context is intact,
including every file it read and every edit it made, so you send only the error log, not the
payload. Spawn a fresh `Agent` **only** if that agent is gone; that path re-sends the whole payload
and the worker has to re-read everything it already read.

Under `[BACKEND] = codex`, "resume" means `codex exec` with the session id — see
[`references/codex-backend.md`](references/codex-backend.md) §C6 for the exact argument order (`-s` and `-C` go **before** the
`resume` subcommand, which rejects them) and the fallback for a session that cannot be resumed. It is
never `SendMessage`. Resume the subtask's `agent_id` — the only session id this run stores.

**Bounded loop: at most 3 iterations per subtask.** If still failing after 3, stop the loop and
report it to the user — do not loop further.

**Exception — secrets:** a secrets finding (5a gate 5) does NOT enter this loop. It hard-blocks the
subtask immediately and waits for the user to remove the secret (see 5a). Retrying cannot fix it.

Handle worker statuses:
- **NEEDS_CONTEXT** → provide the missing context and re-dispatch.
- **BLOCKED** → assess: more context (re-dispatch same model), more reasoning (re-dispatch
  capable model), too large (split), or plan is wrong (escalate to user). Never silently retry
  unchanged.

### 5d. Commit on pass

**If `[NO_COMMIT]` is true, skip committing entirely** — just mark the subtask complete and leave
its changes in the working tree.

**Worktree mode:** follow [`references/worktree-mode.md`](references/worktree-mode.md) (5d + 5e).

**Single-tree mode**: the MAIN session commits, and **commits are serialized — never concurrent**:

1. Process passing subtasks **one at a time**, in completion order. Two `git commit`s never run at
   once (the working tree + index are shared mutable state).
2. Stage **only the files that THIS subtask's worker reported changing** — never `git add -A`.
   Under `[BACKEND] = codex` that is the reconciled list from [`references/codex-backend.md`](references/codex-backend.md) §C5,
   which is also why `graphify-out/` scratch files never reach the index:
   ```bash
   git add [exact files this worker reported]
   git commit -m "[concise subtask summary]"
   ```
3. **Collision rule:** if two parallel subtasks both reported edits to the *same* file, do not split
   them — collapse those subtasks into ONE combined commit covering both file lists, so a single
   file is never half-committed across two commits.

End every commit message with:
```
Co-Authored-By: Claude <noreply@anthropic.com>
```

Record the commit SHA(s) in the subtask state. Mark the subtask complete.

---

## PHASE 6: Synthesis & Release *(Synthesis agent)*

When all subtasks are complete:

1. Collect the approved outputs, strip intermediate logs, and print a clean summary (below).
   `/forge-orchestrate` does **not** tag — releases/tags stay a manual step you run when you want one.
2. Update `doc/progress.txt` with the current status.
3. Append to `doc/changelog.txt` (`Date | Change | Description`, matching contextmap's format).
   Steps 2–3 write test results into files, so 5a's rule binds here: the counts you write are the
   ones 5a's own command printed, never a worker's or a reviewer's report of them.
4. If `doc/task-list.md` exists → read [`references/task-list-formats.md`](references/task-list-formats.md) and follow its *Phase 6*
   section to tick what the gates actually verified. Skip silently if the file does not exist.
5. **Generate the CD release-readiness report** — read [`references/release-readiness.md`](references/release-readiness.md) and write
   `doc/release-readiness.md` as it specifies.
6. Print the final report (under `[NO_COMMIT]`, replace the Branch/Commits block with
   `Mode: --no-commit — changes left in working tree for you to review and commit`):
   ```
   ✅ /forge-orchestrate complete — [TASK]

   Branch:  [BRANCH]
   Commits: [N]
     [sha] — [subtask summary]
     ...

   Files changed:
     [path] — [one-line what/why]
     ...

   CI gate results:
     lint        pass
     tests       [pass / unverified — no test tooling]
     coverage    [pass / skipped — no coverage tooling]
     dep-vuln    [pass / N findings]
     secrets     pass
     SAST        [pass / skipped — semgrep not installed]
     over-eng    [clean / N deleted]

   CD readiness: see doc/release-readiness.md (deploy/monitoring/rollback need your platform).

   On branch [BRANCH] — run /forge-merge when you're happy to land it and clean up.
   ```
   If `[FROM_DIAGNOSIS]` is true, add one line naming the source: `Fixes: doc/diagnosis-<slug>.md`.
   If `[BACKEND] = codex`, add two lines above `Files changed:`:
   ```
   Backend:     codex
   Council:     1 call — [verified | flawed, all findings incorporated | unresolved | degraded, council unusable]
   ```
   When anything went unresolved, follow it with the objection and its risk, in Codex's terms, not
   softened:
   ```
   Unresolved:  [subtask] — [Codex's objection] (risk: [what it says breaks])
   ```
   Never write "verified" for a call that failed to answer, and never describe a run that ended
   unresolved as agreed.

   Add, for any subtask where Codex's `FILES CHANGED:` list disagreed with the git delta (§C5),
   an `unclaimed changes:` line naming those paths. Never suppress it — it is either build output
   the user should gitignore, or scope creep.
7. Clean up — follow **6z** below.

### 6z. Scratch cleanup — reachable from every exit, not only from a finished run

Phase 6 used to be the only place this happened, so every early exit leaked scratch. Call this step
from **all** of them: the C1.1 no-`codex`-binary stop, a model-unavailable stop, the council's unresolved
approval stop, a dirty-tree abort, the no-test-runner abort, a secrets hard block, a gate that failed
3×, and the normal end of Phase 6.

**What to delete:** `graphify-out/.orchestrate_slice.py`, plus every `graphify-out/.orchestrate_*`
scratch file **this run** created — payloads, snapshots, and under `[BACKEND] = codex` this run's
whole `.orchestrate_council_<run_id>/` directory. Never delete by a bare `.orchestrate_*` glob and
never touch another `run_id`'s directory: a second `/forge-orchestrate` may be running in another
session and owns its own files.

**Two exits behave differently:**

- **The council's approval stop is a pause, not an abort — keep everything.** The user is expected to
  answer and continue, and the payloads, slices and round results are the state that continuation
  needs. Deleting them there turns a question into a restart.
- **A C1.1 preflight stop needs no cleanup at all** — it fires before the run has written anything.

**Worktree mode:** follow the cleanup section of [`references/worktree-mode.md`](references/worktree-mode.md). Note it does not
cover the exits above: a gate or secrets failure after Phase 4 leaves a worktree holding uncommitted
work, and that must be **reported and left in place**, never silently removed.

---

## NOTES

- **This command commits** (feature branch + per-subtask commits) — a deliberate reversal of the old
  "never commit, history stays yours" rule. It never tags and never merges to `main` — you own
  releases and the merge. Pass `--no-commit` to run the full pipeline with zero git writes.
- It never rebuilds the graph and never auto-runs `/forge-contextmap`. If `graphify-out/graph.json` is
  missing it hard-stops and asks you to run `/forge-contextmap` manually.
- **A `/forge-diagnose` handoff runs straight through** — `/forge-orchestrate doc/diagnosis-<slug>.md`
  reads the root cause, proposed fix, verification command, blast radius, and ruled-out branches out
  of the handoff instead of re-deriving them, and skips the ambiguity scan the diagnosis already
  answered.
- **Parallel batches use git worktrees** — 2+ independent subtasks each get an isolated checkout on
  a sub-branch off the feature branch, commit there in parallel, then merge back serialized. Real
  isolation: overlapping edits surface as a visible merge conflict instead of a silent index
  collision. Lone/dependent subtasks and `--no-commit` stay single-tree.
- **Gates are adaptive and honest** — a repo with no lint/test/scan tooling gets `skipped — no
  tooling detected`, never a fake green check.
- **Local CI only.** The CD half of the checklist (deploy, canary, monitoring, rollback, DAST) is
  reported in `doc/release-readiness.md`, not executed — it requires a real CI/CD platform.
- The graph slice keeps each worker's context small regardless of repo size; that is the point.
- **Over-engineering review is a built-in gate** — no plugin needed. The minimal-code ladder is
  injected into every worker (Phase 3c) so they write lean at generation time, and each diff is
  reviewed for bloat before commit (5b.3), producing a delete-list routed through the same bounded
  loop. Tests and success-criterion files are always exempt. *Ladder adapted from ponytail
  (dietrichgebert/ponytail, MIT).*
- **Shared blocks** — the `<!-- forge:shared-block ... -->` markers wrap text that is duplicated in
  other forge skills on purpose (each skill installs standalone). Edit one copy, update the rest;
  the README's *Shared blocks* table lists every location.
