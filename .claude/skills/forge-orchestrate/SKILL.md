---
name: forge-orchestrate
description: CI/CD-gated autonomous feature pipeline (/forge-orchestrate). Use when building a feature end-to-end against the knowledge graph — decompose, branch, dispatch graph-scoped worker subagents, run lint/test/secrets/over-engineering gates, commit per subtask. Also use to execute a /forge-diagnose handoff. Triggers include "/forge-orchestrate", "orchestrate this feature", "build the next task", "run the pipeline", "execute the diagnosis".
argument-hint: "[feature | doc/diagnosis-*.md] [--no-commit]"
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

**Roles (mapped from the multi-agent blueprint):** Coordinator = intent-parse + decompose · Engineer
= worker subagent that edits code + writes tests · QA/Critic = the CI gate runner + review loop ·
Synthesis = final report + release-readiness.

**Global state object** — track these across phases and surface them in the final report:
`branch`, `subtasks[]` (goal, criterion, deps, independent, gate_set, status, commit_sha),
`gate_results`, `files_changed`, `commits[]`, `all_gates_pass`.

**Reference files** — read each only when its branch applies, not up front:

| File | Read when |
|---|---|
| `references/graph-slice.md` | Phase 3a — always (the slice script lives there) |
| `references/task-list-formats.md` | `doc/task-list.md` exists (Phase 0d + Phase 6) |
| `references/worktree-mode.md` | a batch has 2+ independent subtasks AND `[NO_COMMIT]` is false |
| `references/release-readiness.md` | Phase 6 |

---

## PHASE 0: Intent Parsing & Clarification *(Coordinator / Router)*

### 0a. Graph prerequisite — hard-stop

<!-- forge:shared-block graph-hard-stop -->
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

<!-- forge:shared-block python-cmd -->
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

First, parse flags out of `$ARGUMENTS`: if it contains **`--no-commit`**, set `[NO_COMMIT] = true`
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
   - If `doc/task-list.md` exists → read `references/task-list-formats.md` and follow its
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
(clarify-then-run):

```
Plan for [TASK] (branch [BRANCH]):

Subtask 1: [goal]
  success: [criterion]   depends on: none   independent: yes   gates: lint,test,audit
Subtask 2: [goal]
  success: [criterion]   depends on: 1      independent: no    gates: lint,test
...

Running now. I'll stop only if a request is ambiguous or a gate fails 3×.
```

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

2. **Create + checkout the branch:**
   ```bash
   git checkout -b [BRANCH]
   ```
   If `[BRANCH]` already exists, check it out and note it.

3. **Merge-conflict pre-check** vs the base branch (checklist: "automated merge conflict detection"):
   ```bash
   git fetch
```
```bash
git merge-base --is-ancestor origin/main HEAD
   ```
   If the branch base is behind `main` / a conflict looks likely, print a one-line warning. Do not
   block.

---

## PHASE 3: Per-Subtask Context Assembly *(RAG — the core)*

For each subtask, build an **isolated context payload**. This payload is the ENTIRE context the
worker will see — it never inherits this session's history.

### 3a. Graph slice

Read `references/graph-slice.md` and follow it exactly. It writes the slice script once, runs it per
subtask with the subtask's key terms, and returns the touched source files + node labels + key
edges. `NO_MATCH` means the graph has no node for these terms — fall back to the universal docs only
(3b) and tell the worker the graph had no specific match.

### 3b. Doc slice

From the `FILES` the slice returned, pick the matching domain docs and ALWAYS add the universal
docs. Read only these — never the whole `doc/` set:

