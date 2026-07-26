---
name: forge-audit
description: Repo-wide over-engineering audit (/forge-audit). Use when sweeping already-landed code for accumulated bloat — dead/orphan nodes, duplicate implementations, god nodes — and handing back a grouped delete/simplify list. Report-only, never edits. Triggers include "/forge-audit", "audit for over-engineering", "find dead code", "sweep the repo for bloat", "what can we delete".
argument-hint: "[path] [--graph-only]"
allowed-tools: Read, Grep, Glob, Bash, Write
---

# /forge-audit — Repo-Wide Over-Engineering Audit

Sweep the ENTIRE codebase (or a scoped path) for already-accumulated bloat and hand back a grouped
delete/simplify list with a rationale per finding. **Reports only — never edits, never commits.** An
on-demand cleanup tool.

Companion to `/forge-contextmap` (which builds `graphify-out/graph.json`) — `/forge-audit` *uses* that map to
point at candidates instead of blind-reading every file. Contrast with `/forge-orchestrate`: its 5b.3 gate
reviews a single worker's fresh diff at commit time; `/forge-audit` sweeps code that already landed.

Review criteria = the **minimal-code ladder** (walk top-down, stop at the first rung that applies).
*Ladder adapted from ponytail (dietrichgebert/ponytail, MIT).*

---

## PHASE 0: Preconditions & Args

### 0a. Parse `$ARGUMENTS`

- If it contains **`--graph-only`**, set `[GRAPH_ONLY] = true` and strip the flag. Otherwise
  `[GRAPH_ONLY] = false`. Under `[GRAPH_ONLY]` the audit skips source confirmation (Phase 2) —
  faster, less precise — and every finding is tagged **`[unconfirmed]`**.
- Whatever text remains (flags stripped) is the optional **scope** — a path prefix to limit the
  audit to. Store as `[SCOPE]`. If empty, `[SCOPE] = ""` meaning the whole repo.

### 0b. Graph prerequisite — hard-stop

<!-- forge:shared-block graph-hard-stop -->
`/forge-audit` is graph-driven. Use the Read tool (or a file check) for `graphify-out/graph.json` in the
current directory.

If it does NOT exist, print this exact message and STOP — do nothing else (do not auto-run
`/forge-contextmap`; the user builds the graph manually):

```
❌ /forge-audit is graph-driven and needs graphify-out/graph.json.
Run /forge-contextmap first to build the knowledge graph, then re-run /forge-audit.
```
<!-- /forge:shared-block graph-hard-stop -->

### 0c. Resolve the Python interpreter as `[PYTHON_CMD]`

<!-- forge:shared-block python-cmd -->
Graphify is normally installed with **uv** or **pipx**, which put it in its own virtualenv. That
venv's python has `networkx`; your system `python3` almost certainly does not. Resolving the wrong
one is the single most common way these scripts die. Work down this list and stop at the first
interpreter that passes step 5.

1. If `graphify-out/.graphify_python` exists, read it — that one line is an interpreter path graphify
   already validated. Use it verbatim. (Recent graphify CLI versions do **not** write this file, so
   expect it to be missing and keep going.)
2. **uv** — ask uv where its tools live:
   ```bash
   uv tool dir
   ```
   Then try, in order, `<that dir>/graphifyy/bin/python` (macOS/Linux) and
   `<that dir>\graphifyy\Scripts\python.exe` (Windows). A path that does not exist just errors —
   harmless, move on.
3. **pipx** — same idea:
   ```bash
   pipx environment --value PIPX_LOCAL_VENVS
   ```
   Try `<that dir>/graphifyy/bin/python` and `<that dir>\graphifyy\Scripts\python.exe`.
4. Fall back to the bare interpreter. Run these as **two separate calls** — `||` is not available in
   PowerShell 5.1, and the first that succeeds wins:
   ```bash
   python --version
   ```
   ```bash
   python3 --version
   ```
5. **Verify the candidate before accepting it** — the Phase 1 scan needs them:
   ```bash
   [PYTHON_CMD] -c "import networkx, json"
   ```
   If it fails, this is the wrong interpreter — go back and try the next candidate. If **every**
   candidate fails, print:
   ```
   ❌ /forge-audit found no Python with networkx. Graphify installs one in its own venv —
      try: uv tool install graphifyy   (or: pipx install graphifyy)
   ```
   and STOP. Always report which interpreter you settled on.
<!-- /forge:shared-block python-cmd -->

---

## PHASE 1: Graph Scan *(candidates only — the graph points, it never decides)*

