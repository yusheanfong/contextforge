---
name: forge-merge
description: Land a reviewed local feature branch into main or master with a --no-ff merge, verify ancestry, safely delete merged branches, and report the push and undo commands. Triggers include "/forge-merge", "merge this branch back", "land the feature branch", "merge and delete the branch", "clean up the merged branch".
argument-hint: "[branch]"
allowed-tools: Bash, AskUserQuestion
---

# /forge-merge — Land a Verified Branch, Prove It Landed, Clean Up

Merges a finished feature branch into the repo's base branch, confirms it landed, then deletes it
and any sub-branches a parallel `/forge-orchestrate` run left behind.

`$ARGUMENTS` is the branch to land; with no arguments it lands the branch you are on, which is the
normal case. **This command commits** — one merge commit on the base branch, locally, never pushed.

## CONTRACT (non-negotiable guarantees)

- **Local only.** No `git push`, no `git fetch`, no remote writes. The report prints the push command
  for you to run when you want the merge to leave your machine.
- **Never force-deletes.** `git branch -d` only, never `-D`. Git's refusal to delete an unmerged
  branch is the last line of defence, and forcing past it is exactly how work disappears.
- **Never resolves a conflict for you.** Two real intents collided and only you know which wins.
  Conflict → abort, confirm the base is back where it started, report, stop.
- **Never touches your working-tree changes.** A dirty tree gets a question, never a silent stash,
  never a silent commit.
- **Never re-runs `/forge-orchestrate`'s gates.** They ran at commit time and you reviewed the branch.
- **Never prints a check it didn't run.** Every report line reflects a real command's exit status.
- **No graph needed.** This reasons about git refs, not code. It runs in any git repo.
- **One git command per Bash call**, so every command runs identically on macOS, Linux, and
  Windows — no `||` chaining, no heredocs, no `2>/dev/null`. Silence on a missing ref comes from
  `--quiet`, not a redirect.

**A non-zero exit is often the answer, not a failure.** `git rev-parse --verify --quiet`,
`git merge-base --is-ancestor`, and `git branch -d` all use exit status to *say something*. Read the
code; don't retry and don't work around it.

---

## PHASE 0: Resolve What Is Being Merged

### 0a.0 Plan mode

