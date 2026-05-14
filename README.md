# ContextForge

One command to scaffold a complete AI-assisted coding workflow for Claude Code — then keep it alive with a live knowledge graph that updates as your code evolves.

```
/contextmap
```

---

## What It Does

Claude Code starts every session cold. It reads `CLAUDE.md` (auto-loaded, ~600 tokens) to know what it's building. Generic placeholders mean generic decisions.

`/contextmap` solves both the scaffolding problem and the accuracy problem:

- **New project** — asks your goal, enhances it, scaffolds all docs, generates a phased task plan
- **Existing project** — analyzes your codebase, extracts real architecture knowledge, fills docs with actual data, waits for your validation
- **Sync** — keeps docs in sync as code evolves, never overwriting your manual content

---

## Install

**Global** (available in all projects):

```bash
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/contextmap.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/contextmap.md
```

**Project-level** (this project only):

```bash
mkdir -p .claude/commands
curl -o .claude/commands/contextmap.md \
  https://raw.githubusercontent.com/yusheanfong/contextforge/main/.claude/commands/contextmap.md
```

**For existing project analysis**, you also need Python 3.10+. `/contextmap` installs Graphify automatically when it needs it.

---

## Usage

### New Project

In an empty or near-empty directory:

```
/contextmap
```

Or force new project mode:

```
/contextmap --new
```

Flow:
1. Asks: *"What's the final goal of this project?"*
2. Enhances your raw description into a polished 2–4 sentence goal — shows it to you for approval
3. Asks: *"What's the tech stack / language?"*
4. Scaffolds all doc files with your goal as the seed
5. Generates a realistic phased task list → shows it for your approval before saving
6. Installs a post-commit hook (silently rebuilds the graph on every commit)

No Python required for new projects. Graphify activates on first sync once you have code.

### Existing Project

In a project with source code (auto-detected):

```
/contextmap
```

Flow:
1. Checks Python ≥ 3.10 (fails with clear instructions if missing)
2. Installs Graphify if not present: `pip install graphifyy`
3. Runs `python -m graphify .` on your codebase
4. Reads the knowledge graph and presents a structured understanding:
   - Architecture pattern
   - Key components (god nodes — highest-degree concepts)
   - Main subsystems (community clusters)
   - Entry points
   - Public API surface
   - Dominant coding patterns
5. Asks: *"Does this match your understanding? What should I add or correct?"*
6. Applies your corrections, then populates all doc files
7. Installs the post-commit hook

### Sync

After code changes:

```
/contextmap sync
```

Flow:
1. Rebuilds the knowledge graph incrementally
2. Refreshes only the `<!-- graphify:auto -->` fenced sections in doc files
3. Preserves everything outside the fences — always
4. Prints a summary of what changed
5. Tombstones deleted modules: `<!-- graphify:removed: ModuleName (last seen: YYYY-MM-DD) -->`

---

## What Gets Created

```
your-project/
├── CLAUDE.md                    ← Auto-loaded every session (goal + architecture summary + rules)
├── .git/hooks/post-commit       ← Silently rebuilds graph on every commit
├── graphify-out/
│   ├── graph.json               ← Knowledge graph (Graphify-managed, don't edit)
│   └── GRAPH_REPORT.md          ← Human-readable graph summary
└── doc/
    ├── architecture.md          ← Tech stack, detected patterns, design rules
    ├── domain-model.md          ← Entities, relationships, business rules
    ├── api-contract.md          ← API surface, services, interfaces
    ├── solution-structure.md    ← Folder layout, dependency flow
    ├── coding-standard.md       ← Detected patterns + your standards
    ├── security.md              ← Auth components + your security rules
    ├── ui-guideline.md          ← UI components + your UX rules
    ├── task-list.md             ← Master task list (100% yours — never graph-populated)
    ├── changelog.txt            ← Updated after every change
    ├── progress.txt             ← Current status (kept short)
    └── Progress/
        └── Progress-N.txt       ← Per-task detailed logs
```

