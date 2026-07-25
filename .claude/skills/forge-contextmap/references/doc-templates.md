# Doc File Templates (v2)

The 13 doc file templates used by NEW PROJECT MODE Step N2, and referenced by EXISTING
PROJECT and MIGRATION modes. Conditional rules: `design-brief.md` only if `[HAS_UI]`,
`backend-schema.md` only if `[HAS_BACKEND]`.

**File 1: `CLAUDE.md`**

<!-- forge:shared-block source-doc-map — Rule 1's task→doc routing below mirrors
     forge-orchestrate/SKILL.md 3b. Keep both in sync. The marker lives outside the template fence
     on purpose: anything inside the fence is copied verbatim into the user's CLAUDE.md. -->


```
# Project Context
<!-- contextforge:format v2 -->

## Goal
[GOAL]

## Tech Stack
[TECH_STACK]

## Doc Navigation
All project docs live in /doc:
- doc/prd.md              — Product requirements: idea overview, core features (F1..Fn), out of scope
- doc/app-flow.md         — Entry point, screen/step map, user journeys, data flow
- doc/design-brief.md     — Color tokens, typography, components, screen style   [omit this line if no UI]
- doc/backend-schema.md   — Storage, entities/tables, relations, indexes         [omit this line if no backend]
- doc/architecture.md     — Tech stack, layers, design patterns
- doc/domain-model.md     — Entities, enums, business rules
- doc/api-contract.md     — API endpoints or service interfaces
- doc/solution-structure.md — Project folder layout
- doc/coding-standard.md  — Language and framework conventions
- doc/security.md         — Auth, roles, data protection rules
- doc/task-list.md        — Engineering plan / master task list (YOUR ONLY TODO SOURCE)
- doc/diagnosis-*.md      — Completed root-cause diagnoses from /forge-diagnose  [if any exist]
- doc/changelog.txt       — Change log
- doc/progress.txt        — Current status

## Graph Sync
Sections marked <!-- graphify:auto start:... --> are auto-populated by /forge-contextmap sync.
Edit ONLY outside these markers — content inside is overwritten on sync.
When Graphify runs for the first time (after code exists), run /forge-contextmap sync to populate.

## Rules
1. Before writing code, ALWAYS read doc/architecture.md, doc/solution-structure.md, doc/coding-standard.md, and doc/prd.md (universal constraints + scope guard). Then review doc/task-list.md and graphify-out/GRAPH_REPORT.md to decide which domain docs the current task touches, and read ONLY those:
   - UI/screen/widget/component task → doc/design-brief.md + doc/app-flow.md
   - data/entity/model task → doc/domain-model.md + doc/backend-schema.md (if present)
   - API/service/endpoint task → doc/api-contract.md + doc/backend-schema.md (if present)
   - auth/permission task → doc/security.md
2. Implement ONLY the next task in doc/task-list.md whose "Depends on" tasks are all done.
3. Update doc/progress.txt after every completed task.
4. Update doc/changelog.txt after every change (format: Date | Change | Description).
5. Follow doc/solution-structure.md exactly — no structural changes.
6. UI code: use ONLY doc/design-brief.md tokens and components — no ad-hoc hex values, font sizes, spacing values, or one-off components.
7. Never invent schema fields, entities, or endpoints not defined in doc/backend-schema.md, doc/domain-model.md, or doc/api-contract.md.
8. Fixing a bug and a doc/diagnosis-*.md covers it → read that file FIRST and follow it. The investigation is already done; do not re-explore the codebase for the same root cause.

## Coding Rules

### Think Before Coding
State assumptions before acting. If uncertain, ask — don't guess.
- Multiple interpretations → present them, don't silently pick one
- Simpler approach exists → say so and push back
- Confused by a requirement → name what's confusing, stop, ask

### Simplicity First
Write the minimum code that solves the stated problem. Nothing speculative.
- No features beyond what was asked
- No abstractions for single-use code
- No "future flexibility" that wasn't requested
- No error handling for impossible scenarios
- If it could be half the length, rewrite it

### Surgical Changes
Touch only what the task requires. Match existing style.
- Don't improve adjacent code, formatting, or comments
- Don't refactor things that aren't broken
- If you notice unrelated dead code, mention it — don't delete it
- Remove imports/variables/functions YOUR changes made unused; leave pre-existing dead code alone

Every changed line must trace directly to the user's request.

### Goal-Driven Execution
Define what "done" looks like before writing code.

Turn tasks into verifiable goals:
- "Add validation" → tests for invalid inputs pass
- "Fix the bug" → test reproduces it, then it passes
- "Refactor X" → tests pass before and after; nothing changed externally

For multi-step tasks, state a brief plan with a verify step per step:
1. [Step] → verify: [check]
2. [Step] → verify: [check]
```

