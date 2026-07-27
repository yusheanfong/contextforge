---
name: forge-merge
description: Land a verified feature branch and clean up (/forge-merge). Use after /forge-orchestrate when you have reviewed the branch and want it merged back into main or master, confirmed fully merged, and the branch deleted. Triggers include "/forge-merge", "merge this branch back", "land the feature branch", "merge and delete the branch", "finish this feature", "clean up the merged branch".
argument-hint: "[branch]"
allowed-tools: Read, Bash, AskUserQuestion
---

# /forge-merge — Land a Verified Branch, Prove It Landed, Clean Up

`/forge-orchestrate` leaves you checked out on a feature branch with N commits and the closing line
*"run /forge-merge when you're happy to land it and clean up."* This is that command.

Three jobs, in order: **merge** the branch into the repo's base branch, **prove** it fully landed
with checks that each map to a real command's exit status, then **delete** the branch — so
`git branch` stays short enough to read.

Reads `$ARGUMENTS` as the branch to land. With no arguments it lands the branch you are on, which is
the normal case: you just finished reviewing it.

**This command commits** — exactly one merge commit on the base branch. It never pushes and never
touches `origin`.

## CONTRACT (non-negotiable guarantees)

- **Local only.** No `git push`, no `git fetch`, no remote writes of any kind. Everything happens in
  your clone and works offline. The final report prints the push command; you run it when you want
  the merge to leave your machine.
- **Never force-deletes.** `git branch -d` only — never `-D`. Git's own refusal to delete an
  unmerged branch is the last line of defence, and forcing past it is exactly how work disappears.
- **Never resolves a conflict for you.** A conflict means two real intents collided over the same
  lines, and only you know which one wins. Conflict → `git merge --abort`, confirm the base branch
  is back where it started, report, stop.
- **Never re-runs the gates.** `/forge-orchestrate` gated every commit and you verified the branch
  before invoking this — re-running lint and tests here would be theatre, not verification.
- **Never touches your working-tree changes.** A dirty tree gets a question, never a silent stash
  and never a silent commit.
- **Never prints a check it didn't run.** Every line in the report's *Verified* block reflects an
  actual command's exit status. A check that didn't run says so.
- Unlike `/forge-orchestrate` and `/forge-audit`, this command does **not** require
  `graphify-out/graph.json` — it reasons about git refs, not about code. It runs in any git repo.

**A non-zero exit is often the answer, not a failure.** `git rev-parse --verify --quiet`,
`git merge-base --is-ancestor`, and `git branch -d` all use exit status to *say something*. Read the
code, don't retry the command and don't treat it as an error to work around.

---

## PHASE 0: Resolve What Is Being Merged

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

If nothing resolves — a repo on `trunk`, `develop`, or anything else, with no `origin/HEAD` to read
it from — do not improvise a base. List the candidates and ask with AskUserQuestion, excluding
`[BRANCH]` itself:

```bash
git branch
```

Stopping outright would be worse than asking: the branch is finished and the user knows the answer
in a second. Only stop if the repo has no other branch to merge into.

If `[BRANCH]` and `[BASE]` are the same, stop: you are already on the base branch and there is
nothing to land.

### 0c. Dirty-tree check

```bash
git status --porcelain
```

If the tree is dirty, ASK before proceeding — stash (`git stash push -u`) or abort. A merge with
uncommitted work in the tree mixes your pending edits into the conflict surface, and the recovery is
worse than the wait. Do not silently discard, stash, or commit the user's pending work.

### 0d. Leftover-worktree check

Clear stale metadata first, so a worktree that was already deleted from disk doesn't stop the run:

```bash
git worktree prune
```

```bash
git worktree list
```

Git will not delete a branch that is checked out in a worktree, and this check is cheap here versus
discovering it in PHASE 3, *after* the merge already happened. `/forge-orchestrate`'s parallel mode
is the usual source of extra worktrees, but a hand-made one counts the same.

If any worktree other than the main one has `[BRANCH]` checked out, report the exact paths and stop.
Removing someone's worktree is not this command's call — `git worktree remove <path>` is one command
and the user knows whether they still need it.

### 0e. Show what is about to land

```bash
git log --oneline [BASE]..[BRANCH]
```

