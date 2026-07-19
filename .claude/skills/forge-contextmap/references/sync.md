## SYNC MODE

*Trigger: `graphify-out/graph.json` exists, or `sync` argument*

### Step S1: Python Check

Same as Step E1 in `references/existing-project.md`. Detect `[PYTHON_CMD]`.

### Step S2: Rebuild Graph

Run incremental graph update:
```bash
[PYTHON_CMD] -m graphify . --update
```

If `--update` flag is not supported by the installed version, fall back to:
```bash
[PYTHON_CMD] -m graphify .
```

Wait for completion.

### Step S3: Parse Updated Graph

Re-read `graphify-out/graph.json`. Extract the same fields as Step E4 in `references/existing-project.md`.

### Step S3.5: Diff Graph and Draft Changelog

Auto-draft structural changelog entries from what actually changed, so the log captures real mutations instead of vague summaries. Use the Read tool on `graphify-out/graph.prev.json` (the baseline saved at the end of the last sync):

1. **If `graphify-out/graph.prev.json` does not exist** (first sync): skip the diff — you'll create the baseline in step 4 below. Note "changelog baseline created — no diff yet."
2. **If it exists**: build a node-identity key for each node as `label + "@" + source_file`, then compute:
   - `added` = nodes in current `graph.json` but not in `graph.prev.json`
   - `removed` = nodes in `graph.prev.json` but not in current `graph.json`
   - For file-level context, also run (best-effort — ignore failure): `git diff --name-only HEAD~1 HEAD 2>/dev/null`
3. **Append a draft block** to `doc/changelog.txt`. This file is user-owned, so mark the block clearly as an editable draft. Use the existing `Date | Change | Description` line style:
   ```
   [TODAY'S DATE] | Auto-draft (review/edit) | [Added] <label> (<source_file>) ; [Removed] <label> (<source_file>)
   ```
   List the most significant added/removed nodes (cap ~10 each to stay readable). If nothing structural changed, write one line noting "no structural changes detected."
4. **Update the baseline** for next time — use the Bash tool:
   ```bash
   cp graphify-out/graph.json graphify-out/graph.prev.json
   ```

Keep the `removed` list in memory — Step S4 uses it for tombstones.

### Step S4: Fence-Aware Merge

For each doc file that has `<!-- graphify:auto start:... -->` markers (v2 set:
`CLAUDE.md` claude-summary, `architecture.md`, `domain-model.md`, `api-contract.md`,
`solution-structure.md`, `coding-standard.md`, `security.md`, `app-flow.md`, `design-brief.md`,
`backend-schema.md` — whichever exist):

1. Read the entire file
2. Find all fence pairs: `<!-- graphify:auto start:KEY -->` ... `<!-- graphify:auto end:KEY -->`
3. For each fence pair:
   - Generate new content based on the updated graph data
   - Replace ONLY the content between the start and end markers
   - Keep the markers themselves intact
   - Keep all content outside the markers exactly as-is
4. **Tombstones**: if a node from the S3.5 `removed` list appeared in this fence's PREVIOUS
   content, append at the end of the new fence content:
   ```
   <!-- graphify:removed: <label> (last seen: YYYY-MM-DD) -->
   ```
   Also carry over any `graphify:removed` lines already present in the old fence content. Cap at 10
   tombstones per fence — drop the oldest beyond that. (Tombstones tell the next reader a module
   was deliberately deleted, not accidentally lost from the docs.)
5. Write the updated file back

### Step S5: Report Changes

Print a summary of what changed:

```
✅ /forge-contextmap sync complete!

Graph rebuilt: [N] nodes, [M] edges
Docs refreshed:
  doc/architecture.md   — [N] sections updated
  doc/domain-model.md   — [N] sections updated
  [...]

Changelog draft (doc/changelog.txt):
  [+N added, -M removed] structural changes drafted — review/edit the auto-draft block
Tombstones: [N] removed modules marked <!-- graphify:removed --> in doc fences

User content: untouched (all content outside <!-- graphify:auto --> preserved)
```
