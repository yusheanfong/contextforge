---
name: forge-contextmap
description: Graph-powered project context scaffold (/forge-contextmap). Use when scaffolding project docs (CLAUDE.md + doc/), starting a new project from scratch, onboarding or analyzing an existing codebase and generating docs from it, syncing graph-generated doc sections after code changes, upgrading old ContextForge (v1) docs, working with ContextForge docs, or when graphify-out/ exists in the repo. Triggers include "/forge-contextmap", "forge-contextmap sync", "scaffold project docs", "set up project context", "analyze this codebase and document it".
argument-hint: "[sync | --new]"
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# /forge-contextmap — Graph-Powered Context Scaffold

Combines Graphify (code → knowledge graph) with ContextForge-style AI workflow docs.
Reads $ARGUMENTS to detect subcommand: `sync` or `--new`. Otherwise auto-detects mode.

Doc format version: **v2** — PRD, app flow, design brief, backend schema, engineering-plan task
list. Old-format (v1) projects are auto-detected and upgraded via **MIGRATION MODE** with all user
content preserved.

## CONTRACT (non-negotiable guarantees)

- Sync writes ONLY inside `<!-- graphify:auto start/end -->` fences; content outside fences is
  never touched.
- `/forge-contextmap` never rewrites `doc/task-list.md` content — it's user-authored.
  (`/forge-orchestrate`'s status ticks are the sole exception.)
- `/forge-contextmap` never rewrites or deletes `doc/diagnosis-*.md` — written by
  `/forge-diagnose`, fully user-owned, no fences. MIGRATION MODE reads every file in `doc/`; it must
  treat these as untouchable, not as v1 docs needing upgrade.
- User-authored content is preserved verbatim — moved, never rewritten.
- SYNC MODE also prints a **bloat signal** (orphan / duplicate-label / god-node / dead-file counts)
  from the graph it already loaded. Counts only — it reads no source and writes nothing extra;
  `/forge-audit` is what confirms those pointers against real code.
- `doc/prd.md` and `doc/task-list.md` have NO fences — 100% user-owned.
- Fence syntax + v2 fence key list: [`references/fence-format.md`](references/fence-format.md).

---

## STEP 1: Detect Mode

Run these checks in order and jump to the matching section:

1. If `$ARGUMENTS` contains `sync` → go to **SYNC MODE** — read [`references/sync.md`](references/sync.md) and follow it exactly
2. If `$ARGUMENTS` contains `--new` → go to **NEW PROJECT MODE** — read [`references/new-project.md`](references/new-project.md) and follow it exactly
3. If `doc/` exists AND contains at least one ContextForge doc (`architecture.md`,
   `task-list.md`, `solution-structure.md`, or `ui-guideline.md`) AND `CLAUDE.md` does NOT
   contain the marker `<!-- contextforge:format v2 -->` → go to **MIGRATION MODE**
   (old-format project — upgrade it) — read [`references/migration.md`](references/migration.md) and follow it exactly
4. Check if `graphify-out/graph.json` exists in the current directory
   - If yes → go to **SYNC MODE** — read [`references/sync.md`](references/sync.md) and follow it exactly
5. Count project files by extension (case-insensitive; this is a shell-level file count):
   - Source: `.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.dart`, `.go`, `.cs`, `.java`, `.rb`, `.rs`, `.swift`, `.kt`, `.cpp`, `.c`, `.h`. This is not Graphify's complete code dispatch set — it omits `.php`, `.scala`, `.sh`, `.vue`, `.svelte`, `.m`, so a repo using only those suffixes counts as zero source here.
   - Documentation: `.md`, `.mdx`, `.qmd`, `.skill` (the same suffixes as `DOC_SUFFIXES` in
     [`references/sync.md`](references/sync.md) Step S2.5)
   - If the source count is non-zero → go to **EXISTING PROJECT MODE** — read [`references/existing-project.md`](references/existing-project.md) and follow it exactly
   - If the source count is zero and the documentation count is at least two → go to **EXISTING PROJECT MODE** — read [`references/existing-project.md`](references/existing-project.md) and follow it exactly. The two-file floor is a judgment call, not a measured threshold: one README commonly describes intent for a project that does not exist yet; two documents are the minimum corpus this mode treats as material to analyze.
6. Otherwise → go to **NEW PROJECT MODE** — read [`references/new-project.md`](references/new-project.md) and follow it exactly
