# Codex backend — audit + execution via `codex exec`

Read this only when `[BACKEND] = codex`. It replaces Phase 4's Agent-tool dispatch with the `codex`
CLI and adds the pre-execution audit (§1b). Everything else in `SKILL.md` — the graph slice, the doc
slice, the 3c payload, the gate runner, the 5b reviewer, the commit policy — is used unchanged.

**The split:** Claude decomposes, audits *in* the findings, and reviews. Codex implements. The 5b
reviewer stays a Claude subagent on purpose — an executor grading its own diff is the weakest
possible critic.

Behaviour below was verified against `codex-cli 0.146.0`.

---

## C1. Preflight — run before Phase 1, stop on failure

Never edit `~/.codex/config.toml` yourself. Print the fix and stop.

1. **`codex` on PATH** — `codex --version`. Missing → stop:
   ```
   /forge-orchestrate codex needs the Codex CLI. Install it, then re-run.
   ```
2. **Codex must read this project's `CLAUDE.md`.** Codex reads `AGENTS.md`, not `CLAUDE.md`, and the
   3c payload deliberately omits the engineering-discipline rules because a Claude subagent
   auto-loads the project's `CLAUDE.md`. Codex only picks it up through a config key. Check for
   `project_doc_fallback_filenames` in `~/.codex/config.toml` (or `$CODEX_HOME/config.toml`).
   Missing → stop with this exact remediation:
   ```
   Codex will not read this project's CLAUDE.md — its engineering rules would be silently
   dropped from every dispatch. Add this line to ~/.codex/config.toml, then re-run:

       project_doc_fallback_filenames = ["CLAUDE.md"]
   ```
3. **`AGENTS.md` shadow check.** The key is a *fallback*: when a project has both files, `AGENTS.md`
   wins and `CLAUDE.md` is never read. If the repo root has an `AGENTS.md`, stop:
   ```
   This repo has both AGENTS.md and CLAUDE.md. Codex reads AGENTS.md and ignores CLAUDE.md, so
   the ContextForge rules would not reach it. Merge CLAUDE.md's rules into AGENTS.md, or remove
   AGENTS.md, then re-run.
   ```
4. **Graph reachability.** If `graphify-out/` or `doc/` sit outside the directory Codex will run in,
   every invocation below needs `--add-dir <path>`. Normally they are both in the repo root and
   nothing is needed.

No trust check is required: `codex exec` runs with approvals disabled and does not prompt in an
untrusted directory.

---

## C2. Invocation rules — apply to every `codex exec` below

**Prompt goes over stdin, never as a shell argument.** Payloads contain backticks, double quotes,
newlines and `$`-adjacent text; inline shell quoting truncates them silently. Write the payload to a
file and pipe it:

```bash
codex exec [flags] - < graphify-out/.orchestrate_payload_<n>.txt
```

**Dispatch in the background.** The Bash tool's timeout defaults to 2 minutes and caps at 10. A
trivial one-function edit measured **56s**; a real subtask exceeds both. A killed subprocess leaves a
half-written working tree that C5 would then hand to `git add` — worse than a clean failure. So run
every execute dispatch with `run_in_background: true`, and set an explicit `timeout` on audit
dispatches rather than taking the 2-minute default.

**Never `--ephemeral`** — it skips writing the session file, which kills the resume path in C6.
**Never `--dangerously-bypass-approvals-and-sandbox`** — `-s` is the whole safety story here.

**Capture the session id.** `codex exec` prints `session id: <uuid>` on stdout before the turn
starts. Grab it and store it in the subtask's existing `agent_id` slot — C6 resumes with it.

**Worktree mode** — pass `-C <worktree abs path>` so Codex's working root is that checkout.

---

## C3. Audit — Phase 1b

### The schema

Write this to `graphify-out/.orchestrate_audit_schema.json` once, before the first round. Phase 6
deletes it.

```json
{
  "type": "object",
  "properties": {
    "verdict": { "type": "string", "enum": ["verified", "flawed"] },
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "severity": { "type": "string", "enum": ["high", "medium", "low"] },
          "subtask": { "type": "string" },
          "location": { "type": "string" },
          "problem": { "type": "string" },
          "why_it_fails": { "type": "string" }
        },
        "required": ["severity", "subtask", "location", "problem", "why_it_fails"],
        "additionalProperties": false
      }
    },
    "execution_order": { "type": "array", "items": { "type": "string" } }
  },
  "required": ["verdict", "findings", "execution_order"],
  "additionalProperties": false
}
```

Structured output is what makes the loop branchable — a magic `SPEC_VERIFIED` token breaks on any
preamble. The `-o` file holds the schema'd object alone, with no prose around it, so `jq -e
'.verdict'` on it is the read.

### The call

```bash
codex exec \
  -s read-only \
  --output-schema graphify-out/.orchestrate_audit_schema.json \
  -o graphify-out/.orchestrate_audit_round<N>.json \
  - < graphify-out/.orchestrate_payload_audit<N>.txt