```bash
git diff --stat [BASE]...[BRANCH]
```

Three dots on the diff: changes made on `[BRANCH]` since it diverged, not changes that happened on
`[BASE]` in the meantime. Print both, then continue — invoking `/forge-merge` after reviewing the
branch *is* the confirmation, so don't ask again. PHASE 4 prints the exact undo command instead,
which is worth more than a prompt.

**If the log is empty**, `[BRANCH]` is already merged or has no commits of its own. Skip PHASE 1
entirely and go to PHASE 2 — the verification and cleanup are still exactly what you want.

---

## PHASE 1: Merge

1. **Capture the undo anchor first** — this has to happen before the merge or it is worthless:

   ```bash
   git rev-parse [BASE]
   ```

   Store as `[BASE_SHA]`.

2. **Check out the base:**

   ```bash
   git checkout [BASE]
   ```

3. **Merge with a real merge commit.** Write a one-line summary of the feature from the PHASE 0e
   commit log — what landed, not "merge branch X":

   ```bash
   git merge --no-ff [BRANCH] -m "merge [BRANCH]: [one-line summary]" -m "Co-Authored-By: Claude <noreply@anthropic.com>"
   ```

   Two `-m` flags rather than a heredoc, which does not exist in PowerShell or cmd. `--no-ff` is
   load-bearing, not cosmetic — see NOTES.

4. **On conflict** (the merge exits non-zero and reports conflicted paths):

   ```bash
   git merge --abort
   ```

   Confirm the abort actually restored the base — both of these, and both must agree:

   ```bash
   git status --porcelain
   ```

   ```bash
   git rev-parse HEAD
   ```

   Empty status and `HEAD` equal to `[BASE_SHA]` means nothing was left half-merged. Then report the
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

   Resolving it here would mean picking a winner between two intents with no way to test the result.
   Do not do it.

---

## PHASE 2: Verify It Fully Merged

The double-check, and the reason this command exists rather than a bare `git merge`. Run all four
and record each one's real result:

| Check | Command | Passes when |
|---|---|---|
| Ancestry | `git merge-base --is-ancestor [BRANCH] [BASE]` | exit 0 |
| Nothing left behind | `git log --oneline [BASE]..[BRANCH]` | no output |
| Branch content is in the base | `git diff --stat [BASE]...[BRANCH]` | no output |
| Git agrees | `git branch --merged [BASE]` | lists `[BRANCH]` |

```bash
git merge-base --is-ancestor [BRANCH] [BASE]
```

```bash
git log --oneline [BASE]..[BRANCH]
```

```bash
git diff --stat [BASE]...[BRANCH]
```

```bash
git branch --merged [BASE]
```

Two argument details that are easy to get wrong, and both fail *silently* in the direction that
matters:

- `--is-ancestor` asks *is the first ref an ancestor of the second*, so `[BRANCH]` comes before
  `[BASE]`. Reversed, it answers a different question and passes for the wrong reason.
- **Three dots on the diff, never two.** `[BASE]...[BRANCH]` compares the merge base against the
  branch: "are the branch's own changes all in the base?" Two dots compares the two tips, which
  reports every commit the base gained *after* the branch was cut as a difference — so a perfectly
  merged branch fails the check the moment anyone else lands anything. That is the common case, not
  an edge case.

These are corroborating, not logically independent — when the merge is a normal `--no-ff` all four
answer the same underlying question. That is the point: they are nearly free, they each print a
*different view* of a failure (an exit code, the missing commits by name, the missing content, git's
own verdict), and the fourth is exactly what `git branch -d` will consult in PHASE 3. If they ever
disagree, something about the refs is not what you think it is — stop and look.

**If any check fails, do not delete anything.** Report which check failed and what it printed, leave
`[BRANCH]` alive, and stop. A branch left behind costs nothing; a branch deleted on a bad assumption
costs the work.

---

## PHASE 3: Delete the Merged Branch

1. **Confirm you are on `[BASE]`** — git refuses to delete the branch that is currently checked out:

   ```bash
   git rev-parse --abbrev-ref HEAD
   ```