<!-- forge:shared-block source-doc-map -->
- **Always:** `doc/architecture.md`, `doc/solution-structure.md`, `doc/coding-standard.md`, and
  `doc/prd.md` if it exists (small — it's the scope guard for spec compliance)
- UI/screen/widget/view/component sources → `doc/design-brief.md` + `doc/app-flow.md`
  (v1 fallback: `doc/ui-guideline.md` if the project hasn't migrated)
- controller/service/handler/endpoint/route/api sources → `doc/api-contract.md`
  + `doc/backend-schema.md` if it exists
- entity/model/enum/domain sources → `doc/domain-model.md` + `doc/backend-schema.md` if it exists
- auth/token/permission/role sources → `doc/security.md`
<!-- /forge:shared-block source-doc-map -->

### 3c. Instruction

Assemble the worker prompt from: the subtask **goal** + **success criterion** + the graph slice
(3a) + the doc slice text (3b) + these constraints, verbatim:

```
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
Also:
  - State any assumption you're making; if the subtask is genuinely unclear, report NEEDS_CONTEXT
    rather than guessing.
  - Surgical changes only — every changed line traces to THIS subtask. Don't refactor, reformat, or
    "improve" adjacent code; match the existing style even if you'd do it differently.
  - The success criterion is the definition of done — verify it (run the test/check), don't assume.
GUARD: the ladder never applies to the tests Phase 4 mandates or to any file needed to satisfy the
success criterion. Those are always required — never skip them as "YAGNI."
```

> The ladder above is `forge:shared-block minimal-ladder`, in its **payload copy**. It must stay
> literal text here — a worker subagent sees only its dispatch payload, never this file, so it can
> never become a pointer to somewhere else.

**If `[RULED_OUT]` exists** (diagnosis handoff), append it to the payload as: "Already ruled out by
the diagnosis — do not re-investigate: [list]."

---

## PHASE 4: Execution *(Worker = Engineer agent)*

Use the Agent tool. **Default `subagent_type` is `general-purpose`** (the documented catch-all);
use `Explore`/`Plan` only for read-only research subtasks.

Model selection: cheap/fast model for mechanical 1–2 file edits with a clear spec; a more
capable model for multi-file integration or design judgment.

Each worker receives ONLY its payload from Phase 3 — not this session's history. Workers edit
code + write tests + run them, and do **NOT** commit. Committing happens in Phase 5 after gates pass.

### Dispatch mode — worktrees vs. single tree

Decide per batch of sibling subtasks:

- **Worktree mode** — a batch has **2+ independent (parallel) subtasks** AND `[NO_COMMIT]` is false.
  Read `references/worktree-mode.md` and follow it for dispatch, commit (5d), merge-back (5e), and
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

### 5b. Review checks

1. **Spec compliance** — does the worker output do exactly what the subtask asked (nothing
   missing, nothing extra)? If the subtask carries a `Builds: Fn` reference, check the diff
   against that feature's line in `doc/prd.md`. If `[FROM_DIAGNOSIS]` is true, check the diff
   against the handoff's §7 *Proposed fix* instead — including any design choice §7 explicitly
   left to this session.
2. **Quality** — only after spec passes — is it well-built (tests real, no obvious smell)?
3. **Over-engineering** *(always runs)* — an over-engineering review pass on THIS worker's diff:
   flag speculative abstractions, unrequested flexibility, reinvented stdlib/deps, and dead
   scaffolding, and hand back a **delete-list**. Review criteria = the minimal-code ladder (walk it
   top-down, stop at the first rung that applies):
   <!-- forge:shared-block minimal-ladder -->
   1. Does this need to exist? → shouldn't (YAGNI)
   2. Already in this codebase? → should have reused it
   3. Stdlib does it? → should have used it
   4. Native platform feature? → should have used it
   5. Installed dependency? → should have used it
   6. One line? → should be one line
   7. Only then: the minimum that works
   <!-- /forge:shared-block minimal-ladder -->
   **Run it from the MAIN session by default** (it can read this whole file, including the ladder).
   If you instead dispatch a read-only reviewer subagent, its payload MUST restate the ladder above
   as the review criteria — a subagent sees only its dispatch payload, not this file.
   **Scope guard — never delete-list these:** the tests Phase 4 mandates, and any file needed to
   satisfy the subtask's success criterion. Minimality never overrides the "workers must write
   tests" rule.
4. **Design compliance** *(UI subtasks only — skip when `doc/design-brief.md` doesn't exist)* —
   scan the diff for style values: every color, font size, spacing, and radius must come from
   `doc/design-brief.md` tokens, and components must be the brief's reusable ones. An ad-hoc hex
   value, magic font size, or one-off component = review FAILURE routed through the same bounded
   loop (5c) with the offending values listed. If the value is genuinely needed, the fix is to add
   it to `design-brief.md` first (surface that to the user), not to hardcode it.

Either dispatch a reviewer subagent (read-only) with the criterion + the worker's diff, or
review directly.

### 5c. Conditional routing loop (bounded)

If any **gate** (5a) or **review check** (5b) FAILS, re-dispatch the SAME worker with the specific
failures as an error log, and re-run 5a–5b. An over-engineering delete-list (5b.3) counts as a
review failure — feed it back as the error log ("delete these, keep tests + success-criterion
files") through this same loop; do not open a separate one.

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

**Worktree mode:** follow `references/worktree-mode.md` (5d + 5e).

**Single-tree mode**: the MAIN session commits, and **commits are serialized — never concurrent**:

1. Process passing subtasks **one at a time**, in completion order. Two `git commit`s never run at
   once (the working tree + index are shared mutable state).
2. Stage **only the files that THIS subtask's worker reported changing** — never `git add -A`:
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
4. If `doc/task-list.md` exists → read `references/task-list-formats.md` and follow its *Phase 6*
   section to tick what the gates actually verified. Skip silently if the file does not exist.
5. **Generate the CD release-readiness report** — read `references/release-readiness.md` and write
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

   On branch [BRANCH] — merge it when you're happy.
   ```
   If `[FROM_DIAGNOSIS]` is true, add one line naming the source: `Fixes: doc/diagnosis-<slug>.md`.
7. Clean up:
   - Remove `graphify-out/.orchestrate_slice.py`.
   - **Worktree mode:** follow the cleanup section of `references/worktree-mode.md`.

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
