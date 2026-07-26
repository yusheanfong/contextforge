## MIGRATION MODE

*Trigger: old-format `doc/` present (ContextForge docs exist, but CLAUDE.md lacks the
`<!-- contextforge:format v2 -->` marker)*

Upgrades a v1 project to the v2 doc format. **Guarantee: no user-authored line is ever dropped or
reworded — content is moved, never rewritten.**

### Step M1: Inventory

Read every file in `doc/` plus `CLAUDE.md`. Note which v1 docs exist, which have
`<!-- graphify:auto -->` fences, and where user content lives (outside fences). Also detect
`[HAS_UI]` / `[HAS_BACKEND]` — from the graph if `graphify-out/graph.json` exists, otherwise from
source-file extensions and doc contents.

Announce:

```
Old-format ContextForge docs detected — upgrading to v2.
New in v2: doc/prd.md, doc/app-flow.md, doc/design-brief.md (absorbs ui-guideline.md),
doc/backend-schema.md (if backend), and an engineering-plan task-list format.
All your existing content is preserved verbatim.
```

### Step M2: Create `doc/prd.md`

- `## Idea Overview` — seed from the `## Goal` section of the existing `CLAUDE.md` (or
  task-list.md's `## Goal`). If neither exists, ask the user for the goal (enhance + approve, as in
  `references/new-project.md` N1 Question 1).
- `## Core Features` — infer F1..Fn with priorities from the existing task list, docs, and graph
  (if present). Present the inferred list for confirmation/correction before writing (same exchange
  style as `references/new-project.md` N1 Question 3).
- `## Out of Scope` — `[FILL IN]` placeholder.

### Step M3: Create `doc/app-flow.md`

Use the doc/app-flow.md template from `references/existing-project.md` Step E7. If `graphify-out/graph.json` exists, populate the
`project:app-flow` fence from detected entry points/screens/routes; otherwise leave the fence with
the "no graph data yet" placeholder. Draft `## User Journeys` from the confirmed P0 features.

### Step M4: Create `doc/design-brief.md` (absorbs `ui-guideline.md`)

Only if `[HAS_UI]` OR `doc/ui-guideline.md` exists:

1. Create `doc/design-brief.md` from the standard template (`references/doc-templates.md` File 4).
2. Move **ALL** content of `doc/ui-guideline.md` (including its fence and fenced content, if any)
   verbatim into a `## UX Rules (migrated from ui-guideline.md)` section — every line preserved.
3. Get concrete tokens: if the codebase has a theme/token file, extract values from it; otherwise
   ask the vibe question (`references/new-project.md` N1 Question 4) and generate per **`references/design-brief-generation.md`**; approve.
4. Ask the user: "ui-guideline.md's content now lives in design-brief.md — OK to delete
   doc/ui-guideline.md?" **Delete only after explicit yes.** If no, leave it and add a one-line
   pointer at its top: `> Superseded by doc/design-brief.md — new content goes there.`

### Step M5: Create `doc/backend-schema.md`

Only if `[HAS_BACKEND]`: standard template (`references/doc-templates.md` File 5); populate the fence from the graph if present.

### Step M6: Upgrade `doc/task-list.md` in place

**First check whether it is already v2.** If the file already contains `### Task N.M` headings, it is
in engineering-plan format — a v1 project can reach here with a task list someone already upgraded by
hand. Leave the file **byte-untouched**, convert nothing, and record `already v2` for the M8 report.
Do not add the `## How to work this list` block to a file you are not otherwise rewriting. Then skip
to M7.

Otherwise, convert flat checkbox lines to the engineering-plan format. Conversion rules — mechanical,
no rewording:

- Keep the existing `## Goal` and `## Phase N — ...` headings as-is.
- Each `- [ ] <text>` line under Phase N becomes (numbering by order within the phase):
  ```markdown
  ### Task N.M — <text, verbatim>
  - [ ] done
  - Depends on: none   <!-- FILL IN if this task needs an earlier one -->
  - Builds: [Fn if it clearly maps to a PRD feature, else —]
  - Acceptance criteria:
    - [ ] works as expected: <text, verbatim>
    - [ ] no errors (lint/console clean)
    - [ ] meets PRD requirement [Fn or —]
    - [ ] test added
  ```
- Each `- [x] <text>` (already completed) becomes:
  ```markdown
  ### Task N.M — <text, verbatim>
  - [x] done (migrated as completed — criteria not retro-verified)
  - Depends on: none
  ```
  Do NOT fabricate ticked acceptance criteria for work you didn't verify.
- Default every `Depends on` to `none` — do not invent dependencies; the FILL-IN comment invites
  the user to add real ones.
- Keep any non-checkbox lines (notes, sections like `## Completed (orchestrated)`) exactly where
  they are.
- Add the `## How to work this list` block (from the task-list format in `references/doc-templates.md`) after `## Goal`.

Show the converted file to the user before writing it. Count lines: every v1 task line must appear
verbatim in the v2 version.

### Step M7: Upgrade `CLAUDE.md`

- Add the `<!-- contextforge:format v2 -->` marker under the title.
- Replace the `## Doc Navigation` and `## Rules` sections with the v2 versions (`references/doc-templates.md`
  File 1), keeping the conditional lines consistent with `[HAS_UI]`/`[HAS_BACKEND]`.
- Keep `## Goal`, `## Tech Stack`, `## Key Architecture`, `## Graph Sync`, `## Coding Rules`, and
  ANY sections the user added themselves — preserved verbatim, untouched.
- **`## Key Architecture` needs one structural fix.** v1 put hand-written architecture prose inside
  the `project:claude-summary` fence, where sync destroys it on the next run (see
  `references/fence-format.md`). While preserving the content verbatim, **split it**: graph-derived
  facts (god nodes, communities, entry points) stay inside the fence; every hand-written line moves
  below the `end` marker under a new `### Notes` heading. Not a rewrite — the same lines, relocated.
  Show the user the split before writing, and name it in the M8 report:
  ```
  ## Key Architecture (from Graphify)
  <!-- graphify:auto start:project:claude-summary -->
  **God nodes**: ...
  **Communities**: ...
  **Entry points**: ...
  <!-- graphify:auto end:project:claude-summary -->

  ### Notes
  [the user's original prose, verbatim]
  ```

### Step M8: Report

```
✅ Migration to v2 complete!

Created:
  doc/prd.md              (features confirmed by you)
  doc/app-flow.md
  doc/design-brief.md     [if created — absorbed ui-guideline.md]
  doc/backend-schema.md   [if created]

Upgraded in place:
  doc/task-list.md        → engineering-plan format ([N] tasks converted, [M] kept completed)
                            [or: → already v2 (no conversion needed, file untouched)]
  CLAUDE.md               → v2 navigation + rules, format marker stamped

Deleted (with your OK):
  doc/ui-guideline.md     [only if approved]

Guarantee: every task line and every user-authored doc line was preserved verbatim.
Next: review the "Depends on: none" defaults in doc/task-list.md and fill in real dependencies,
then run /forge-contextmap sync (if you have a graph) to refresh the new fences.
```