2. **Delete it:**

   ```bash
   git branch -d [BRANCH]
   ```

   `-d` refuses if git thinks the branch isn't merged. That refusal is a real signal that PHASE 2
   missed something — quote git's exact message, leave the branch, and stop. Never reach for `-D`.

3. **Sweep the orchestrate sub-branches.** A parallel `/forge-orchestrate` run leaves
   `[BRANCH]-st<i>` branches behind when its Phase 6 cleanup didn't finish. List them first — the
   glob cannot match `[BRANCH]` itself, so there is no risk of catching the wrong ref:

   ```bash
   git branch --list "[BRANCH]-st*"
   ```

   Read the names off that output without their leading marker: `git branch` prefixes the current
   branch with `*` and a branch checked out in **another worktree** with `+`. The `+` case is
   exactly what shows up here, and feeding `+name` to `git branch -d` gets you
   `error: branch '+name' not found` — a confusing miss rather than the real refusal.

   Each was merged into `[BRANCH]`, which is now in `[BASE]`, so plain `-d` accepts them. Delete one
   at a time:

   ```bash
   git branch -d [BRANCH]-st1
   ```

   A refusal here — most likely `cannot delete branch … used by worktree at …` — gets reported, and
   you carry on with the rest. One stubborn sub-branch is not a reason to abandon the cleanup, and
   it is never a reason to reach for `-D`.

---

## PHASE 4: Report

```
✅ /forge-merge complete — [BRANCH] → [BASE]

Merged:   [N] commits, [M] files changed
  [sha] — [subject]
  ...

Verified: [BRANCH] is an ancestor of [BASE]        pass
          no commits left on [BRANCH]              pass
          branch content all present in [BASE]     pass
          git branch --merged lists it             pass

Deleted:  [BRANCH]
          [BRANCH]-st1, [BRANCH]-st2   (orchestrate worktree sub-branches)

Local only — nothing was pushed.
  Push when ready:  git push origin [BASE]
  Undo the merge:   git reset --hard [BASE_SHA]   (while on [BASE])
```

Drop the sub-branch line when there were none. If PHASE 0e found the branch already merged, say so
in place of the `Merged:` block rather than printing zero commits as if work landed.

---

## NOTES

- **Local only, and no `fetch` either.** `/forge-orchestrate` never pushes, and neither does this —
  the whole pipeline stays in your clone until you decide otherwise. Skipping `fetch` also means the
  command works on a plane, and that nothing about the merge depends on network state you didn't ask
  for.
- **`--no-ff` is what makes PHASE 2 cheap.** A real merge commit puts `[BRANCH]` in `[BASE]`'s
  ancestry, so git itself can confirm the merge and `git branch -d` becomes a genuine second opinion
  rather than a formality. A `--squash` merge produces the same files but leaves the branch *not* an
  ancestor: `git branch --merged` wouldn't list it, `-d` would refuse, and deleting would require
  `-D` — throwing away the exact safety check this command is built around. Same reason a
  fast-forward is not enough: no commit means `/forge-contextmap`'s post-commit hook never fires, so
  the graph wouldn't refresh on the merged code.
- **Never `-D`.** If a check fails, the answer is to investigate, never to force. Every path in this
  skill that could reach a force-delete stops and reports instead.
- **No gate re-run.** The gates ran at commit time in `/forge-orchestrate` and you verified the
  branch yourself — that is the premise of invoking this. Re-running them would add minutes and
  verify nothing new.
- **The base branch is detected, not assumed.** `main` on some repos, `master` on others, and a
  wrong guess merges the work somewhere you didn't intend. The `base-branch` shared block is the
  same text `/forge-orchestrate` uses for its merge-conflict pre-check — edit one copy, update both.
- **Portability contract.** Every command here runs identically on macOS, Linux, and Windows: no
  heredocs, no `cp`/`mv`/`rm`, no `mkdir -p`/`chmod`, no `2>/dev/null`, no `||` chaining. Every bash
  block is a single git invocation, which is why silence-on-missing-ref comes from `--quiet` rather
  than a redirect.
- **Shared blocks** — the `<!-- forge:shared-block ... -->` markers wrap text duplicated in other
  forge skills on purpose (each skill installs standalone). Edit one copy, update the rest; the
  README's *Shared blocks* table lists every location.
