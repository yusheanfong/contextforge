# Architecture diagram — the shared render step

Read this when a mode calls for a diagram. Two call sites use it and they differ only in when they
run:

| Called from | When | Runs D0? |
|---|---|---|
| [`references/existing-project.md`](existing-project.md) Step E4.6 | first run on an existing project, after the graph is parsed | no — nothing to compare against |
| [`references/sync.md`](sync.md) Step S4.5 | every sync, after the fences are merged | yes |

The output is one file, `doc/diagram/architecture.html` — a self-contained interactive page the
user opens in a browser. No server, no assets directory, no build.

> **Portability contract.** Same as `sync.md`'s, with one addition: this step may run
> `node [ARCHIFY_DIR]/bin/archify.mjs …`. Still **no heredocs, no `cp`/`mv`/`rm`, no `mkdir -p`/
> `chmod`, no `2>/dev/null`, no `||` chaining.** Multi-line Python goes into a file written with the
> **Write tool** and is run as `[PYTHON_CMD] <script>.py`.

**Nothing here may fail the caller — no exceptions.** Every step below ends in a recorded status
line and a return. That includes the steps that run a script of your own: if the fingerprint script
or the spec builder raises, catch it, record `Diagram: skipped — <what failed>`, and return. This is
the same best-effort posture `sync.md` Step S3.6 already takes. A user syncing their docs must never
lose the sync because a diagram renderer had a bad day.

---

## D1. Preflight — Node first, then the archify skill

**Check Node before looking for archify.** With no Node, every probe below fails for the wrong
reason and you would report a missing skill when the real problem is a missing interpreter.

```bash
node --version
```

Absent, or below v18: record `Diagram: skipped — Node >= 18 not found (found <version>)` and return.
Do not attempt an install; that is the user's environment.

### Resolving `[ARCHIFY_DIR]`

**Archify is a separate skill the user installs themselves.** ContextForge does not ship it. Three
directories are probed, in this order, and the first whose `bin/archify.mjs` exists wins:

```
.claude/skills/archify              (project-level install, relative to the repo root)
<home>/.claude/skills/archify       (personal install)
<home>/.agents/skills/archify       (the installer's real target — the personal path is often a symlink to it)
```

**Resolve `<home>` with `[PYTHON_CMD]`, never by writing `~` into a shell command.** `cmd` and
PowerShell do not expand `~`, so a command containing it breaks the portability contract above.
`[PYTHON_CMD]` is already resolved before both call sites — `sync.md` Step S1 runs before S4.5,
`existing-project.md` Step E2.6 runs before E4.6 — so this costs nothing.

Write this with the **Write tool** to `graphify-out/.forge_archify_probe.py`, run it, read its
single line of output, then delete it.

```python
"""Print the installed archify skill directory, or nothing at all. Stdlib only."""
from pathlib import Path

home = Path.home()
candidates = [
    Path('.claude') / 'skills' / 'archify',
    home / '.claude' / 'skills' / 'archify',
    home / '.agents' / 'skills' / 'archify',
]

for d in candidates:
    if (d / 'bin' / 'archify.mjs').is_file():
        print('%s' % d)
        break
```

```bash
[PYTHON_CMD] graphify-out/.forge_archify_probe.py
```

`[ARCHIFY_DIR]` is that line. It is a **directory** — every command below appends
`/bin/archify.mjs` to it, so never store the `.mjs` path itself. Empty output, or a non-zero exit:
record `Diagram: skipped — archify skill not installed` and return.

**An Archify installed as a plugin is deliberately not probed for.** Its path
(`~/.claude/plugins/cache/<marketplace>/<plugin>/<sha>/…`) is keyed by commit and several SHAs
coexist — three on the machine this was written on. Dead ones are marked `.orphaned_at`, but relying
on an undocumented internal marker to pick the live one is worse than not guessing: a wrong pick
silently runs a stale bundle. Three misses is a skip, not a wider search.

```bash
node [ARCHIFY_DIR]/bin/archify.mjs doctor
```

Non-zero exit: record `Diagram: skipped — archify doctor failed`, include its last line, and return.

