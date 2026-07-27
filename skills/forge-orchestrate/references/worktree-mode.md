# Worktree mode — parallel dispatch, commit, merge-back, cleanup

Read this **only** when a batch has **2+ independent (parallel) subtasks** AND `[NO_COMMIT]` is
false. Lone subtasks, dependent sequences, and every `--no-commit` run stay single-tree — that path
is fully described in `SKILL.md` and does not need this file.

Each parallel subtask gets its own isolated checkout + sub-branch, so overlapping edits can never
collide in a shared index. This is the preferred path for real parallelism.

This file covers four points in the pipeline: dispatch (Phase 4), commit (5d), merge-back (5e), and
cleanup (Phase 6).

---

## Phase 4 — dispatch

For each parallel subtask `i` in the batch, the MAIN session creates an isolated worktree on a
sub-branch off `[BRANCH]`:

```bash
git worktree add ../<repo>-st<i> -b [BRANCH]-st<i> [BRANCH]
```

The sub-branch is `[BRANCH]-st<i>`, with a **hyphen** — not `[BRANCH]/st<i>`. Git refs are
filesystem paths, so `refs/heads/[BRANCH]` cannot be a file and a directory at once: with `[BRANCH]`
already created in Phase 2, the slash form fails outright with `cannot lock ref … exists; cannot
create …` and the whole parallel path dies at dispatch. The hyphen makes each sub-branch a sibling
ref instead of a child, which is what lets this run at all. The worktree *directory* keeps its
`-st<i>` suffix for the unrelated reason that it reads well next to the repo.

If `git worktree add` fails with `'…' already exists`, that directory is a leftover from an
interrupted run. Report the path and let the user clear it — do not delete a directory you did not
create.

Then dispatch worker `i` with its payload (Phase 3) **plus** the absolute worktree path, instructing
it: "Make ALL edits and run ALL commands under `<worktree abs path>` — that is your isolated
checkout. Do not touch any other directory." Workers in the same batch run fully in parallel with
zero shared mutable state.

Both halves of Phase 5 need that same path:

- **The gate runner (5a)** runs with that worktree as cwd.
- **The 5b reviewer subagent** is spawned from the MAIN session, so it starts in the main session's
  cwd — *not* the worktree. Its payload MUST carry the absolute worktree path and use
  `git -C <worktree abs path> diff -- [files]`. A bare `git diff` returns nothing there and the
  reviewer reports `VERDICT: PASS` on an empty diff.

After a worker passes its gates, its subtask is committed **on its sub-branch inside its worktree**
(5d below), then merged back (5e).

---

## 5d — commit inside the worktree

Commit inside the subtask's worktree, on its sub-branch. Each worktree is its own isolated checkout,
so these commits CAN run in parallel — there is no shared index. Stage only the worker's reported
files (never `git add -A`):

```bash
git -C ../<repo>-st<i> add [exact files this worker reported]
git -C ../<repo>-st<i> commit -m "[concise subtask summary]"
```

End every commit message with:

```
Co-Authored-By: Claude <noreply@anthropic.com>
```

Record the sub-branch + commit SHA in the subtask state. Merge-back happens in 5e.

---

## 5e — merge sub-branches back

Once every subtask in a parallel batch has committed on its sub-branch, the MAIN session merges each
sub-branch back into `[BRANCH]`, **one at a time** (serialized — merges share `[BRANCH]`'s index):

```bash
git checkout [BRANCH]
git merge --no-ff [BRANCH]-st<i> -m "merge [BRANCH]-st<i>: [subtask summary]"
```

- **Conflict on merge** = two subtasks really did edit the same lines. This is now a *visible, real*
  conflict (the whole point of worktrees vs. the silent single-tree collision). Resolve it, or if it
  signals the decomposition overlapped badly, escalate to the user.
- After all sub-branches are merged, proceed to Phase 6.

---

## Phase 6 — cleanup

Remove every worktree and delete its merged sub-branch:

```bash
git worktree remove ../<repo>-st<i>
git branch -d [BRANCH]-st<i>
```

Order matters: remove the worktree **before** deleting its branch. Git refuses to delete a branch
that is checked out anywhere — `error: cannot delete branch '…' used by worktree at '…'`.

Run `git worktree prune` to clear any stale entries. If a worktree won't remove because of
uncommitted leftovers, report it rather than force-removing.