Write the scan to `graphify-out/.forge_audit_scan.py` with the **Write tool**, run it, then delete
it. (A heredoc would be shorter but does not exist in PowerShell or cmd — this skill must run on
Windows too. Same pattern `/forge-orchestrate` uses for `.orchestrate_slice.py`.) Pass `[SCOPE]` as
the one argument; empty string means whole repo. The script loads the graph exactly as
graphify/forge-contextmap do (`node_link_graph(..., edges='links')`) and emits three candidate
buckets.

```python
import sys, json
from collections import defaultdict
from pathlib import Path
import networkx as nx
from networkx.readwrite import json_graph

scope = (sys.argv[1] if len(sys.argv) > 1 else "").replace("\\", "/").strip("/")

# forge:shared-block graph-loader
DOC_SUFFIXES = {".md", ".mdx", ".qmd", ".skill"}

data = json.loads(Path('graphify-out/graph.json').read_text(encoding='utf-8'))
G = json_graph.node_link_graph(data, edges='links')

def label_of(n):
    return G.nodes[n].get('label', str(n))

def src_of(n):
    s = G.nodes[n].get('source_file')
    return s.replace("\\", "/") if s else None

def is_doc(n):
    # S2.5 already dropped these, but the post-commit hook's `graphify update`
    # re-adds them on the next commit — so filter at read time too. file_type
    # alone is not enough: with an LLM backend the semantic pass mints
    # file_type="code" nodes for symbols surfaced from inside a doc.
    ft = G.nodes[n].get('file_type')
    if ft is not None and ft != 'code':
        return True
    s = src_of(n)
    return s is not None and Path(s).suffix.lower() in DOC_SUFFIXES


# Drop doc nodes from the GRAPH, not just from the node list. Filtering membership
# alone leaves their edges in place, so G.degree() still counts references from
# markdown (inflating degree, hiding orphans, skewing the god-node cut) and
# G.neighbors() still yields them (making a dead file look referenced). No-op when
# S2.5 already pruned; load-bearing when it has not.
G = G.subgraph([n for n in G.nodes() if not is_doc(n)])
# /forge:shared-block graph-loader

def in_scope(n):
    s = src_of(n)
    if s is None:
        return False          # skip nodes with no source_file (guard like orchestrate's slice)
    return (not scope) or s.startswith(scope)

nodes = [n for n in G.nodes() if in_scope(n)]
if not nodes:
    print("SCOPE_EMPTY"); sys.exit(0)

# forge:shared-block bloat-buckets
# --- bucket 1: orphans / near-dead (degree <= 1) -> rung 1
orphans = sorted(
    [(label_of(n), src_of(n), G.degree(n)) for n in nodes if G.degree(n) <= 1],
    key=lambda t: (t[1] or "", t[0]),
)
print(f"ORPHANS (total={len(orphans)})")
if orphans:
    for lab, src, deg in orphans[:40]:
        print(f"  {lab} | {src} | degree={deg}")
    if len(orphans) > 40:
        print(f"  ... {len(orphans) - 40} more not shown")
else:
    print("  NO_SIGNAL")

# --- bucket 2: duplication (same label across different source_files) -> rung 2
by_label = defaultdict(set)
for n in nodes:
    by_label[label_of(n)].add(src_of(n))
dups = sorted(
    [(lab, sorted(srcs)) for lab, srcs in by_label.items() if len(srcs) > 1],
    key=lambda t: (-len(t[1]), t[0]),
)
print(f"DUPLICATES (total={len(dups)})")
if dups:
    for lab, srcs in dups[:30]:
        print(f"  {lab} | in {len(srcs)} files: {', '.join(s for s in srcs if s)}")
    if len(dups) > 30:
        print(f"  ... {len(dups) - 30} more not shown")
else:
    print("  NO_SIGNAL")

# --- bucket 3: god nodes (top-decile degree, and >= 3x median) -> structural, NOT a rung
degs = sorted((G.degree(n) for n in nodes), reverse=True)
med = degs[len(degs) // 2] if degs else 0
cut = max(degs[max(0, len(degs) // 10 - 1)] if degs else 0, 3 * med, 5)
gods = sorted(
    [(label_of(n), src_of(n), G.degree(n)) for n in nodes if G.degree(n) >= cut],
    key=lambda t: -t[2],
)

# --- bucket 4: dead FILES (no edge leaves the file) -> rung 1
# `degree <= 1` structurally cannot catch these: a module node earns degree from
# `contains` edges to its own symbols, so a wholly unreferenced file still scores
# >= 2. The signal is EXTERNAL degree — an edge crossing to another source_file.
TEST_HINTS = ('tests/', 'test/', 'spec/', '/test_', '/spec_')
by_file = defaultdict(set)
for n in nodes:
    by_file[src_of(n)].add(n)


def is_test_path(s):
    low = '/' + s.lower()
    name = low.rsplit('/', 1)[-1]
    return (any(h in low for h in TEST_HINTS)
            or name.startswith('test_') or name.startswith('spec_')
            or '_test.' in name or '_spec.' in name)


dead_files = []
for s, members in by_file.items():
    if is_test_path(s):
        continue          # HARD GUARD: tests are never delete-listed
    external = any(
        src_of(nb) != s
        for n in members for nb in G.neighbors(n)
        if src_of(nb)
    )
    if not external:
        dead_files.append(s)
dead_files.sort()
# /forge:shared-block bloat-buckets
print(f"GODNODES (total={len(gods)}, degree>={cut}, median={med})")
if gods:
    for lab, src, deg in gods[:15]:
        print(f"  {lab} | {src} | degree={deg}")
    if len(gods) > 15:
        print(f"  ... {len(gods) - 15} more not shown")
else:
    print("  NO_SIGNAL")

print(f"DEAD_FILES (total={len(dead_files)})")
if dead_files:
    for s in dead_files[:20]:
        print(f"  {s} | {len(by_file[s])} node(s), no edge leaves the file")
    if len(dead_files) > 20:
        print(f"  ... {len(dead_files) - 20} more not shown")
else:
    print("  NO_SIGNAL")
```