**Run only `doctor`, `validate` and `deliver`. Never any other subcommand — this is a flat
prohibition, not a default to be overridden.** The reason is simple: *a documentation sync makes no
network request and binds no port.* Specifically, never `check-update` — the installed copy ships
`scripts/check-update.mjs` and `[ARCHIFY_DIR]/SKILL.md` tells an agent to run it, so ignore that
instruction here — never `brands capture` (outbound request), and never `preview` (binds a local
server).

## D1.5. Keep the artifact out of git by default

Preflight passed, so a render is about to be attempted. Do this before it happens, not after —
the sync call site can take D0's freshness skip below and return without ever reaching D4, and a
repo scaffolded before this step existed would otherwise never get the line.

Read `.gitignore` at the repo root. If nothing in it already covers `doc/diagram/`, append these
two lines with the **Write tool** (create the file if it is absent):

```
# ContextForge architecture diagram — regenerated on every /forge-contextmap sync
doc/diagram/
```

`doc/diagram/`, `doc/diagram` and a blanket `doc/` all already cover it — recognise those and do
not add a duplicate. If there is no `.git` directory, skip this entirely and render anyway.

**The default runs this way because the file is generated output**, rewritten in full whenever the
graph changes, ~700 KB a time. A user who wants it versioned deletes the line; a user who does not
would otherwise have to notice a 700 KB blob in their first `git status` after every sync.

**This step can never fail the render.** If `.gitignore` cannot be read or written, carry on to the
render and say nothing about it — an unignored diagram is a smaller problem than no diagram.

## D0. Freshness skip — sync call site only

*Skip this section entirely at the existing-project call site: there is no previous graph and no
existing artifact.*

`sync.md` Step S3.5 already diffs the graph, but **its diff is not sufficient here.** S3.5 keys nodes
on `label@source_file` only, so a changed relationship or a re-clustered community produces
`added=0 removed=0` while the diagram they would draw is different. Reusing it would leave a stale
picture with no signal. Measured: changing one edge's `relation` leaves S3.5 reporting
`added=0 removed=0`, while the fingerprint below changes.

Write this with the **Write tool** to `graphify-out/.forge_diagram_fp.py`, run it, read its output,
then delete it. **If it exits non-zero, record `Diagram: skipped — fingerprint failed` and return**
— do not fall through to a render on an unknown state.

```python
"""Fingerprint the graph fields the diagram is built from. Stdlib + networkx."""
import hashlib
import json
from pathlib import Path
import networkx as nx
from networkx.readwrite import json_graph

data = json.loads(Path('graphify-out/graph.json').read_text(encoding='utf-8'))
G = json_graph.node_link_graph(data, edges='links')

# Bump when the transform below changes shape, so an unchanged graph still
# re-renders after a ContextForge upgrade.
TRANSFORM_VERSION = '2'


def relation_of(u, v):
    # MultiGraph yields {key: attrs}; plain Graph yields attrs. Same guard the
    # orchestrate slice script uses -- without it a MultiGraph silently
    # fingerprints every relation as ''.
    raw = G[u][v]
    e = next(iter(raw.values()), {}) if isinstance(G, nx.MultiGraph) else raw
    return (e or {}).get('relation', '')


nodes = sorted(
    '%s|%s|%s' % (
        d.get('label'),
        (d.get('source_file') or '').replace(chr(92), '/'),
        d.get('community'),
    )
    for _, d in G.nodes(data=True)
)
edges = sorted(
    '%s|%s|%s' % (
        (G.nodes[u].get('source_file') or '').replace(chr(92), '/'),
        (G.nodes[v].get('source_file') or '').replace(chr(92), '/'),
        relation_of(u, v),
    )
    for u, v in G.edges()
)
payload = TRANSFORM_VERSION + '\n' + '\n'.join(nodes) + '\n--\n' + '\n'.join(edges)
print(hashlib.sha256(payload.encode('utf-8')).hexdigest())
```

Compare against `doc/diagram/.architecture.fingerprint`:

- missing, or the hash differs, or `doc/diagram/architecture.html` does not exist → render.
- hash matches **and** the artifact exists → record `Diagram: unchanged` and return. Nothing is
  rewritten, so a sync that changed no structure leaves the file and its git status untouched.

