## FENCE FORMAT REFERENCE

```
<!-- graphify:auto start:QUALIFIED_KEY:SECTION -->
Content here is managed by /forge-contextmap sync.
DO NOT manually edit — changes will be overwritten.
<!-- graphify:auto end:QUALIFIED_KEY:SECTION -->
```

**Key format**: `project:section_name` for project-level summaries.

v2 fence keys: `project:claude-summary`, `project:architecture`, `project:domain-model`,
`project:api-contract`, `project:solution-structure`, `project:coding-standard`,
`project:security`, `project:app-flow`, `project:design-brief`, `project:backend-schema`.

Example:
- `<!-- graphify:auto start:project:domain-model -->`

**Rules**:
- Content inside fences: overwritten on every sync
- Content outside fences: never touched
- `doc/prd.md`, `doc/task-list.md` and `doc/diagnosis-*.md` have NO fences — 100% user-owned
  (orchestrate's status ticks are the sole exception for task-list.md)

**Curated prose belongs OUTSIDE the fence.**

A fence carries **graph-derived facts only** — god node labels, community names and ids, entry
points, detected components, counts. Anything a hand wrote and an AST cannot reproduce goes *below*
the `end` marker.

This is not a style preference. Sync regenerates fence content wholesale from `graph.json`; there is
no merge step and cannot be one, because "regenerate from the graph" and "preserve what isn't in the
graph" are contradictory instructions. Prose left inside a fence is destroyed on the next sync.

The conventional shape is a `### Notes` heading right after the end marker:

```
## Key Architecture (from Graphify)
<!-- graphify:auto start:project:claude-summary -->
**God nodes**: GaugeReader, OtaService, CaptureController
**Communities**: 7 — Capture, Inference, OTA, Storage, UI, Bridge, Utils
**Entry points**: lib/main.dart
<!-- graphify:auto end:project:claude-summary -->

### Notes
- results/2-up cards + UserValue + save
- OTA zip unpacks into app-support, not cache
```

Sync's Step S4 detects non-derivable lines inside a fence, warns, and rescues them **once** so an
existing project does not lose them on the sync that introduces this rule. That rescue does not
repeat — move the lines when you see the warning.