```bash
[PYTHON_CMD] graphify-out/.forge_audit_scan.py "[SCOPE]"
```

Delete `graphify-out/.forge_audit_scan.py` once the scan output is read.

- **ORPHANS** (`degree <= 1`) → candidates for **rung 1** (YAGNI / dead code).
- **DUPLICATES** (same label across different files) → candidates for **rung 2** (should have reused).
- **GODNODES** (very high degree) → **structural / over-centralization** — reported separately in
  Phase 3, NOT forced onto a ladder rung.
- **DEAD_FILES** (no edge leaves the file) → candidates for **rung 1**, at *file* granularity.
  ORPHANS cannot find these: a module node earns degree from `contains` edges to its own symbols, so
  a file nothing references still scores `degree >= 2`. Test paths are excluded by construction.
- `SCOPE_EMPTY` means `[SCOPE]` matched no nodes — tell the user the scope hit nothing (check the
  path / slash style) and stop. A `NO_SIGNAL` under any bucket means honestly report "no signal
  here" rather than inventing findings.
- **Note any `... N more not shown` line explicitly in the report.** The per-bucket print caps
  (40 / 30 / 15 / 20) mean a truncated bucket is a partial answer — say "N more candidates were not
  examined", and offer to re-run with a narrower `[SCOPE]`. Never let a cap read as "that's all of
  them."

These are POINTERS only. Nothing here is a finding yet — Phase 2 confirms each against real code.

---

## PHASE 2: Source Confirmation *(the code decides)*

**If `[GRAPH_ONLY]` is true, skip this phase** — carry every Phase 1 candidate straight into the
report tagged **`[unconfirmed]`**.

Otherwise, for each candidate, open its live `source_file` (Read tool) and confirm the finding
against the actual code before flagging it. Walk the ladder top-down, stop at the first rung that
applies:

<!-- forge:shared-block minimal-ladder -->
1. **Does this need to exist?** → no ⇒ **delete** (YAGNI / dead code)
2. **Already in this codebase?** → ⇒ **reuse it** (duplication — inline/collapse to the original)
3. **Stdlib does it?** → ⇒ **replace-with-stdlib**
4. **Native platform feature?** → ⇒ **use it**
5. **Installed dependency?** → ⇒ **use-dependency**
6. **One line?** → ⇒ **inline**
7. **Only then:** the minimum that works — flag speculative abstraction, unrequested flexibility,
   dead scaffolding ⇒ **simplify**
<!-- /forge:shared-block minimal-ladder -->

Drop any candidate the source proves legitimate. A graph orphan that is a real public entry point, a
duplicate label that is genuinely two different things, a god node that is a legitimately central
module — none of these are findings. **The graph points; the code decides.**

---

## PHASE 3: Report *(grouped delete/simplify list)*

Print two sections.

### Ladder findings — grouped by rung (files within)

Each finding is one line:
```
[file:node/symbol] · rung N · <one-line rationale> · action: <delete|inline|replace-with-stdlib|use-dependency|simplify>
```
Tag `[unconfirmed]` on every line if `[GRAPH_ONLY]` was set.

### Structural notes — god nodes (no rung)

