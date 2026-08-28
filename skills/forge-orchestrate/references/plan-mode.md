# Plan mode — what `/forge-orchestrate` does when it may not write

Read this only when plan mode is active. It replaces Phases 1b through 6 for the duration of the
planning pass; Phase 0 still runs as written.

<!-- forge:shared-block plan-mode -->
**Governing rule — the plan describes the work and the decisions, never the machinery that will
execute it.** The machinery is whatever executes *this* plan: this skill's own stage numbers —
whichever spelling it uses, `PHASE 3` or `STEP 1` or `0a` — plus gate names, worktrees, the
planning council, slice scripts and subagent dispatch. None of it belongs in the plan, except as at
most one line under *How it runs*. Naming the subject matter is a different thing and stays
allowed: a plan whose subject *is* a phased skill still names the sections it edits. A summary of
this skill's own pipeline is not a plan — it is the wall of words the user cannot read.

**These six headings are the harness's own slots, not a second set layered on top.** Where the
plan-mode instructions ask for a Context section, a recommended approach, the critical files named,
and a verification section: *Context* is that section, *What changes* is the approach and its
*Files* column is the critical-files requirement, and *Verify* is the verification section. Emit
one shape, never both — two heading sets compounding is what produces the wall of words.

**The same holds for the harness's process.** Its plan workflow prescribes Explore agents and then
a Plan agent before writing. This skill's read-only steps *are* that exploration, already scoped by
the graph and by the skill itself — so do not spawn agents to re-derive what you have just read.
Dispatch one only for a question those steps genuinely left open.

Write the plan file with exactly these headings, in this order:

```
## Context
1–2 sentences: what was asked and why it needs doing.

## What changes
| # | Change | Done when | Files |
One row per unit of work. One line per row — never wrap a cell.

## Decisions I made for you
One line each: the choice, and the alternative rejected. Write "none" if there were none.

## How it runs
At most 4 lines total: branch/commits, gates, parallelism, what stays untouched.

## Verify
The exact commands that prove it worked.

## Not doing
Explicit out-of-scope list.
```

Length: as short as it can be while staying detailed and easy to understand, and no longer. There
is no word count — prefer a table to prose, and cut any sentence that repeats what a table already
says, but never cut evidence or a decision to hit a length. The governing rule above already bans
what actually makes these plans long. Never restate the request back at the user. Anything the user
has to decide goes under *Decisions I made for you* — never buried in prose, where it is missed.

Two procedural rules:

- **ExitPlanMode is the approval.** Do not ask a second confirmation question in chat before or
  after it. Where this skill has its own approval checkpoint, that checkpoint's content becomes the
  plan body and ExitPlanMode asks its question.
- **On approval, resume at the phase named below** and re-run anything the read-only pass could
  only approximate.
<!-- /forge:shared-block plan-mode -->

## What runs, and what cannot

Run, in order: **0a** (the graph hard-stop — still the cheapest stop and still correct), **0d**
(resolve the request, including a diagnosis handoff), **0e** (the ambiguity scan — ask here, not in
the plan), **0f** (derive `[BRANCH]`, and only derive it: creating a branch is a write), then
**1a**'s decomposition rules.

Everything else is blocked because it writes:

- **Phase 3a cannot slice.** The slice script is written to `graphify-out/.orchestrate_slice.py`
  before it runs. Under `[BACKEND] = codex` the council (1b) is blocked for the same reason — it
  writes its schema, payloads and results into `graphify-out/.orchestrate_council_<run_id>/`.
- **Phase 2** creates no branch, **Phase 4** dispatches no worker, **Phase 5** commits nothing, and
  **Phase 6** writes no `progress.txt`, `changelog.txt` or `release-readiness.md`.

## Filling the plan

- **What changes** is the 1a decomposition, one subtask per row. The *Files* column comes from
  `graphify-out/GRAPH_REPORT.md` plus grep, not from a slice — so **mark that column `unsliced`**
  and add one line under *How it runs*: `File lists are unsliced; Phase 3a re-slices
  authoritatively before any worker is dispatched.` The hazard is the one Phase 3a already names
  for the council's advisory slices: a list computed before the branch exists can describe a tree
  nobody is editing.
- **Decisions I made for you** carries whatever 0e resolved, the derived branch name, and — from a
  diagnosis handoff — the §7 fix direction you adopted.
- **How it runs**: the branch name, that it commits per subtask, `--no-commit` if set, and the
  gates you expect to run. Four lines, no gate table.
- **Verify** is each subtask's success criterion, as commands.

## On approval

Resume at **0b** (resolve `[PYTHON_CMD]`), then Phase 2 onward exactly as written. Re-run 3a's
slice for every subtask and overwrite the plan's unsliced file lists — they were an estimate, and
Phase 3a is authoritative. Under `[BACKEND] = codex`, run the council at 1b before Phase 2: it was
never dispatched, so nothing has critiqued this decomposition yet.
