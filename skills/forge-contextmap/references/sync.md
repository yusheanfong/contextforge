## SYNC MODE

*Trigger: `graphify-out/graph.json` exists, or `sync` argument*

> **Portability contract.** Every command in this file must run identically on macOS, Linux, and
> Windows. That means: **no heredocs, no `cp`/`mv`/`rm`, no `mkdir -p`/`chmod`, no `2>/dev/null`, no
> `||` chaining.** Multi-line Python goes into a file written with the **Write tool** and is run as
> `[PYTHON_CMD] <script>.py`; file copies use the **Read + Write tools**. The only shell commands
> left are `git …`, `graphify …`, and `[PYTHON_CMD] <script>.py`.

### Step S1: Resolve the Python Interpreter

<!-- forge:shared-block python-cmd variant:contextmap -->
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
5. **Verify the candidate before accepting it** — every later step in this file runs a script that needs them:
   ```bash
   [PYTHON_CMD] -c "import networkx, json"
   ```
   If it fails, this is the wrong interpreter — go back and try the next candidate. If **every**
   candidate fails, print:
   ```
   ❌ /forge-contextmap found no Python with networkx. Graphify installs one in its own venv —
      try: uv tool install graphifyy   (or: pipx install graphifyy)
   ```
   and STOP. Always report which interpreter you settled on.
<!-- /forge:shared-block python-cmd -->

### Step S1.5: Resolve the Graphify CLI

<!-- forge:shared-block graphify-cli -->
Graphify's CLI is **verb-based** (`graphify update <path>`), and its flags have changed across
versions. Never hardcode an invocation — resolve it once, then reuse.

1. If `graphify-out/.graphify_cli` exists, read it and skip to step 5. It has two lines:
   ```
   build=<command>
   update=<command>
   ```
2. Probe the console script:
   ```bash
   graphify --version
   ```
   If the command is not found, graphify is not installed — print the install guidance from
   [`references/existing-project.md`](existing-project.md) Step E2 and STOP. Call the resolved command `[GRAPHIFY]`.
3. Read the command list — it is the authority, not this file:
   ```bash
   graphify --help
   ```
   Confirm three things and note what you find:
   - **The build verb, selected by corpus composition.** Count files in the `.graphifyignore`-scoped
     corpus by extension before the first build, using the same case-insensitive shell-level counts
     as SKILL.md Step 1. Documentation is
     `.md`, `.mdx`, `.qmd`, `.skill` — the suffix half of S2.5's `is_doc` test; `DOC_TYPES` cannot
     exist until after extraction. The source suffixes are the source list from SKILL.md Step 1.
     When the source count is zero, use `graphify update <path>`, which cold-builds AST-extractable
     documents without an LLM backend. The boundary is zero rather than a majority because
     `extract` already produces a graph when any detected source file is present, while S2.5 would
     strip the doc nodes added by `update` from that mixed graph anyway. Otherwise use `graphify
     extract <path>` — "headless full extraction (AST + semantic LLM)" — and append
     **`--code-only`** when no LLM API key is configured; it indexes real source by local AST with no
     API call. There is **no bare `graphify <path>` form.**
   - **The shrink-override flag on `update`.** `graphify update` is incremental and **refuses to
     overwrite `graph.json` when the rebuild has fewer nodes than the existing graph.** As of 0.9.26
     the override is `--force` (`overwrite graph.json even if the rebuild has fewer nodes`; env var
     `GRAPHIFY_FORCE=1`). Use whatever name *this* version's help shows. If no such flag exists,
     record none — do **not** invent one.
   - **Whether `cluster-only <path>` exists** (S2.5 re-clusters with it).