Each line names the over-centralized module + its degree + a **decompose?** question:
```
[file:module] · degree=N · possible over-centralization — decompose?
```
Apply the guard hardest here: a high-degree module is the **most likely false positive** of the three
signals. Frame each as a question, never a delete.

A `DEAD_FILES` candidate is a **rung 1 / action: delete** finding at file granularity, and gets the
same Phase 2 treatment as any other: open the file, confirm nothing references it, and grep the
repo for its symbols before flagging. Dynamic entry points (CLI registrations, plugin discovery,
reflection, DI containers) are invisible to the AST and are the expected false positive here.

### Summary count

End with:
```
Audit summary — scope: [SCOPE or "whole repo"]
  rung 1 (dead):        N        of which whole files: N
  rung 2 (duplication): N
  rung 3-7 (other):     N
  structural notes:     N
  files touched:        N        confirmed: N / unconfirmed: N
  not examined:         N        [omit if no bucket was truncated]
```

If the whole scan surfaced nothing worth flagging, say so plainly — a clean codebase is a valid
result, not a prompt to manufacture findings.

---

## HARD GUARDS

- **Read-only with respect to your code.** No source edits, no git writes, no report file. `Edit` is
  omitted from `allowed-tools`, so no existing file can be modified. `Write` is present for exactly
  one purpose — emitting `graphify-out/.forge_audit_scan.py`, which is deleted once its output is
  read. Writing the scan to a file instead of piping a heredoc is what makes this skill run on
  Windows; `/forge-orchestrate` uses the same pattern for `.orchestrate_slice.py`. **Never use
  `Write` for anything else here** — no report, no findings file, no scratch notes. The guarantee
  that matters is unchanged: `/forge-audit` never touches a byte of your source or your git history.
- **Never delete-list:** tests, or any file needed for current behavior / success criteria.
  Minimality never overrides working functionality — flag genuine bloat, not "code I'd have written
  differently."
- **Adaptive + honest.** If a graph region gives no useful signal, say so (`NO_SIGNAL`) rather than
  inventing findings.

---

## VERIFY WHEN DONE

1. With no `graphify-out/graph.json` present, Phase 0b prints the exact hard-stop message and halts.
2. With a graph present, the run produces a grouped list with a per-finding rung + rationale, and
   never flags a test file.
3. Zero source edits and zero commits were made — and `graphify-out/.forge_audit_scan.py` was
   deleted, so no scan file is left on disk.

---

## NOTES

- **Report-only, single-pass.** No subagents, no branches, no gate loop — unlike `/forge-orchestrate`.
  `/forge-audit` reads the graph, confirms against source, and prints. That is the whole command.
- **Documentation is filtered out at load time.** `graphify update` indexes markdown (it has no
  `--code-only`), so `CLAUDE.md` and `doc/*.md` — the files `/forge-contextmap` itself wrote — come
  back as graph nodes whose headings would otherwise dominate ORPHANS and DUPLICATES. `is_doc()` in
  the `graph-loader` block drops them. This is a read-time filter because the post-commit hook
  re-adds them after every commit and `/forge-audit` never prunes.
- **What the graph can and can't surface.** Candidates come from four graph signals: orphan → rung
  1, duplicate label → rung 2, dead file → rung 1, god node → structural. Rungs 3–6 (reinvented
  stdlib, native feature,
  existing dependency, one-liner) have **no discovery mechanism of their own** — they're caught
  opportunistically *while confirming* a graph-surfaced candidate, not by a repo-wide line sweep.
  The ladder is the *evaluation* applied to candidates, not a sweep for every rung. This is
  deliberate (the point is to NOT blind-read every file) — state it, don't oversell `/forge-audit` as
  "reads every line."
- **Duplication is heuristic.** Same-label-different-file is a *pointer*; two things sharing a name
  may be genuinely distinct. Always confirmed in Phase 2 (unless `--graph-only`), never flagged from
  the graph alone.
- **God nodes are the softest signal.** High degree often means legitimately central, not bloated.
  Reported as a question, outside the ladder taxonomy.
- **Only `DEAD_FILES` excludes test paths in Phase 1** — it has to, because a test file legitimately
  has no inbound edges (nothing imports it; the runner discovers it), so every test would otherwise
  land in the bucket. `ORPHANS` and `GODNODES` do **not** filter test paths: on a large repo, test
  helpers will appear in ORPHANS and test files in GODNODES. That is not a bug in the scan — the
  HARD GUARD ("never delete-list tests") is enforced in Phase 2/3, where you drop them. Watch for it.
- It never rebuilds the graph and never auto-runs `/forge-contextmap`. If `graph.json` is missing it
  hard-stops and asks you to run `/forge-contextmap` manually.
