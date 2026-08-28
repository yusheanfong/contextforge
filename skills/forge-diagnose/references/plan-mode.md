# Plan mode — what `/forge-diagnose` does when it may not write

Read this only when plan mode is active. PHASE 0 through PHASE 5 run unchanged; this file replaces
PHASE 5's presentation format and defers PHASE 6.

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

## Reproduction still happens

PHASE 2 runs in full. Plan mode forbids edits, and running the program, the failing test, a probe
script in the system temp dir or `curl` against an endpoint is not an edit — the CONTRACT already
says the caches those runs drop (`__pycache__`, `.pytest_cache`, coverage files) do not count as
modifications. Skipping reproduction would cap every root cause at `[Likely]` and strip the
diagnosis of most of its value, for no safety gained.

The confidence ceiling is set by PHASE 2's outcome, exactly as written: reproduced means the root
cause may reach `[Certain]`; not reproduced means `[Likely]` at best, and the plan says so.

## The two approval gates collapse into one

PHASE 5 asks the user to confirm, redirect, or go deeper. ExitPlanMode asks the same question.
**Asking both is the duplication that makes these plans unreadable.** So: PHASE 5's content becomes
the plan body, and ExitPlanMode is the confirmation. Do not print a checkpoint in chat and then
write a plan repeating it.

If the user redirects instead of approving, loop back to PHASE 3 or PHASE 4 with the new
information and rewrite the plan file.

## Filling the plan

- **Context** — the symptom, and whether it reproduced.
- **What changes** — the proposed fix, one row per file that has to change. If the fix is one edit,
  it is one row; do not pad it.
- **Decisions I made for you** — the root cause with its confidence tag, and the ruled-out branches
  with the evidence that killed each one. This is the highest-value part of the diagnosis; it does
  not get compressed away.
- **How it runs** — one line: `Writes doc/diagnosis-<slug>.md only. No code changes, no commits.`
- **Verify** — §9's before/after commands, verbatim.
- **Not doing** — the open questions, stated as unknowns.

Evidence stays as `path` → `symbol`. An unsourced claim does not enter the plan, for the same
reason it does not enter the handoff.

## On approval

Run PHASE 6: fill [`references/handoff-template.md`](handoff-template.md) from the investigation and write
`doc/diagnosis-[SLUG].md`. The plan file is not the handoff — it is scoped to this session, and the
handoff exists to be pasted into a fresh one.
