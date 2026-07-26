# Graph slice — the per-subtask context payload (Phase 3a)

Read this at PHASE 3 of `SKILL.md`. It builds the graph half of each worker's isolated payload;
the doc half (3b) and the instruction block (3c) stay in `SKILL.md`.

---

### 3a. Graph slice

Write this script to `graphify-out/.orchestrate_slice.py` (once), then run it per subtask. It
loads the graph exactly as graphify/forge-contextmap do (`node_link_graph(..., edges='links')`),
finds nodes matching the subtask's key terms, then expands to their community plus a BFS
neighborhood, and prints the touched source files + node labels + key edges.

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

# 2) community-mates of seeds
seed_comms = {G.nodes[n].get('community') for n in seeds if G.nodes[n].get('community') is not None}
slice_nodes = set(seeds)
for n, d in G.nodes(data=True):
    if d.get('community') in seed_comms:
        slice_nodes.add(n)

# 3) BFS neighborhood (depth 2) from seeds
frontier = set(seeds)
for _ in range(2):
    nxt = set()
    for n in frontier:
        for nb in G.neighbors(n):
            if nb not in slice_nodes:
                nxt.add(nb)
    slice_nodes |= nxt
    frontier = nxt

# 4) emit
files = sorted({G.nodes[n].get('source_file') for n in slice_nodes if G.nodes[n].get('source_file')})
print('FILES'); [print(' ', f) for f in files]
print('NODES'); [print(' ', label_of(n)) for n in sorted(slice_nodes, key=label_of)[:40]]
print('EDGES')
seen = 0
for u, v in G.edges():
    if u in slice_nodes and v in slice_nodes and seen < 30:
        raw = G[u][v]; e = next(iter(raw.values()), {}) if isinstance(G, nx.MultiGraph) else raw
        print(f"  {label_of(u)} --{e.get('relation','')} [{e.get('confidence','')}]--> {label_of(v)}")
        seen += 1
```

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
