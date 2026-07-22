---
name: forge-diagnose
description: Root-cause diagnosis that never changes code, ending in a cross-session handoff prompt (/forge-diagnose). Use when something is broken and the fix will be executed in a different session — a bug, test failure, crash, regression, or unexpected behavior that needs investigating before any code is touched. Triggers include "/forge-diagnose", "diagnose this bug", "find the root cause", "why is this failing", "investigate before fixing".
allowed-tools: Read, Grep, Glob, Bash, AskUserQuestion, Write
---

# /forge-diagnose — Diagnose Without Changing Code + Cross-Session Handoff

A discussion-driven investigation session that never touches code. It ends by writing a **handoff
prompt** — a self-contained instruction the *next* session pastes in and executes without
re-exploring the codebase.

Diagnosis and execution split across two sessions: this one finds the root cause, the next one
fixes it.

Reads `$ARGUMENTS` as the issue to diagnose.

## CONTRACT (non-negotiable guarantees)

- **This session never modifies code.** Enforced by this CONTRACT, *not* by the tool sandbox —
  `Write` is permitted (the handoff has to be written) and `Write` can reach any path. `Edit` is
  omitted from `allowed-tools` as friction, not as a lock. The guarantee is behavioral: honor it.
- `Write` targets exactly one path: `doc/diagnosis-<slug>.md`. Nothing else, ever.
- **Bash investigates actively — it just can't change code.** Allowed and encouraged: run the
  program, run the test suite, hit endpoints with `curl`, read/tail logs, toggle env vars for a
  run, add temporary verbose/debug flags on the command line, write throwaway probe scripts to the
  system temp dir (never into the repo). Caches and build artifacts these runs drop
  (`__pycache__`, `.pytest_cache`, coverage files) don't count as modifications.
  Banned: editing any repo file, `git commit` / `push` / `checkout` / `stash` / `reset`,
  installing/removing packages, schema migrations, deleting data, anything destructive or
  irreversible.
- No fix is proposed before PHASE 4 has a root cause with evidence behind it.
- **No guessing — anywhere.** A guess is a claim you could verify but didn't. Every "probably"
  must be converted before it enters the handoff: read the code path, run the program or a probe
  to observe the real behavior, or ask the user. Every claim carries `[Certain]` (verified: reproduced, or read directly in the
  code) or `[Likely]` (inference from evidence you can cite — name the evidence). `[Guessing]`
  never appears in a handoff: if that's the honest tag, the diagnosis isn't done — keep
  investigating, or record it as an **open question**, clearly separated from findings.
- Unlike `/forge-orchestrate` and `/forge-audit`, this command does **not** require
  `graphify-out/graph.json`. It uses the graph when present and greps when not — a bug is never
  blocked behind a graph build.

---

## PHASE 0: Frame

1. `$ARGUMENTS` is the issue. If empty, ask the user what's broken and stop until answered.
2. Check whether `graphify-out/graph.json` exists → set `[HAS_GRAPH]`. Do not hard-stop either way.
3. Derive `[SLUG]` from the issue: kebab-case, ≤5 words (e.g. `checkout-500-on-empty-cart`).

## PHASE 1: Interrogate Before Exploring

Close the gaps that make exploration cheap **before reading a single source file**. Ask whatever
`$ARGUMENTS` left open — AskUserQuestion when the answer is a choice (which environment? every
time or intermittent?), plain chat when it's free-form (paste the stack trace):

- Exact error text / stack trace
- Expected behavior vs actual behavior
- Reproduction steps — and does it happen every time?
- When did it start? Did it ever work?
- Where does it happen — local, CI, or production?

Skip anything already answered in `$ARGUMENTS`. Never re-ask what the user already told you.

## PHASE 2: Reproduce

1. Trigger the issue: run the program, the failing test, or `curl` the endpoint. Tail the relevant
   logs while it happens. Record the **shortest decisive output** verbatim — do not paraphrase an
   error.
2. Find what changed:
   ```bash
   git log --oneline -20
   git diff HEAD~1 --stat
   ```
   Narrow to suspect paths once PHASE 3 has candidates.
