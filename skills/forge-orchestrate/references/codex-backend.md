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

**Always pass `-m` explicitly.** Without it every dispatch silently inherits whatever `model =` sits
in the user's `~/.codex/config.toml`, which makes the audit's quality a function of an unrelated
setting. This replaces the model-selection paragraph in Phase 4, which applies to Agent-tool workers
only.

| Phase | Model | Why |
|---|---|---|
| **C3 audit** | `gpt-5.6-sol` | The audit exists to catch what a cheap pass won't — "this subtask is architecturally wrong for this codebase", not just "this path does not exist". It is also the cheap phase: read-only, ~2 minutes, and one caught plan flaw saves a whole execute round. |
| **C4 execute** | `gpt-5.6-terra` | Default. Mini-like tier, right for mechanical 1–2 file edits against a clear spec. |
| **C4 execute** | `gpt-5.6-sol` | For a subtask involving multi-file integration or design judgment — the same distinction Phase 4 draws for Claude workers. |

Tier descriptions are the CLI's own: `sol` is *"flagship … for hardest quality-first, coding, and
reasoning workflows"*, `terra` is *"mini-like … for balanced cost, latency, and quality"*.

The audit row is a measured result, not a preference. Same fixture, same schema, same prompt: `terra`
returned criterion-wording nits, while `sol` caught that the plan's central term was undefined
("valid code" with no codes or rates specified), cross-checked the plan against the project
`CLAUDE.md`'s two-decimal rounding rule, and read the source to find that `checkout()` returns a dict
so "returns the discounted amount" names nothing real. Those are the findings that stop a wasted
execute round; the nits are not. Both ran ~2 minutes.

A model id this account cannot use **fails loudly** — verified, not assumed. `codex exec` exits `1`
and prints:

```
warning: Model metadata for `<id>` not found. Defaulting to fallback metadata; this can degrade
performance and cause issues.
ERROR: {"type":"error","status":400,"error":{"type":"invalid_request_error","message":"The '<id>'
model is not supported when using Codex with a ChatGPT account."}}
```

Note the warning line alone is not fatal — it precedes the real error. Branch on the **exit code**,
not on the presence of `warning:`. A non-zero exit here is a stopped run, not a degraded one, so
report it and stop rather than retrying: no `-m` value will fix an account that lacks the model.

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
  -m gpt-5.6-sol \
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
  it as a finding. Missing lint, coverage or SAST tooling is also not a finding: the gate runner
  detects what exists and honestly skips the rest.
  Return execution_order using the subtask numbers exactly as written above, in the order they
  must run.
  Do not write code. Do not edit files.
  ```
  All three carve-outs are load-bearing and were added after fixture runs. Without the first, the
  audit reports every new file the plan creates as a missing path and drowns the real findings.
  Without the second, every round emits a spurious "pytest could not start" finding. Without the
  third, any repo with no linter burns a round on "the lint gate is not verifiable" — which 5a
  already handles by design, so rewriting a subtask over it is pure waste.

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
codex exec -m gpt-5.6-terra -s workspace-write - < graphify-out/.orchestrate_payload_<n>.txt
```

Use `-m gpt-5.6-sol` instead when the subtask is multi-file integration or design judgment (C2).
Add `-C <worktree abs path>` in worktree mode, `--add-dir <path>` per C1.4. Run in the background per
C2.

**The payload is the Phase 3c payload, verbatim, with nothing removed** — that reuse is the point of
this design. Append only these two lines:

```
Do NOT git commit, do NOT create branches, do NOT run git add. The orchestrating session owns git.
List every file you created or modified, one path per line, under a final "FILES CHANGED:" heading.
```

The commit ban is already in 3c, but Codex has its own git tooling and a second explicit statement is
cheap insurance. The file list is neither trusted nor discarded — it is one of two inputs to C5.

---

## C5. Changed-file reconciliation — git is the source of truth

`5b` (diff scope), `5a` gate 5 (secrets fallback) and `5d.2` (`git add [exact files]`, never
`git add -A`) all need the list of files this subtask changed. In the Claude backend that list is a
worker return-contract item. Codex's self-report is not trustworthy enough to stage commits from.