Note: rules 1/6/7 reference `design-brief.md` / `backend-schema.md` — drop those references from the rules text too when the corresponding doc isn't created.

**File 2: `doc/prd.md`** — fully user-owned, no graph fence:

```
# Product Requirements

## Idea Overview
[GOAL]

## Core Features
<!-- Priority: P0 = must-have for launch, P1 = important, P2 = nice-to-have -->
[FEATURES — the approved F1..Fn list]

## Out of Scope
[Derive 2-4 explicit non-goals from the conversation, or write "[FILL IN: what you're explicitly NOT building]"]
```

**File 3: `doc/app-flow.md`** — draft it FROM `[GOAL]` + `[FEATURES]` (real content, not placeholders):

```
# App Flow

<!-- graphify:auto start:project:app-flow -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:app-flow -->

## Entry Point
[e.g. app launch → auth check → home screen; or for an API: request → middleware → router]

## Screen / Step Map
1. [Screen/Step] — [purpose]
2. [Screen/Step] — [purpose]
...

## User Journeys
[One journey per P0 feature:]
### F1 — [feature name]
[step] → [step] → [outcome]

## Data Flow
[Where data originates, how it moves through the app, where it's persisted]
```

**File 4: `doc/design-brief.md`** — ONLY if `[HAS_UI]`. Fill from the approved `[DESIGN_SYSTEM]`:

```
# Design Brief

Single source of truth for look & feel. Consistency over novelty — every screen uses these
tokens and components. No randomness.

<!-- graphify:auto start:project:design-brief -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:design-brief -->

## Color Tokens
| Token   | Hex     | Usage |
|---------|---------|-------|
| Primary | [#hex]  | buttons, links, active states |
| Surface | [#hex]  | backgrounds, cards |
| Text    | [#hex]  | primary text |
| Muted   | [#hex]  | secondary text, placeholders |
| Border  | [#hex]  | dividers, input borders |
| Success | [#hex]  | success states |
| Error   | [#hex]  | errors, destructive actions |

## Typography
- Family: [font]
- Scale: [e.g. 32 / 24 / 18 / 16 / 14] (display / h1 / h2 / body / caption)
- Weights: [e.g. 400 regular, 500 medium, 700 bold]

## Spacing & Radius
- Base unit: [e.g. 4px] — allowed spacing: [e.g. 4 / 8 / 12 / 16 / 24 / 32]
- Radius: [e.g. 8px cards & inputs, 999px pills]

## Reusable Components
[From DESIGN_SYSTEM, e.g.:]
- Button: primary / secondary / ghost
- Card
- Input (text, with label + error state)
- Modal
- Toast (success / error)

## Screen Style Guidance
- List screens: [e.g. cards in a single column, 16px gaps, pull-to-refresh]
- Detail screens: [e.g. hero section + sectioned content]
- Forms: [e.g. single column, labels above inputs, primary action pinned bottom]
- Empty states: [e.g. icon + one line + primary action]

## UX Rules
- Confirm before destructive actions
- Toast notifications for success/error
[FILL IN: more rules as they emerge]

## Hard Rule
Use ONLY the tokens and components defined above. No ad-hoc hex values, font sizes, spacing
values, or one-off components. Need something new? Add it HERE first, then use it.
```

**File 5: `doc/backend-schema.md`** — ONLY if `[HAS_BACKEND]`:

```
# Backend Schema

<!-- graphify:auto start:project:backend-schema -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:backend-schema -->

## Storage
[e.g. Postgres 16 via Prisma / Firestore / SQLite + Drift]

## Entities / Tables
### [table/collection name]
| Field | Type | Constraints |
|-------|------|-------------|
| [FILL IN] | | |

## Relations
[e.g. User 1—N Order; Order 1—N OrderItem]

## Indexes
[FILL IN: fields queried often]

## Auth & Ownership
See doc/security.md — roles, ownership filters, and data-protection rules live there.
```