4. Write `graphify-out/.graphify_cli` with the **Write tool** (create `graphify-out/` first if the
   directory does not exist — in EXISTING PROJECT MODE it will not until graphify's first run):
   ```
   build=[GRAPHIFY] update .                                      <no source files>
   build=[GRAPHIFY] extract . <--code-only if no LLM backend is configured>  <one or more source files>
   update=[GRAPHIFY] update . <force-flag if step 3 found one>
   ```
   Write exactly one `build=` line: the branch selected from the corpus counts.
5. Use those two commands verbatim wherever this skill needs to build or update the graph.

*Why probe instead of hardcode:* `python -m graphify . --update` — the invocation this skill used to
ship — matches no released graphify CLI. It fails, or silently does nothing when backgrounded. The
verb set has also churned across versions, so read `graphify --help` rather than trusting this list.
<!-- /forge:shared-block graphify-cli -->

### Step S2: Rebuild Graph

Run the `update=` command resolved in S1.5. Wait for completion and read its output — graphify's
update path reports its own pruning, e.g. `Pruned N ghost node(s) from M deleted file(s)`.

If the update reports that it **refused to write** because the rebuild had fewer nodes, and S1.5
found no override flag, note it — S2.5 handles the leftover stale nodes.

### Step S2.5: Prune Stale Nodes

This step does two things: it drops **stale** nodes, and it drops **doc-derived** nodes.

*Stale.* `graphify update` prunes nodes for files it knows were deleted, but nodes survive when the
update never ran or when its shrink guard blocked the write-back. (Files that left the corpus via
`.graphifyignore` are handled by `update`'s own corpus sweep, not here — an ignored file still
exists on disk, so an existence check cannot see it.) Those stale nodes show up in every doc fence
and make the S3.5 diff meaningless.

*Doc-derived.* `graphify update` has **no `--code-only`** — it indexes markdown too, so the docs this
skill just wrote come back as graph nodes on the next rebuild. Left in, they rewrite every fence with
descriptions of the fences, and they swamp the S3.6 / `/forge-audit` bloat buckets with markdown
headings. Dropping them here is what makes a no-op sync a genuine no-op.

Write this to `graphify-out/.forge_prune.py` with the **Write tool**, run it, then delete it:

```python
"""Drop stale + doc-derived graph nodes. Stdlib only, no networkx."""
import json
import sys
from pathlib import Path

# A real refactor can legitimately delete most of a corpus (observed: 61%).
# The failure this guards against — a bad root, or Windows separators read on POSIX —
# has a different signature: NOTHING resolves. Calibrate for that, not for "a lot".
MAX_DROP_RATIO = 0.9

# graphify's own doc dispatch (extract.py _DISPATCH): the suffixes routed to
# extract_markdown, i.e. exactly the files `graphify update` mints heading nodes for.
# `.txt` is deliberately absent — it has no AST extractor, so changelog.txt /
# progress.txt produce no nodes, and dropping it would delete real semantic nodes
# from a user's .txt corpus.
DOC_SUFFIXES = {".md", ".mdx", ".qmd", ".skill"}

# file_type values that can ONLY come from a non-code file. `concept` and
# `rationale` are deliberately NOT here: graphify mints them from Python
# docstrings too, and those are code-derived content worth keeping in the graph.
DOC_TYPES = {"document", "paper", "image"}

graph_path = Path(sys.argv[1] if len(sys.argv) > 1 else "graphify-out/graph.json").resolve()
# graph.json lives at <repo-root>/graphify-out/graph.json — source_file values are
# repo-root-relative, so resolve against the root, not the cwd.
root = graph_path.parent.parent

data = json.loads(graph_path.read_text(encoding="utf-8"))
nodes = data.get("nodes", [])
total = len(nodes)


def src_of(n):
    s = n.get("source_file")
    # Windows-built graphs store lib\foo.dart; normalize before touching the filesystem.
    return s.replace("\\", "/") if s else None


def is_doc(n):
    """True when the node came from a documentation FILE.

    Keyed on the source file, not on file_type, because both directions are wrong:
      - a doc-sourced node can be file_type="code" (with an LLM backend the semantic
        pass mints code-typed nodes for symbols surfaced from inside a .md), and
      - a code-sourced node can be file_type="rationale" (a Python module docstring).
    Only document/paper/image are unambiguous on their own.
    """
    if n.get("file_type") in DOC_TYPES:
        return True
    s = src_of(n)
    # .lower() is load-bearing: macOS is case-insensitive, so README.MD is a real
    # file, and graphify AST-extracts it (extract.py falls back to suffix.lower()).
    return s is not None and Path(s).suffix.lower() in DOC_SUFFIXES


stale_ids, stale_files = set(), set()
for n in nodes:
    s = src_of(n)
    if s is None:
        continue  # synthetic / concept node — never stale
    if not (root / s).exists():
        stale_ids.add(n["id"])
        stale_files.add(s)

sourced = sum(1 for n in nodes if src_of(n) is not None)
ratio = len(stale_ids) / sourced if sourced else 0
if ratio > MAX_DROP_RATIO:
    print(
        f"REFUSED: prune would drop {len(stale_ids)}/{sourced} source-backed nodes "
        f"({ratio:.0%}) from {len(stale_files)} missing file(s). Nearly nothing resolved "
        f"on disk — that is a root/separator bug, not real deletions."
    )
    print(f"  graph root resolved to: {root}")
    print("  sample missing: " + ", ".join(sorted(stale_files)[:5]))
    print("Graph left unchanged.")
    sys.exit(2)

# Doc drop is accounted SEPARATELY from stale — folding it into MAX_DROP_RATIO would
# trip the refusal on a doc-heavy repo, which is not the failure that guard exists for.
doc_ids = {n["id"] for n in nodes if is_doc(n)}
if doc_ids and len(doc_ids) * 2 > total:
    # Mostly-prose corpus: dropping its majority content would empty its doc fences.
    print(
        f"Doc filter disabled: documentation is the majority ({len(doc_ids)}/{total} nodes); "
        f"{len(doc_ids - stale_ids)} non-stale doc node(s) kept."
    )
    doc_ids = set()

drop = stale_ids | doc_ids
if not drop:
    print(f"No stale nodes. ({total} nodes)")
    sys.exit(0)

data["nodes"] = [n for n in nodes if n["id"] not in drop]
data["links"] = [
    e
    for e in data.get("links", [])
    if e.get("source") not in drop and e.get("target") not in drop
]
graph_path.write_text(json.dumps(data), encoding="utf-8")
if stale_ids:
    print(f"Pruned {len(stale_ids)} node(s) from {len(stale_files)} deleted file(s).")
if doc_ids:
    print(f"Dropped {len(doc_ids)} non-code node(s) (docs).")
print(f"{len(data['nodes'])} node(s) remain.")
```

```bash
[PYTHON_CMD] graphify-out/.forge_prune.py graphify-out/graph.json
```

Five guards are load-bearing — do not simplify them away:

- **Separator normalization.** A graph built on Windows stores `lib\foo.dart`. Without
  `.replace("\\", "/")`, a sync run from macOS finds *nothing* on disk and prunes the whole graph.
- **`source_file is None` is skipped.** Synthetic and concept nodes have no source file and are
  never stale.
- **`MAX_DROP_RATIO`.** Measured against *source-backed* nodes only, at 90%, and against **stale
  only**. A real refactor can legitimately delete most of a corpus — 61% was observed in testing and
  must pass. The failure being guarded against (bad root, or Windows separators read on POSIX) looks
  different: **nothing** resolves. Tuning this below ~0.9 turns ordinary cleanups into false refusals.
- **`is_doc` checks the suffix, not just `file_type`.** With an LLM backend configured, graphify's
  semantic pass mints `file_type="code"` nodes for symbols it surfaced from *inside* a `.md` file. A
  `file_type == 'code'` test alone passes those straight through. Suffix comparison is lowercased
  because macOS is case-insensitive and graphify itself dispatches on `suffix.lower()`.
- **The majority-doc floor.** If doc-derived nodes are more than half of all graph nodes, the filter
  disables itself. Total nodes are the denominator because this classifies the graph consumed by
  the fences; incidental code nodes must not make a prose corpus discard its content. The printed
  kept count excludes stale docs, which the stale-node path still handles separately.

**Expect the node count to oscillate, and do not "fix" it.** The post-commit hook runs
`graphify update`, which re-adds doc nodes; the next sync drops them again. The oscillation lives
entirely inside `graphify-out/` — a doc fence only ever sees the pruned graph, because S3 reads
`graph.json` after this step.

**If it prints `REFUSED`:** the script already left the graph untouched. Print its message
prominently, then **continue into S3 with the unpruned graph** — do not halt the sync. The most
likely cause is not a prune bug but a legitimate large deletion or a new `.graphifyignore` entry,
and halting would also block the doc refresh, which has nothing to do with pruning. S3.5's own >20%
branch catches the resulting diff blowup. Repeat the refusal in the S5 report so it is not lost.

**If nodes were pruned**, community assignments are now stale. Re-cluster using the `[GRAPHIFY]`
command resolved in S1.5 — same rule as everywhere else, do not hardcode the invocation:

```bash
graphify cluster-only .
```

Confirm `cluster-only` is a subcommand on this version first (it appears in `graphify --help`).
If it is not, keep the pruned graph but have **S4 omit the community-derived lines** from every
fence. Writing known-wrong community ids into ten docs is worse than writing fewer lines. Say so in
the S5 report.

Delete `graphify-out/.forge_prune.py` when done.

### Step S3: Parse Updated Graph

Re-parse the pruned `graphify-out/graph.json` using the **same `.forge_parse.py` script** as Step E4
in [`references/existing-project.md`](existing-project.md) — write it, run it, read its output, delete it. **Do not open
`graph.json` with the Read tool**; it is ~424 KB on a small repo.

### Step S3.5: Diff Graph and Draft Changelog

Auto-draft structural changelog entries from what actually changed, so the log captures real
mutations instead of vague summaries.

Both graphs are ~500 KB on a small repo, so neither one goes through the Read tool. Write this to
`graphify-out/.forge_diff.py` with the **Write tool**, run it, then delete it. It prints the diff
**and** rewrites the baseline in one pass:

```python
"""Diff graph.json against graph.prev.json, then refresh the baseline. Stdlib only."""
import json
import shutil
import sys
from pathlib import Path

out = Path("graphify-out")
cur_path, prev_path = out / "graph.json", out / "graph.prev.json"


def keys(path):
    data = json.loads(path.read_text(encoding="utf-8"))
    # Identity is label@source_file. Normalize separators so a Windows-built
    # baseline does not read as "everything removed, everything added" on POSIX.
    return {
        f"{n.get('label')}@{(n.get('source_file') or '').replace(chr(92), '/')}": n
        for n in data.get("nodes", [])
    }


cur = keys(cur_path)
if not prev_path.exists():
    print("FIRST_SYNC — no baseline; diff skipped.")
else:
    prev = keys(prev_path)
    added = sorted(set(cur) - set(prev))
    removed = sorted(set(prev) - set(cur))
    print(f"added={len(added)} removed={len(removed)} current={len(cur)}")
    for tag, items in (("ADDED", added), ("REMOVED", removed)):
        for k in items[:10]:
            print(f"  {tag} {k}")
        if len(items) > 10:
            print(f"  ... {len(items) - 10} more {tag}")

# shutil.copyfile, not a shell `cp` — `cp` does not exist in cmd and must never
# silently no-op. Copy LAST, so a crash above leaves the old baseline intact.
shutil.copyfile(cur_path, prev_path)
print("baseline updated")
```

```bash
[PYTHON_CMD] graphify-out/.forge_diff.py
```

1. **If it prints `FIRST_SYNC`**: there was no baseline; it has just been created. Note "changelog
   baseline created — no diff yet" and go to step 4.
2. **Otherwise** read `added` / `removed` from the output. For file-level context, also run
   (best-effort — ignore failure): `git diff --name-only HEAD~1 HEAD`
3. **Baseline-mismatch check.** Both sides of this diff must be *pruned* graphs (S2.5 guarantees the
   current side). If `graph.prev.json` was written before S2.5 existed, the first pruned sync shows a
   large one-time burst of `[Removed]` for nodes that were already stale, and a baseline that was
   itself pruned against an unpruned current would invent phantom `[Added]`. When either list
   exceeds ~20% of `current`, do **not** write thousands of entries — write one line instead:
   ```
   [TODAY'S DATE] | Baseline realigned | First pruned sync — baseline rewritten, diff resumes next sync
   ```
   and skip to step 4. **This suppresses the changelog entries only — not S4.** Carry the `removed`
   list forward as usual; S4 step 5 still writes tombstones for anything that appeared in a fence.
   A realigned baseline is exactly when a reader most needs to know a module left deliberately.
4. **Append a draft block** to `doc/changelog.txt`. This file is user-owned, so mark the block clearly as an editable draft. Use the existing `Date | Change | Description` line style:
   ```
   [TODAY'S DATE] | Auto-draft (review/edit) | [Added] <label> (<source_file>) ; [Removed] <label> (<source_file>)
   ```
   List the most significant added/removed nodes (cap ~10 each to stay readable). If nothing structural changed, write one line noting "no structural changes detected."
5. Delete `graphify-out/.forge_diff.py`. The baseline was already refreshed by the script — there is
   no separate copy step.

Keep the `removed` list in memory — Step S4 uses it for tombstones.

### Step S3.6: Bloat Signal (counts only)

The graph is already loaded — surface the three `/forge-audit` signals cheaply so accumulated bloat
gets noticed without waiting for someone to remember to audit. **Counts only.** Do not open a single
source file, do not evaluate anything against the ladder, do not report findings — that is
`/forge-audit`'s job, and doing it here would turn a routine sync into a slow interactive review.

Write this to `graphify-out/.forge_bloat.py` with the **Write tool**, run it, then delete it:

```python
import json
from collections import defaultdict
from pathlib import Path
from networkx.readwrite import json_graph

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
    # Deliberately STRICTER than S2.5's prune: the buckets below count auditable
    # code symbols, and a docstring-derived `rationale` node is not one — it would
    # land in ORPHANS as permanent noise. S2.5 keeps those in graph.json (they are
    # real code-derived content for the fences); this drops them from the counts.
    # Suffix check is still needed: with an LLM backend the semantic pass mints
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

nodes = [n for n in G.nodes() if src_of(n)]

# forge:shared-block bloat-buckets — thresholds MUST match forge-audit/SKILL.md Phase 1
orphans = [n for n in nodes if G.degree(n) <= 1]

by_label = defaultdict(set)
for n in nodes:
    by_label[label_of(n)].add(src_of(n))
dups = [lab for lab, srcs in by_label.items() if len(srcs) > 1]

degs = sorted((G.degree(n) for n in nodes), reverse=True)
med = degs[len(degs) // 2] if degs else 0
cut = max(degs[max(0, len(degs) // 10 - 1)] if degs else 0, 3 * med, 5)
gods = [n for n in nodes if G.degree(n) >= cut]

# Dead FILES. `degree <= 1` structurally cannot catch these: a module node earns
# degree from `contains` edges to its own symbols, so a wholly unreferenced file
# still scores >= 2. The signal is EXTERNAL degree — an edge leaving the file.
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

print(f"BLOAT {len(orphans)} {len(dups)} {len(gods)} {len(dead_files)}")
```

```bash
[PYTHON_CMD] graphify-out/.forge_bloat.py
```

Carry the four numbers into the S5 report, then delete the script. Best-effort — if it fails, omit
the line rather than blocking the sync.

### Step S4: Fence-Aware Merge

For each doc file that has `<!-- graphify:auto start:... -->` markers (v2 set:
`CLAUDE.md` claude-summary, `architecture.md`, `domain-model.md`, `api-contract.md`,
`solution-structure.md`, `coding-standard.md`, `security.md`, `app-flow.md`, `design-brief.md`,
`backend-schema.md` — whichever exist):

1. Read the entire file
2. Find all fence pairs: `<!-- graphify:auto start:KEY -->` ... `<!-- graphify:auto end:KEY -->`
3. **Curated-prose check — do this BEFORE regenerating.** Fence content is replaced wholesale, and
   AST data cannot reproduce a hand-written sentence. Scan the OLD fence content for lines that are
   not graph-derivable (anything that is not a god-node list, community list, entry-point list,
   detected-component list, or a count). If any are found, print this once per fence and **carry
   those lines through unchanged into the new fence content for this run only**:
   ```
   ⚠️ doc/X.md fence project:Y has hand-written lines the graph cannot reproduce:
        "<line>"
      Move them below the <!-- graphify:auto end --> marker — fence content is
      regenerated on every sync and this rescue does not repeat.
   ```
   This turns silent data loss into a visible prompt without weakening the contract: the fence still
   means "regenerated," the user is told exactly what to move and where.
4. For each fence pair:
   - Generate new content from the updated graph data — graph-derived facts only
   - Replace ONLY the content between the start and end markers
   - Keep the markers themselves intact
   - Keep all content outside the markers exactly as-is
   - If S2.5 could not re-cluster, omit the community lines rather than emitting stale ids
5. **Tombstones**: if a node from the S3.5 `removed` list appeared in this fence's PREVIOUS
   content, append at the end of the new fence content. This runs even when S3.5 step 3 took the
   `Baseline realigned` path — that branch suppresses changelog *entries*, never tombstones.
   ```
   <!-- graphify:removed: <label> (last seen: YYYY-MM-DD) -->
   ```
   Also carry over any `graphify:removed` lines already present in the old fence content. Cap at 10
   tombstones per fence — drop the oldest beyond that. (Tombstones tell the next reader a module
   was deliberately deleted, not accidentally lost from the docs.)
6. Write the updated file back

### Step S5: Report Changes

Print a summary of what changed:

```
✅ /forge-contextmap sync complete!

Graph rebuilt: [N] nodes, [M] edges
  Pruned: [N] stale node(s) from [M] deleted file(s)   [omit if none]
  Dropped: [N] non-code node(s) (docs)                 [omit if none]
Docs refreshed:
  doc/architecture.md   — [N] sections updated
  doc/domain-model.md   — [N] sections updated
  [...]

Changelog draft (doc/changelog.txt):
  [+N added, -M removed] structural changes drafted — review/edit the auto-draft block
Tombstones: [N] removed modules marked <!-- graphify:removed --> in doc fences

Bloat signal: [N] orphan nodes, [M] duplicate labels, [K] god nodes, [D] dead file(s)
  → run /forge-audit for the confirmed list

User content: untouched (all content outside <!-- graphify:auto --> preserved)
```

If all four bloat counts are zero, print `Bloat signal: clean` instead. These are graph pointers,
not findings — say so if the user reacts to them. `/forge-audit` confirms each against real source
before anything is called bloat.

If S4's curated-prose check fired, repeat the affected files at the end so the warning is not lost
above the summary.