If you have been told to write your plan to a plan file and make no other edits, plan mode is
active. Plan mode runs 0a through 0e with exactly one command excepted — resolving the branch and
`[BASE]`, checking the tree, checking for leftover worktrees, and showing what is about to land is
exactly the content a merge plan needs. The exception is 0d's first command: `git worktree
prune` deletes stale administrative directories under `.git/worktrees`, so skip it and run `git
worktree list` alone. The leftover-worktree check still happens; only the pruning waits — and
because the prune exists so that a worktree already deleted from disk does not stop the run,
ignore any listed entry whose path is gone. That is stale metadata, not a leftover worktree; only
one whose directory is still present can hold `[BRANCH]`. Nothing past 0e runs: PHASE 1 merges,
PHASE 3 deletes.

<!-- forge:shared-block plan-mode -->
**Governing rule — the plan describes the work and the decisions, never the machinery that will
execute it.** The machinery is whatever executes *this* plan: this skill's own stage numbers —
whichever spelling it uses, `PHASE 3` or `STEP 1` — plus gate names, worktrees, the planning
council, slice scripts and subagent dispatch. None of it belongs in the plan, except as at
most one line under *How it runs*. Naming the subject matter is a different thing and stays
allowed: a plan whose subject *is* a phased skill still names the sections it edits. A summary of
this skill's own pipeline is not a plan — it is the wall of words the user cannot read.

**These six headings are the harness's own slots, not a second set layered on top.** Where the
plan-mode instructions ask for a Context section, a recommended approach, the critical files named,
and a verification section: *Context* is that section, *What changes* is the approach and its
*Files* column is the critical-files requirement, and *Verify* is the verification section. Emit
one shape, never both — two heading sets compounding is what produces the wall of words.

**The same holds for the harness's process.** Its plan workflow prescribes Explore agents and then
a Plan agent before writing. The steps this skill just ran *are* that exploration, already scoped,
so do not spawn those agents to repeat work already done. Where this skill states its own rule
about subagents, outside this block, that rule governs.

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

- **ExitPlanMode is the approval.** Do not ask a second confirmation question about *this plan*,
  in chat, before or after it. Where this skill has its own checkpoint covering the same ground,
  that checkpoint's content becomes the plan body and ExitPlanMode asks its question. A later
  checkpoint over content the plan could not have carried — something this skill only drafts after
  approval — is a different question and is still asked.
- **On approval, resume at the phase named below** and re-run anything the read-only pass could
  only approximate.
<!-- /forge:shared-block plan-mode -->

**Filling the plan.** *What changes* is one row per commit 0e listed, plus a row for each branch
that would be deleted. *Decisions I made for you* carries the resolved `[BASE]` and which probe
resolved it — guessing there merges into the wrong branch, so the user should see it. *How it runs*
is one line: `--no-ff merge on [BASE], then git branch -d. Local only: no push, no fetch.` *Verify*
is the ancestry check, `git merge-base --is-ancestor [BRANCH] [BASE]`.

A dirty tree or a leftover worktree still stops the run at 0c/0d. Do not fold either into the plan
as something the user will deal with later — they block the merge, and the plan cannot proceed
around them.

**On approval**, resume at PHASE 1 and run it as written.

### 0a. Resolve `[BRANCH]`

If `$ARGUMENTS` names a branch, that is `[BRANCH]`. Otherwise take the current branch:

```bash
git rev-parse --abbrev-ref HEAD
```

If the result is `HEAD`, you are on a detached HEAD — stop and say so; there is no branch to land.

Confirm the branch exists before going further:

```bash
git rev-parse --verify --quiet refs/heads/[BRANCH]
```

### 0b. Resolve the base branch as `[BASE]`

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

If nothing resolves — a repo on `trunk` or `develop` with no `origin/HEAD` to read it from — do not
improvise a base. List the candidates and ask with AskUserQuestion, excluding `[BRANCH]` itself:

```bash
git branch
```

Only stop outright if there is no other branch to merge into. If `[BRANCH]` and `[BASE]` are the
same, stop: you are already on the base branch and there is nothing to land.

### 0c. Dirty-tree check

```bash
git status --porcelain
```

If the tree is dirty, ASK — stash (`git stash push -u`) or abort. Uncommitted work mixes into the
conflict surface; recovery is worse than the wait.

### 0d. Leftover-worktree check

Clear stale metadata first, so a worktree already deleted from disk doesn't stop the run.

```bash
git worktree prune
```

```bash
git worktree list
```

If any worktree other than the main one has `[BRANCH]` checked out, report the exact paths and stop.
Git will not delete a branch checked out in a worktree, and catching it here is cheaper than
discovering it in PHASE 3 *after* the merge. Removing someone's worktree is not this command's call.

### 0e. Show what is about to land

```bash
git log --oneline [BASE]..[BRANCH]
```

```bash
git diff --stat [BASE]...[BRANCH]
```

**Three dots on the diff, never two.** `[BASE]...[BRANCH]` diffs against the merge base — the
branch's own changes. Two dots compares the tips, so everything `[BASE]` gained after the branch was
cut shows up as a difference. That is the common case, not an edge case.

Keep the commit count and the changed-file count; PHASE 4 needs them. Then continue without asking —
invoking `/forge-merge` after reviewing the branch *is* the confirmation.

**If the log is empty**, `[BRANCH]` is already merged or has no commits of its own. Skip PHASE 1 and
go to PHASE 2 — verification and cleanup are still what you want.

---

## PHASE 1: Merge

1. **Capture the undo anchor** — before the merge, or it is worthless:

   ```bash
   git rev-parse [BASE]
   ```

   Store as `[BASE_SHA]`.

2. **Check out the base:**

   ```bash
   git checkout [BASE]
   ```

3. **Merge with a real merge commit.** Summarize the feature in one line from 0e's commit log —
   what landed, not "merge branch X":

   ```bash
   git merge --no-ff [BRANCH] -m "merge [BRANCH]: [one-line summary]" -m "Co-Authored-By: Claude <noreply@anthropic.com>"
   ```

   Two `-m` flags rather than a heredoc, which does not exist in PowerShell or cmd.

   **`--no-ff` is load-bearing.** It puts `[BRANCH]` into `[BASE]`'s ancestry — what PHASE 2 checks
   and what lets `git branch -d` accept the delete. `--squash` produces the same files but leaves
   the branch *not* an ancestor, so deleting would need `-D`. A fast-forward mints no commit, so
   `/forge-contextmap`'s post-commit hook never fires and the graph goes stale.

4. **On conflict** (the merge exits non-zero and reports conflicted paths):

   ```bash
   git merge --abort
   ```

   Confirm the abort restored the base — both of these, and both must agree:

   ```bash
   git status --porcelain
   ```

   ```bash
   git rev-parse HEAD
   ```

   Empty status and `HEAD` equal to `[BASE_SHA]` means nothing was left half-merged. Report the
   conflicted files and STOP:

   ```
   ❌ Merge conflict — [BASE] moved since [BRANCH] was cut. Nothing was merged; [BASE] is untouched.

   Conflicted:
     [path]
     ...

   Fix it on the branch, where your tests still run:
     git checkout [BRANCH]
     git merge [BASE]
     (resolve, commit, re-verify)
   Then re-run /forge-merge.
   ```

   Resolving it here means picking a winner between two intents with no way to test the result.

---

## PHASE 2: Verify It Fully Merged

One check. `[BRANCH]`'s tip must be reachable from `[BASE]`:

```bash
git merge-base --is-ancestor [BRANCH] [BASE]
```

Exit 0 passes. `--is-ancestor` asks *is the first ref an ancestor of the second*, so `[BRANCH]` comes
before `[BASE]` — reversed, it answers a different question and passes for the wrong reason.

**On a non-zero exit, do not delete anything.** Run this once as a diagnostic to name what is
missing, then leave `[BRANCH]` alive and stop:

```bash
git log --oneline [BASE]..[BRANCH]
```

One check, because there is one question. That log coming back empty, `git branch --merged` listing
the branch, and `--is-ancestor` exiting 0 are the same reachability test in three spellings — which
is why the log belongs on the failure path only. `git branch -d` consults that same reachability, so
PHASE 3 is git's own backstop on the deletion, not an independent second opinion.

---

## PHASE 3: Delete the Merged Branch

1. **Get onto `[BASE]`** — git refuses to delete the checked-out branch. PHASE 1 already left you
   here; the already-merged path from 0e skipped PHASE 1 and its checkout, so run this then (a no-op
   otherwise) rather than watching `-d` fail with `cannot delete branch '…' used by worktree at '…'`:

   ```bash
   git checkout [BASE]
   ```

2. **Delete it:**

   ```bash
   git branch -d [BRANCH]
   ```

   `-d` refuses if git thinks the branch isn't merged. That contradicts PHASE 2 and means something
   about the refs is not what you think — quote git's exact message, leave the branch, stop. Never
   reach for `-D`.

3. **Sweep the orchestrate sub-branches.** A parallel `/forge-orchestrate` run leaves
   `[BRANCH]-st<i>` branches behind when its Phase 6 cleanup didn't finish. List them first — the
   glob cannot match `[BRANCH]` itself:

   ```bash
   git branch --list "[BRANCH]-st*"
   ```

   Strip the leading marker off each name: `git branch` prefixes the current branch with `*` and a
   branch checked out in **another worktree** with `+` — the case that shows up here. Feeding
   `+name` to `git branch -d` gets you `error: branch '+name' not found`, a confusing miss rather
   than the real refusal.

   Each was merged into `[BRANCH]`, now in `[BASE]`, so plain `-d` accepts them. One at a time:

   ```bash
   git branch -d [BRANCH]-st1
   ```

   A refusal comes in two flavours and they mean opposite things — read which one git printed:

   - **`cannot delete branch '…' used by worktree at '…'`** — housekeeping. Report the path and carry
     on with the sweep; one stubborn sub-branch is not a reason to abandon the cleanup.
   - **`the branch '…' is not fully merged`** — a **finding**, and the reason this sweep exists.
     `/forge-orchestrate`'s 5e merge-back never completed for that subtask, so committed work on the
     sub-branch never reached `[BRANCH]` and therefore never reached `[BASE]`. Print what is
     stranded and leave the branch alone — it is the only copy:

     ```bash
     git log --oneline [BASE]..[BRANCH]-st1
     ```

   Git's own hint on that second message suggests `-D`. Do not take it.

---

## PHASE 4: Report

Counts come from 0e, not a fresh query — after the merge `git log [BASE]..[BRANCH]` is empty by
design, so re-querying reports zero commits for work that landed.

```
✅ /forge-merge complete — [BRANCH] → [BASE]

Merged:   [N] commits, [M] files changed
Verified: [BRANCH] is an ancestor of [BASE]   pass
Deleted:  [BRANCH]
          [BRANCH]-st1   (orchestrate worktree sub-branch)

Local only — nothing was pushed.
  Push when ready:  git push origin [BASE]
  Undo the merge:   git reset --hard [BASE_SHA]   (while on [BASE])
```

Drop the sub-branch line when there were none; list any branch you preserved with git's reason. If
0e found the branch already merged, say so in place of the `Merged:` line rather than printing zero
commits as if work landed, and drop the undo line — there was no merge to undo.

---

**Maintainer note.** The `base-branch` shared block is duplicated verbatim in `/forge-orchestrate` on
purpose, because a skill can be installed alone. Edit one copy, update the other; the README's
*Shared blocks* table lists every location.