Write the new hash only after D4 reports a delivered artifact. Writing it earlier would mark a
failed render as current and suppress the retry on the next sync.

## D2. Build the base spec — components only, and it must always render

Read `[ARCHIFY_DIR]/SKILL.md` before authoring. It is the authority on Archify's invariants
and repair order, and it ships with whichever version the user installed. Do not paraphrase its
rules into this file.

**Why components-only first.** Archify rejects a spec whose connection routing is unclean —
`clean-flow/edge-through-node` fires when a connection crosses an unrelated component box, and a
grid layout produces exactly that whenever two connected components are not neighbours. There is no
fixed formula that assigns sides correctly for an arbitrary graph; a blanket rule was measured
failing at every density from 8 connections upward, and a dense spec crashes the renderer outright
(`internal/unclassified`, "Renderer failed before emitting a structured diagnostic"). So the base
spec carries **no connections**, which is proven to render at 3, 7 and 12 components (9/9 checks,
0 errors, 0 warnings). D3 adds connections on top and can always fall back to this.

Schema facts, already established — do not re-derive them. Every hard number in D2, D3 and D4 was
verified against **archify 2.16.0**; a different installed version is carried by D3's and D4's
existing failure paths.

- `"schema_version": 1` — **the integer 1**, not `"1"` and not `1.0`. It is a const; anything else
  fails with `/schema_version must be equal to constant`.
- Required top-level: `schema_version`, `diagram_type`, `meta`, `components`.
- Required per component: `id`, `type`, `label`. `pos` and `size` are **optional** — never author
  coordinates.
- `type` is an enum: `frontend`, `backend`, `database`, `cloud`, `security`, `messagebus`,
  `external`. There is no "unknown" member, so the transform must map every component to one.
- `id` must match `^[a-zA-Z][a-zA-Z0-9_-]*$`. A path-derived id breaks this for any dotfile or
  digit-leading name — `.claude-plugin/marketplace.json` naively becomes
  `-claude-plugin-marketplace-json`, which is rejected.
- `sublabel` is a **string**. A bare integer is a schema error.
- `layout` accepts `{"mode": "grid", "cols": N}`; components carry `row` and `col`, both
  **0-indexed**. A `col` equal to `cols` is a hard error.

The transform — deterministic, so the same graph yields a byte-identical spec:

1. **Roll up to one component per `source_file`.** Sort by node count descending, then by path
   ascending. Take at most **12**. Every tie breaks on the path string, never on dict order.
2. **`id`**: lowercase the path, replace every character outside `[a-z0-9]` with `-`, collapse runs
   of `-`, strip leading and trailing `-`. If the result is empty or does not start with a letter,
   prefix `n-`. If two paths collide, append `-2`, `-3`, … in the sorted order from step 1.
3. **`label`** is the file stem. **`sublabel`** is `"<n> nodes"` — a string, with the count
   interpolated.
4. **`type`**, by first match against the lowercased path, so it is total and deterministic:
   `test`/`spec` → `external`; `model`/`schema`/`migration`/`entity` → `database`;
   `auth`/`security`/`permission` → `security`; `queue`/`event`/`broker` → `messagebus`;
   `ui`/`component`/`view`/`page`/`css` → `frontend`; `deploy`/`docker`/`infra`/`terraform` →
   `cloud`; anything else → `backend`.
5. **`layout`** is `{"mode": "grid", "cols": 3}`; component *i* in the sorted order gets
   `row = i / 3`, `col = i % 3`, both 0-indexed.
6. **`connections`** is `[]`. **No `boundaries`.** A boundary needs a `kind` and a defensible
   grouping; the graph gives neither, and inventing one is worse than omitting it.

Set `meta.title` from the repository directory name and `meta.quality_profile` to `"showcase"`.

**Say what the diagram actually shows.** Where the graph's edges are heading containment rather than
code references — a documentation corpus — this is *heading topology*, not runtime architecture. Say
so in `meta.title` and never manufacture connections to make it look connected. A repo whose graph
has no cross-file edges legitimately renders as unconnected components. That is information.

Write it with the **Write tool** to `graphify-out/.forge_diagram_spec.json`. If building it raises,
record `Diagram: skipped — spec build failed` and return.