3. Set the confidence ceiling honestly:
   - **Reproduced** → the root cause may reach `[Certain]`.
   - **Not reproduced** (production-only, timing-dependent, environment-specific) → say so plainly.
     Everything downstream is `[Likely]` at best. Do not dress a hypothesis up as a finding.

## PHASE 3: Locate & Trace

1. If `doc/` exists, read `doc/architecture.md` and `doc/solution-structure.md` first — the map
   `/forge-contextmap` already built. Reuse it instead of rebuilding it.
2. If `[HAS_GRAPH]`, read `graphify-out/graph.json` to scope which modules can plausibly be
   involved, then read only those files. Otherwise grep outward from the error string or symptom.
3. Trace **backward**: where does the bad value originate? What called this with it? Keep walking up
   until you reach the source. The fix belongs at the source, not at the symptom.
4. Record evidence for every step as `path` → `symbol` (function/class/method), with a line number
   only as a hint. An unsourced claim does not go in the handoff.

## PHASE 4: Hypothesize & Rule Out

1. Write ranked hypotheses in the form: *"X is the root cause because Y."*
2. Test the cheapest discriminator first — run the program, a test, a probe script, `curl`, a log
   read. One variable at a time. A hypothesis stays a hypothesis until a discriminator confirms or
   kills it — it never enters the handoff as a finding on plausibility alone. Can't discriminate
   without changing code? Ask the user to run/check the thing, or park it as an open question. Do
   not promote it.
3. **Record every eliminated branch together with the evidence that killed it.** This is the
   highest-value part of the handoff — it's what stops the next session re-walking dead ends.
4. Three hypotheses dead → stop. Raise "this may be architectural, not a single bug" with the user
   rather than grinding out a fourth.

## PHASE 5: Discussion Checkpoint

This is what makes it a discussion, not a report. Present to the user:

- The root cause, with its confidence tag
- The evidence chain (`path` → `symbol`)
- What was ruled out and why
- Any open questions — unknowns you could not resolve without changing code, stated as unknowns
- The proposed fix direction

Then ask: confirm, redirect, or go deeper. **Do not write the handoff until the user confirms.** If
they reject it, loop back to PHASE 3 or PHASE 4 with the new information.

## PHASE 6: Emit Handoff

1. Read `references/handoff-template.md` (in this skill's directory, next to SKILL.md) and fill it
   from the investigation.
2. Write to `doc/diagnosis-[SLUG].md`. If that file already exists (earlier diagnosis of the same
   issue), Read it first, then overwrite — the old diagnosis is superseded, and Write refuses to
   overwrite a file you haven't Read. If `doc/` doesn't exist (non-ContextForge repo), Write
   creates it — do not scaffold the rest of the doc set and do not invoke `/forge-contextmap`.
3. Print exactly:
   ```
   Diagnosis complete → doc/diagnosis-[SLUG].md
   Open a fresh session and paste its contents to execute the fix.
   ```

---

## RED FLAGS — stop and go back to the phase you skipped

| Thought | Reality |
|---------|---------|
| "I'll just fix it quickly, it's one line" | This session does not fix. That's the whole point. |
| "It's probably X" — with no `path` → `symbol` behind it | A guess in the handoff sends the next session exploring anyway. Back to PHASE 3. |
| "The error is in `parse()`, that's the root cause" | That's where it surfaced. Trace backward to where the bad value was born. Back to PHASE 3. |
| "I'm confident enough, writing the handoff" | PHASE 5 confirmation is not optional. |
| Reaching for `Edit` | Not in `allowed-tools`, and not in scope. |
| "Can't easily check, I'll assume X" | Assuming IS guessing. Read the code, run a probe, or ask the user — those are the only three moves. |
| "It's [Likely] because it makes sense" | [Likely] requires citable evidence, not plausibility. No evidence = open question, not a finding. |
| "Fourth fix idea might work" | Three dead hypotheses = architectural question. Raise it with the user (PHASE 4.4). |