---

## How It Works

### The Knowledge Graph (Graphify)

[Graphify](https://github.com/safishamsi/graphify) is an open-source Python tool that parses your codebase via tree-sitter AST analysis (29+ languages) and builds a queryable knowledge graph:

| Concept | What it means |
|---------|---------------|
| **Node** | A concept: function, class, module, interface, file |
| **Edge** | A relationship: calls, imports, extends, implements |
| **Edge confidence** | `EXTRACTED` (AST-confirmed) / `INFERRED` / `AMBIGUOUS` |
| **Community** | A cluster of related nodes (Leiden algorithm) |
| **God node** | A highest-degree concept — something everything else depends on |

`/contextmap` reads `graphify-out/graph.json` and maps this data into your doc files.

### The Fence Protocol

Graph-managed content is wrapped in HTML comment fences:

```markdown
<!-- graphify:auto start:src/auth/auth_service:architecture -->
Content here is extracted from your code.
Overwritten on every /contextmap sync.
<!-- graphify:auto end:src/auth/auth_service:architecture -->
```

**Everything outside the fences is yours.** Claude never touches it during sync.

- Add notes, rules, and context anywhere outside the fences — they survive every sync
- Graph sections refresh automatically when you run `/contextmap sync`
- Fence keys are fully-qualified paths (`source_file:section`) — no collision between modules with the same short name

### The Post-Commit Hook

`.git/hooks/post-commit` runs `python -m graphify . --update` silently in the background after every commit. This keeps `graphify-out/graph.json` current without blocking your workflow.

The hook only rebuilds the graph — it never writes to doc files. Run `/contextmap sync` explicitly when you want docs refreshed.

### Session Flow

```
You write code → git commit
  └─→ post-commit hook: graphify --update (background, silent)

/contextmap sync
  └─→ reads updated graph.json
  └─→ refreshes <!-- graphify:auto --> sections
  └─→ your content outside fences: unchanged

Claude Code session start
  └─→ CLAUDE.md auto-loads (~600 tokens)
  └─→ Claude knows your architecture, key components, rules
  └─→ First message: already in context, no setup needed
```

---

## Why It Works

### Real data beats placeholders

Standard templates fill `architecture.md` with `[FILL IN: your tech stack]`. You do the work, Claude uses what you wrote — which may be incomplete or stale after a week of coding.

`/contextmap` extracts architecture from your actual code. It finds your central abstractions, maps module dependencies, and detects coding patterns. Docs reflect reality.

### Structured context = better decisions

Claude makes better decisions with accurate, structured context:
- Won't suggest patterns that contradict your existing architecture
- Won't invent schema fields that don't exist in your domain model
- Won't suggest refactors that break your layer rules

### Fenced sections prevent drift

Docs that require manual updates go stale. The fence protocol solves this: graph sections are always current, your manual sections are always preserved. Neither overwrites the other.

### God nodes = what matters most

Not all code is equally important. God nodes are the concepts with the most connections — what everything else depends on. These surface in `CLAUDE.md`, giving Claude immediate awareness of your architecture's center of gravity before you type anything.

---

## Keeping Claude on Track

`/contextmap` writes these rules into `CLAUDE.md`:

1. Read all `/doc` files before writing any code
2. Implement ONLY the next incomplete task from `task-list.md`
3. Update `progress.txt` after every completed task
4. Update `changelog.txt` after every change (`Date | Change | Description`)
5. Follow `solution-structure.md` exactly — no structural changes
6. Never invent schema, fields, or endpoints not defined in the docs

```
Session N:   auto-load CLAUDE.md → read task-list → implement → update progress + changelog → commit
Session N+1: fresh session → same auto-load → continue from progress.txt
```

---

## Requirements

- Claude Code (any version)
- Python 3.10+ (for Graphify — only needed for existing project analysis and sync; not required for new projects)

---

## License

MIT