## D3. Add connections, bounded, with the base spec as the floor

Skip this step entirely when the graph has no cross-file edges — the base spec is already the
answer.

Otherwise derive candidate connections from graph edges whose endpoints live in **different**
source files that both survived the cap. Deduplicate on `(from, to)`. Sort by `from` then `to`. Give
each a deterministic `id` of `"<from>-to-<to>"`. Take at most **12**; a diagram is a map, not an
inventory, and density is what crashes the renderer.

Then iterate, following `[ARCHIFY_DIR]/SKILL.md`'s repair order:

```bash
node [ARCHIFY_DIR]/bin/archify.mjs validate architecture graphify-out/.forge_diagram_spec.json --quality showcase --json
```

**Read the result from the right fields, because there are two different shapes.**

| Outcome | How to detect it |
|---|---|
| clean | `ok: true`, `composition.status: "pass"`, `composition.summary.errors` and `.warnings` both 0 |
| schema/composition problem | `ok: false` **with** a `composition` object |
| render-stage failure | `ok: false`, `stage: "render"`, and **no `composition` key at all** — read `diagnostics[]` and `error` instead |

Never look for `checksPassed` here; that field belongs to `deliver`'s output and its absence in
`validate` reads as a pass.

Repair only the subject a diagnostic names — `clean-flow/edge-through-node` and
`clean-flow/endpoint-side-direction` are fixed by setting that connection's `fromSide`/`toSide`, or
by dropping that connection. **Bound the loop by the diagnostic count**, which exists in both failure
shapes, rather than by error/warning counts, which do not exist on a render-stage failure.

**Stop after 2 rounds that do not reduce the diagnostic count.** Then discard the connections, keep
the D2 base spec, and carry the note `(connections omitted — routing unresolved)` into the status
line. Never lower `quality_profile` to make a failure disappear; it does not help anyway — the
clean-flow checks are not quality-gated, and `standard` fails identically.

## D4. Deliver

```bash
node [ARCHIFY_DIR]/bin/archify.mjs deliver architecture graphify-out/.forge_diagram_spec.json doc/diagram/architecture.html --quality showcase --json
```

`deliver` **creates missing parent directories itself** — verified two levels deep. No `mkdir` is
needed, which is what keeps this inside the portability contract.

On success it reports `validation.checksPassed` / `checkCount` (9/9 for showcase), `errors`,
`warnings`, `artifact.bytes` and a SHA-256. Record
`Diagram: doc/diagram/architecture.html (9/9 checks, N bytes)` and write D0's fingerprint.

**A non-zero exit is a delivery failure and needs its own state.** Never infer success from the
output file existing — a previous run's artifact sits at that path and would read as a pass. On
failure: leave the previous artifact untouched, do **not** write the fingerprint, and record
`Diagram: deliver failed — previous artifact retained (last delivered <date>)`, or
`Diagram: deliver failed — no artifact` when there was none.

If delivery fails on a spec that still carried connections, retry once with the D2 base spec before
giving up. A components-only diagram is worth more than none.

**Never run `visual-check` here.** It writes six sidecar files totalling ~600 KB beside the output
(PNGs at two sizes in both themes, a contact sheet, a receipt). That belongs in a fixture run, not
in every user's repository on every sync.

## D5. Report

Return exactly one status line to the caller, which prints it in its own summary:

```
Diagram: doc/diagram/architecture.html (9/9 checks, 705,746 bytes)
Diagram: doc/diagram/architecture.html (9/9 checks, 705,746 bytes) (connections omitted — routing unresolved)
Diagram: unchanged
Diagram: skipped — Node >= 18 not found (found v16.20.0)
Diagram: skipped — archify skill not installed
Diagram: skipped — archify doctor failed: <last line>
Diagram: skipped — fingerprint failed
Diagram: skipped — spec build failed
Diagram: deliver failed — previous artifact retained (last delivered 2026-08-30)
Diagram: deliver failed — no artifact
```

Delete `graphify-out/.forge_diagram_spec.json`, `graphify-out/.forge_diagram_fp.py` and
`graphify-out/.forge_archify_probe.py` on every path out, including every failure path.
