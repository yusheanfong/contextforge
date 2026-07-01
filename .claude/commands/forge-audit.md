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

`/forge-audit` is graph-driven. Use the Read tool (or a file check) for `graphify-out/graph.json` in the
current directory.

If it does NOT exist, print this exact message and STOP — do nothing else (do not auto-run
`/forge-contextmap`; the user builds the graph manually):

```
❌ /forge-audit is graph-driven and needs graphify-out/graph.json.
Run /forge-contextmap first to build the knowledge graph, then re-run /forge-audit.
```

### 0c. Resolve the Python interpreter as `[PYTHON_CMD]`

1. If `graphify-out/.graphify_python` exists, read it — that one line is the interpreter path
   graphify already validated (uv/pipx/venv-aware). Use it verbatim as `[PYTHON_CMD]`.
2. Otherwise, detect with the Bash tool:
   ```bash
   python --version 2>&1 || python3 --version 2>&1
   ```
   Use whichever of `python` / `python3` works and is ≥ 3.10. If neither is ≥ 3.10, print:
   ```
   ❌ /forge-audit needs Python 3.10+ to read the graph. Install it, then re-run.
   ```
   and STOP.

---

## PHASE 1: Graph Scan *(candidates only — the graph points, it never decides)*

Run the scan **inline via the Bash tool** — no file is written anywhere (single-pass + read-only, so
nothing is persisted to disk). Pass `[SCOPE]` as the one argument; empty string means whole repo.
The script loads the graph exactly as graphify/forge-contextmap do (`node_link_graph(..., edges='links')`)
and emits three candidate buckets.

```bash
[PYTHON_CMD] - "[SCOPE]" <<'PY'
import sys, json
from collections import defaultdict
from pathlib import Path
import networkx as nx
from networkx.readwrite import json_graph

scope = (sys.argv[1] if len(sys.argv) > 1 else "").replace("\\", "/").strip("/")

data = json.loads(Path('graphify-out/graph.json').read_text(encoding='utf-8'))
G = json_graph.node_link_graph(data, edges='links')

def label_of(n):
    return G.nodes[n].get('label', str(n))

def src_of(n):
    s = G.nodes[n].get('source_file')
    return s.replace("\\", "/") if s else None

def in_scope(n):
    s = src_of(n)
    if s is None:
        return False          # skip nodes with no source_file (guard like orchestrate's slice)
    return (not scope) or s.startswith(scope)

nodes = [n for n in G.nodes() if in_scope(n)]
if not nodes:
    print("SCOPE_EMPTY"); sys.exit(0)

# --- bucket 1: orphans / near-dead (degree <= 1) -> rung 1
orphans = sorted(
    [(label_of(n), src_of(n), G.degree(n)) for n in nodes if G.degree(n) <= 1],
    key=lambda t: (t[1] or "", t[0]),
)
print("ORPHANS")
if orphans:
    for lab, src, deg in orphans[:40]:
        print(f"  {lab} | {src} | degree={deg}")
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
print("DUPLICATES")
if dups:
    for lab, srcs in dups[:30]:
        print(f"  {lab} | in {len(srcs)} files: {', '.join(s for s in srcs if s)}")
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
print(f"GODNODES (degree>={cut}, median={med})")
if gods:
    for lab, src, deg in gods[:15]:
        print(f"  {lab} | {src} | degree={deg}")
else:
    print("  NO_SIGNAL")
PY
```

- **ORPHANS** (`degree <= 1`) → candidates for **rung 1** (YAGNI / dead code).
- **DUPLICATES** (same label across different files) → candidates for **rung 2** (should have reused).
- **GODNODES** (very high degree) → **structural / over-centralization** — reported separately in
  Phase 3, NOT forced onto a ladder rung.
- `SCOPE_EMPTY` means `[SCOPE]` matched no nodes — tell the user the scope hit nothing (check the
  path / slash style) and stop. A `NO_SIGNAL` under any bucket means honestly report "no signal
  here" rather than inventing findings.

These are POINTERS only. Nothing here is a finding yet — Phase 2 confirms each against real code.

---

## PHASE 2: Source Confirmation *(the code decides)*

**If `[GRAPH_ONLY]` is true, skip this phase** — carry every Phase 1 candidate straight into the
report tagged **`[unconfirmed]`**.

Otherwise, for each candidate, open its live `source_file` (Read tool) and confirm the finding
against the actual code before flagging it. Walk the ladder top-down, stop at the first rung that
applies:

1. **Does this need to exist?** → no ⇒ **delete** (YAGNI / dead code)
2. **Already in this codebase?** → ⇒ **reuse it** (duplication — inline/collapse to the original)
3. **Stdlib does it?** → ⇒ **replace-with-stdlib**
4. **Native platform feature?** → ⇒ **use it**
5. **Installed dependency?** → ⇒ **use-dependency**
6. **One line?** → ⇒ **inline**
7. **Only then:** the minimum that works — flag speculative abstraction, unrequested flexibility,
   dead scaffolding ⇒ **simplify**

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

### Summary count

End with:
```
Audit summary — scope: [SCOPE or "whole repo"]
  rung 1 (dead):        N
  rung 2 (duplication): N
  rung 3-7 (other):     N
  structural notes:     N
  files touched:        N        confirmed: N / unconfirmed: N
```

If the whole scan surfaced nothing worth flagging, say so plainly — a clean codebase is a valid
result, not a prompt to manufacture findings.

---

## HARD GUARDS

- **Read-only.** No source edits, no git writes, **no files written at all** — the scan is an inline
  heredoc that persists nothing.
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
3. Zero edits and zero commits were made (read-only confirmed) — including no scan file left on disk.

---

## NOTES

- **Report-only, single-pass.** No subagents, no branches, no gate loop — unlike `/forge-orchestrate`.
  `/forge-audit` reads the graph, confirms against source, and prints. That is the whole command.
- **What the graph can and can't surface.** Candidates come from three graph signals: orphan → rung
  1, duplicate label → rung 2, god node → structural. Rungs 3–6 (reinvented stdlib, native feature,
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
- It never rebuilds the graph and never auto-runs `/forge-contextmap`. If `graph.json` is missing it
  hard-stops and asks you to run `/forge-contextmap` manually.
