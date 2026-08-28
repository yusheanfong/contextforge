# Plan mode — what `/forge-contextmap` does when it may not write

Read this only when plan mode is active. STEP 1's mode detection runs unchanged; this file says how
far into the detected mode you may go, and what the plan contains.

<!-- forge:shared-block plan-mode -->
**Governing rule — the plan describes the work and the decisions, never the machinery that will
execute it.** Phase numbers, gate names, worktrees, the planning council, slice scripts and
subagent dispatch do not belong in the plan, except as at most one line under *How it runs*. A
summary of this skill's own pipeline is not a plan — it is the wall of words the user cannot read.

**These six headings are the harness's own slots, not a second set layered on top.** Where the
plan-mode instructions ask for a Context section, a recommended approach, the critical files named,
and a verification section: *Context* is that section, *What changes* is the approach and its
*Files* column is the critical-files requirement, and *Verify* is the verification section. Emit
one shape, never both — two heading sets compounding is what produces the wall of words.

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

Budget: prose outside the tables stays under 200 words. Never restate the request back at the
user. Anything the user has to decide goes under *Decisions I made for you* — never buried in
prose, where it is missed.

Two procedural rules:

- **ExitPlanMode is the approval.** Do not ask a second confirmation question in chat before or
  after it. Where this skill has its own approval checkpoint, that checkpoint's content becomes the
  plan body and ExitPlanMode asks its question.
- **On approval, resume at the phase named below** and re-run anything the read-only pass could
  only approximate.
<!-- /forge:shared-block plan-mode -->

## How far each mode gets

The discriminator is whether the mode has read-only work whose result is worth showing. Two modes
do, two do not, and pretending otherwise produces a plan that says "I will run the thing."

| Mode | Runs under plan mode | The plan is |
|---|---|---|
| NEW PROJECT | The interview — it is `AskUserQuestion`, which writes nothing | The scaffold file list, plus every design decision the interview settled |
| MIGRATION | The v1 `doc/` survey — reading every file to classify it is read-only | The v1 → v2 file mapping, one row per file, naming what moves where |
| SYNC | Nothing past detection. S1's interpreter probe is read-only, but S2.5's prune is a Python file written to disk | The fences that will be regenerated, and the bloat signal it cannot compute yet |
| EXISTING PROJECT | Nothing past detection. The graph has to be built before E5 has anything to present | The file list it will create |

For SYNC and EXISTING PROJECT, print one line saying the analysis needs to write and run a script,
so the plan is a file list rather than findings. That is honest. Silently emitting a thin plan and
letting the user assume it was analyzed is not.

## Filling the plan

- **What changes** — one row per file created or overwritten. Name them exactly; this is the whole
  value of the plan for a scaffolding command.
- **Decisions I made for you** — the detected mode and which STEP 1 check selected it, plus every
  answer the NEW PROJECT interview settled.
- **How it runs** — one line naming the fence contract: `Writes only inside graphify:auto fences;
  everything outside them is left verbatim.`
- **Not doing** — the files it will not touch: `doc/prd.md`, `doc/task-list.md`,
  `doc/diagnosis-*.md`.

## On approval

Resume at the detected mode's first writing step and follow its reference file exactly. Nothing in
the plan replaces those steps — the mode still runs in full.
