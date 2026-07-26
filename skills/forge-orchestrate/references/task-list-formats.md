# `doc/task-list.md` — format detection and status ticks

Read this **only** when `doc/task-list.md` exists. It covers two points: picking the next task
(Phase 0d) and ticking what shipped (Phase 6). Projects without a task list skip this file entirely.

---

## Phase 0d — pick the next incomplete task

Read the file and offer the next incomplete task. Wait for confirmation. Format detection:

- **v2 engineering-plan format** (`### Task N.M` blocks): the next incomplete task is the first
  `### Task N.M` in document order whose `- [ ] done` is unchecked AND whose `Depends on` tasks are
  all done (`Depends on: none` is always eligible). Carry its `Acceptance criteria` lines and
  `Builds: Fn` reference forward — Phase 1 uses them.
- **v1 flat format** (plain `- [ ]` lines, unmigrated project): the first incomplete `[ ]` line.

If the task came from a v2 task block, store `[TASK_CRITERIA]` (its acceptance-criteria lines) and
`[TASK_FEATURE]` (its `Builds:` PRD feature ref) alongside `[TASK]`.

---

## Phase 6 — tick what the gates actually verified

Update the file regardless of whether `[TASK]` came from the list:

- **v2 engineering-plan format** (`### Task N.M` blocks): find the task block that best matches
  `[TASK]` (semantic match on the title/goal, not an exact string match). If one matches:
  - tick its `- [ ] done` → `- [x] done`
  - tick ONLY the acceptance-criteria boxes the gates actually verified: `test added` ← tests gate
    passed, `no errors` ← lint gate passed, `works as expected` ← the success-criterion check
    passed, `meets PRD requirement` ← spec-compliance (5b.1) passed. Leave anything unverified
    unticked — never fake a check.
- **v1 flat format**: find the incomplete `[ ]` line that best matches `[TASK]`; tick it
  `[ ]` → `[x]`.
- If nothing matches (an ad-hoc request not on the list), append under a
  `## Completed (orchestrated)` section — create that section once if it isn't there — so the master
  list reflects what actually shipped. In a v2 file append a minimal task block
  (`### Task — [TASK]` + `- [x] done`); in a v1 file append `- [x] [TASK]`.
- Only tick or append; never rewrite, reorder, or reword existing user-authored lines.

`/forge-orchestrate`'s status ticks are the **sole** exception to `/forge-contextmap` never
rewriting `doc/task-list.md` — the rest of the file stays user-authored.
