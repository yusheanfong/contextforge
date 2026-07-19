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
- `doc/prd.md` and `doc/task-list.md` have NO fences — 100% user-owned (orchestrate's
  status ticks are the sole exception for task-list.md)