**File 6: `doc/architecture.md`**

```
# Architecture — [GOAL summary]

## Tech Stack
[TECH_STACK]

<!-- graphify:auto start:project:architecture -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:architecture -->

## Architecture Style
[FILL IN: e.g. Clean Architecture / MVC / Layered / Feature-first]

## Project Layers
[FILL IN: list your layers, e.g. Domain / Application / Infrastructure / API]

## Naming Conventions
[FILL IN: e.g. PascalCase for classes, camelCase for variables]

## Non-Negotiable Rules
[FILL IN: e.g. No direct DB access in controllers]
```

**File 7: `doc/domain-model.md`**

```
# Domain Model

<!-- graphify:auto start:project:domain-model -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:domain-model -->

## Enums
[FILL IN: list enum names and values]

## Entity Invariants
[FILL IN: e.g. Order must belong to a Customer]

## Business Rules
[FILL IN: e.g. Cannot delete record if child records exist]
```

**File 8: `doc/api-contract.md`**

```
# API Contract

<!-- graphify:auto start:project:api-contract -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:api-contract -->

## Endpoints
[FILL IN: list endpoints grouped by resource]

## Notes
[FILL IN: e.g. JWT required on all endpoints except /auth]
```

**File 9: `doc/solution-structure.md`**

```
# Solution Structure

## Tech Stack
[TECH_STACK]

<!-- graphify:auto start:project:solution-structure -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:solution-structure -->

## Root Layout
[FILL IN: describe your folder structure]

## Layer Dependency Rules
[FILL IN: which layers depend on which]

## AI Instruction
Follow this structure exactly. Do not invent new layers or folders.
```

**File 10: `doc/coding-standard.md`**

```
# Coding Standards

## Language: [TECH_STACK]

<!-- graphify:auto start:project:coding-standard -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:coding-standard -->

## Rules
[FILL IN: e.g. Use async/await everywhere]
[FILL IN: e.g. Validate all inputs at boundary]
[FILL IN: e.g. No magic strings — use constants]
[FILL IN: e.g. Use dependency injection for all services]
```

**File 11: `doc/security.md`**

```
# Security Model

<!-- graphify:auto start:project:security -->
_No graph data yet. Run /forge-contextmap sync after adding source code to auto-populate this section._
<!-- graphify:auto end:project:security -->

## Roles
[FILL IN: list roles, e.g. Admin / User / Viewer]

## Rules
[FILL IN: e.g. JWT required for all endpoints]
[FILL IN: e.g. All queries filtered by OwnerId]
[FILL IN: e.g. Soft delete only — no hard deletes]
```

**File 12: `doc/changelog.txt`**

```
# Changelog
# Format: Date | Change | Description

[TODAY'S DATE] | Project initialized | /forge-contextmap new project setup
```

Replace `[TODAY'S DATE]` with today's actual date (YYYY-MM-DD).

**File 13: `doc/progress.txt`**

```
# Progress

## Current Status
Project initialized. No tasks completed yet.

## Summary
Waiting to begin Phase 1.
```

## Task List Format (used by Step N3)

```markdown
# Master Task List (Engineering Plan)

## Goal
[GOAL]

## How to work this list
- Build order = task numbering; a task is eligible when all its "Depends on" tasks are done.
- Implement ONLY the next eligible task.
- A task is done when every acceptance criterion is ticked.

## Phase 1 — Foundation
### Task 1.1 — [title]
- [ ] done
- Depends on: none
- Builds: [Fn or —]
- Acceptance criteria:
  - [ ] works as expected: [specific expected behavior for THIS task]
  - [ ] no errors (lint/console clean)
  - [ ] meets PRD requirement [Fn]
  - [ ] test added

### Task 1.2 — [title]
- [ ] done
- Depends on: Task 1.1
- Builds: [Fn]
- Acceptance criteria:
  - [ ] works as expected: [...]
  - [ ] no errors (lint/console clean)
  - [ ] meets PRD requirement [Fn]
  - [ ] test added

## Phase 2 — Core Features
[...]

## Phase 3 — Polish & Launch
[...]

## Notes
Update this file as requirements evolve. Keep tasks small — one sitting each.
```