**Snapshot the tree Codex is actually editing.** In worktree mode that is the worktree, not this
session's cwd — the same trap 5b's *"Where the reviewer runs `git diff`"* section exists for. A
bare `git status` there returns the main checkout, the delta comes back empty, and C7's own rule
("delta empty after execute → re-dispatch") then burns all three 5c iterations on a subtask that
worked. Use the `-C` form whenever `-C` was passed to `codex exec`:

Immediately **before** dispatch:

```bash
git -C <same dir codex exec runs in> status --porcelain > graphify-out/.orchestrate_before_<n>.txt
```

After it returns, run the identical command again and take the delta. `--porcelain` is used rather
than `git diff --name-only` because it also reports untracked files, which a new-file subtask
produces. Note it can report a whole **directory** (`?? src/__pycache__/`), not only files — match on
the path prefix, not on exact filenames.

**Two inputs, three outcomes.** Intersect the git delta with Codex's `FILES CHANGED:` list:

| In git delta | Claimed by Codex | Outcome |
|---|---|---|
| yes | yes | **the authoritative list** — this is what 5b diffs and 5d stages |
| yes | no | **report, never stage.** Print it in the Phase 6 report as `unclaimed changes` |
| no | yes | **report.** Usually a sandbox denial or a path outside the writable root |

Neither list alone works. Git alone stages junk: a fixture run showed that a subtask which runs
`pytest` — which Phase 4 mandates — leaves `src/__pycache__/` and `tests/__pycache__/` in the delta,
and `git add`ing those into a feature commit is a real defect, not a cosmetic one. Codex's list alone
is a self-report, which is exactly what must not be trusted to stage a commit.

Before intersecting, **drop every path under `graphify-out/`** from the delta unconditionally. That
directory is not guaranteed to be gitignored, and this backend writes payload, schema, snapshot and
audit files into it.

The `unclaimed changes` row is deliberately loud rather than silently filtered: a file Codex touched
but did not report is either build output (harmless, and the user can gitignore it) or scope creep
(a 5b review failure). Suppressing it would hide the second case.

---

## C6. Retry — Phase 5c

5c's rule is *resume, not respawn*. The Codex equivalent of `SendMessage` is:

```bash
codex exec -s workspace-write -C <same dir as the original dispatch> \
  resume <session id> -m <same model as the original dispatch> \
  - < graphify-out/.orchestrate_retry_<n>.txt
```

**`-s` and `-C` go BEFORE `resume`, not after it.** They are options of `codex exec`, and the
`resume` subcommand does not accept them — it takes `-m`, `-c`, `--output-schema` and `-o`, but has
no sandbox and no working-directory flag of its own. Putting them after the subcommand is not a
degraded run, it is a dead one, before any model call:

```
error: unexpected argument '-s' found
Usage: codex exec resume [OPTIONS] [SESSION_ID] [PROMPT]
```

`-C` must match the original dispatch — in worktree mode a resume without it edits the wrong tree.

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
| Exit 1 with a 400 `invalid_request_error` naming the model | the account cannot use that `-m` id | report and stop — retrying cannot fix it (C2) |
| `error: unexpected argument '-s' found` on a retry | `-s`/`-C` placed after `resume` | move them before the subcommand (C6) |
| Codex reports a file it could not write | path outside the writable root | add `--add-dir <path>` and retry that subtask |
| Empty or unparseable `-o` file after an audit | run died before its final turn | treat as a failed round; it still counts against the 3 |
| `git status` delta is empty after execute | **check the snapshot cwd first** (C5) — in worktree mode a bare `git status` reads the main checkout and always comes back empty. Otherwise Codex answered without editing, or was killed mid-run | fix the `-C`; if the cwd was right, re-dispatch with the goal restated |
| Reconciled delta contains files from an unrelated subtask | `[NO_COMMIT]`, no moving baseline | expected; 5b's out-of-scope note (single-tree `[NO_COMMIT]`) covers it |
| `NEEDS_CONTEXT` in the final message | Codex lacked a doc or file | supply it and re-dispatch via C6, same as a Claude worker |