```

### The payload

- The full Phase 1 decomposition: every subtask's goal, success criterion, deps, gate set.
- The path `graphify-out/graph.json` and the Phase 3a `FILES` list for each subtask — **paths, not
  pasted text**, Codex has its own read tools.
- `[TASK]`, plus `[TASK_CRITERIA]` when it exists.
- These questions, verbatim:
  ```
  Audit this decomposition BEFORE any code is written. For each subtask check:
  - Does every file path a subtask READS, IMPORTS or MODIFIES actually exist in the repo? A path
    a subtask explicitly CREATES is expected not to exist yet — never flag those.
  - Are there missing imports, conflicting dependencies, or omitted migrations?
  - Is a subtask's success criterion actually verifiable as written?
  - Does the stated dependency order work, or does a subtask need something a later one builds?
  - Is any subtask architecturally wrong for this codebase?
  You are in a read-only sandbox and cannot run the test suite. That is expected — do not report
  it as a finding.
  Return execution_order using the subtask numbers exactly as written above, in the order they
  must run.
  Do not write code. Do not edit files.
  ```
  Both carve-outs are load-bearing and were added after a fixture run: without the first, the audit
  reports every new file the plan creates as a missing path and drowns the real findings; without the
  second, every round emits a spurious "pytest could not start" finding.

### The loop — bounded at 3 rounds

- `verdict == "verified"` → go to Phase 4.
- `verdict == "flawed"` → Claude rewrites **only the subtasks the findings name**, then re-audits.
  Do not regenerate the whole decomposition; unaffected subtasks stay byte-identical.
- **After the 3rd `flawed`, stop.** Print every round's verdict and the full findings list, and hand
  control back to the user. There is no 4th `codex exec`. An unbounded audit loop burns both quotas.

**Ordering conflict.** Phase 1 already emits per-subtask `deps`. If `execution_order` disagrees with
them, that disagreement is itself a finding Claude reconciles **in the decomposition** before the
next round — never a silent override at dispatch time. Two sources of truth for sequencing is a bug.
Phase 4 follows the reconciled `deps`; `execution_order` from the **final** round is the tiebreaker
for subtasks the deps leave unordered.

---

## C4. Execute — Phase 4

```bash
codex exec -s workspace-write - < graphify-out/.orchestrate_payload_<n>.txt
```

Add `-C <worktree abs path>` in worktree mode, `--add-dir <path>` per C1.4. Run in the background per
C2.

**The payload is the Phase 3c payload, verbatim, with nothing removed** — that reuse is the point of
this design. Append only these two lines:

```
Do NOT git commit, do NOT create branches, do NOT run git add. The orchestrating session owns git.
List every file you created or modified, one path per line, under a final "FILES CHANGED:" heading.
```

The commit ban is already in 3c, but Codex has its own git tooling and a second explicit statement is
cheap insurance. The file list is cross-checked, not trusted — see C5.

---

## C5. Changed-file reconciliation — git is the source of truth

`5b` (diff scope), `5a` gate 5 (secrets fallback) and `5d.2` (`git add [exact files]`, never
`git add -A`) all need the list of files this subtask changed. In the Claude backend that list is a
worker return-contract item. Codex's self-report is not trustworthy enough to stage commits from.

Immediately **before** dispatch:

```bash
git status --porcelain > graphify-out/.orchestrate_before_<n>.txt
```

After it returns, run `git status --porcelain` again and take the delta. That delta is the
authoritative list. `--porcelain` is used rather than `git diff --name-only` because it also reports
untracked files, which a new-file subtask produces.

Two mandatory filters and one check:

- **Drop every path under `graphify-out/`.** That directory is not guaranteed to be gitignored, and
  this backend writes payload, schema, snapshot and audit files into it. Unfiltered, they would be
  staged as if Codex authored them.
- **Drop nothing else.** A file Codex touched that the subtask did not ask for is a 5b review
  failure, not something to hide.
- **Cross-check Codex's `FILES CHANGED:` list against the delta.** If Codex names a path git does not
  show, surface the mismatch in the Phase 6 report rather than swallowing it — it usually means a
  sandbox denial or an edit to a path outside the writable root.

---

## C6. Retry — Phase 5c

5c's rule is *resume, not respawn*. The Codex equivalent of `SendMessage` is:

```bash
codex exec resume <session id> -s workspace-write - < graphify-out/.orchestrate_retry_<n>.txt
```

The retry payload is the failure log alone — the gate output, the reviewer's `FAILURES` lines, or the
delete-list. Not the original payload: the session still holds every file it read and every edit it
made.

Resume prints a new session id; record it, because the next retry resumes from that one.

If the session id was never captured or resume errors, fall back to a fresh `codex exec` with the
**full** payload plus the failure log — the same fallback 5c already documents for a dead agent.
Never use `resume --last` under parallel dispatch; it races between concurrent sessions.

The 3-iteration cap in 5c applies unchanged, and the secrets hard-block still bypasses the loop
entirely.

---

## C7. Failure modes

| Symptom | Cause | Response |
|---|---|---|
| Codex reports a file it could not write | path outside the writable root | add `--add-dir <path>` and retry that subtask |
| Empty or unparseable `-o` file after an audit | run died before its final turn | treat as a failed round; it still counts against the 3 |
| `git status` delta is empty after execute | Codex answered without editing, or was killed mid-run | do not mark the subtask passed — re-dispatch with the goal restated |
| Reconciled delta contains files from an unrelated subtask | `[NO_COMMIT]`, no moving baseline | expected; 5b's out-of-scope note (single-tree `[NO_COMMIT]`) covers it |
| `NEEDS_CONTEXT` in the final message | Codex lacked a doc or file | supply it and re-dispatch via C6, same as a Claude worker |
