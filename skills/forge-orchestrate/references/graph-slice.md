# Graph slice — the per-subtask context payload (Phase 3a)

Read this at PHASE 3 of `SKILL.md`. It builds the graph half of each worker's isolated payload;
the doc half (3b) and the instruction block (3c) stay in `SKILL.md`.

---

### 3a. Graph slice

Write this script to `graphify-out/.orchestrate_slice.py` (once), then run it per subtask. It
loads the graph exactly as graphify/forge-contextmap do (`node_link_graph(..., edges='links')`),
finds nodes matching the subtask's key terms, expands to a depth-2 BFS neighborhood — falling back
to their community only when that BFS comes back nearly empty — and prints the touched source files
(ranked and capped) + node labels + key edges.

```python
import sys, json
from pathlib import Path
import networkx as nx
from networkx.readwrite import json_graph

# forge:shared-block graph-loader
data = json.loads(Path('graphify-out/graph.json').read_text(encoding='utf-8'))
G = json_graph.node_link_graph(data, edges='links')

terms = [t.lower() for t in ' '.join(sys.argv[1:]).split() if len(t) > 3]

def label_of(n):
    return G.nodes[n].get('label', str(n))
# /forge:shared-block graph-loader

# 1) seed nodes: best label-term overlap
scored = []
for n, d in G.nodes(data=True):
    lab = str(d.get('label', '')).lower()
    s = sum(1 for t in terms if t in lab)
    if s:
        scored.append((s, n))
scored.sort(reverse=True)
seeds = [n for _, n in scored[:5]]

if not seeds:
    print('NO_MATCH'); sys.exit(0)

BFS_THIN = 15   # below this, the BFS found almost nothing — fall back to community
FILE_CAP = 12   # both tuned on a real graph; see "Why these numbers" below

# 2) BFS neighborhood (depth 2) from seeds, recording hop distance for ranking
dist = {n: 0 for n in seeds}
frontier = set(seeds)
for hop in (1, 2):
    nxt = set()
    for n in frontier:
        for nb in G.neighbors(n):
            if nb not in dist:
                dist[nb] = hop
                nxt.add(nb)
    frontier = nxt
slice_nodes = set(dist)

# 3) community-mates — ONLY as a fallback when the BFS came back thin
if len(slice_nodes) < BFS_THIN:
    seed_comms = {G.nodes[n].get('community') for n in seeds
                  if G.nodes[n].get('community') is not None}
    for n, d in G.nodes(data=True):
        if d.get('community') in seed_comms and n not in dist:
            dist[n] = 3          # ranks below every BFS hop
            slice_nodes.add(n)

# 4) emit — files ranked seed-hosting first, then by hop distance, then capped
seed_files = {G.nodes[n].get('source_file') for n in seeds}
best = {}
for n in slice_nodes:
    f = G.nodes[n].get('source_file')
    if f:
        best[f] = min(best.get(f, 99), dist[n])
ranked = sorted(best, key=lambda f: (0 if f in seed_files else 1, best[f], f))
print('FILES'); [print(' ', f) for f in ranked[:FILE_CAP]]
if len(ranked) > FILE_CAP:
    print('  ... +%d more (truncated - ask if you need one)' % (len(ranked) - FILE_CAP))
print('NODES'); [print(' ', label_of(n)) for n in sorted(slice_nodes, key=label_of)[:25]]

# edges ranked the same way as files — closest to a seed first, or the 15 slots
# fill up with whatever G.edges() happened to yield first
edges = [(u, v) for u, v in G.edges() if u in slice_nodes and v in slice_nodes]
edges.sort(key=lambda uv: (min(dist[uv[0]], dist[uv[1]]), max(dist[uv[0]], dist[uv[1]])))
print('EDGES')
for u, v in edges[:15]:
    raw = G[u][v]; e = next(iter(raw.values()), {}) if isinstance(G, nx.MultiGraph) else raw
    print("  %s --%s--> %s" % (label_of(u), e.get('relation', ''), label_of(v)))
```

**BFS runs before community, and that ordering is load-bearing.** In the old order the community
sweep ran first and the BFS guard (`if nb not in slice_nodes`) then skipped every community-mate, so
those nodes never entered the frontier and depth-2 never expanded through them — community expansion
*shrank* the slice below what a plain BFS would have found. Measured on a 1457-node graph: for one
subtask, BFS alone reached 298 nodes while the shipped algorithm returned 247.

**Why these numbers** — tuned against a real graph (Flask: 1457 nodes, 2465 edges, 102 communities),
not guessed:

- `FILE_CAP = 12` — across six representative subtask phrasings, the seed-hosting and one-hop files
  (the ones that are actually right) numbered 1–9. Twelve covers all of them with headroom, and cuts
  the worst case from 65 emitted files to 12. Hop-2 is where the noise lives — on this fixture it
  contributed 7–54 files per subtask, mostly `examples/` and tutorial code irrelevant to the task.
- `BFS_THIN = 15` — depth-2 BFS on that graph returned 31–298 nodes, so the fallback never fires on
  a healthy graph. Fifteen marks a genuinely degenerate slice, which is the only case where pulling
  a whole community is better than what BFS found.

Both are plain constants — retune them if your graphs are much smaller or sparser.

Note the `... ` truncation marker and `%`-formatting are deliberate: plain ASCII and no f-strings,
per the portability contract.

Run it with the subtask goal's key words as arguments:
```bash
[PYTHON_CMD] graphify-out/.orchestrate_slice.py "<subtask goal key words>"
```

If it prints `NO_MATCH`, the graph has no node for these terms — fall back to the universal docs
only (3b) and tell the worker the graph had no specific match.

*Optional accelerators (not required):* graphify's own `query` / `path` / `explain` inline
scripts (see graphify SKILL.md), or `[PYTHON_CMD] -m graphify.serve graphify-out/graph.json`
(MCP tools `get_community`, `get_neighbors`, `shortest_path`). All read the same `graph.json`;
the script above is the dependency-free default.
