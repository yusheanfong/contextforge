# Codex backend — planning council + execution via `codex exec`

Read this only when `[BACKEND] = codex`. It replaces Phase 4's Agent-tool dispatch with the `codex`
CLI and replaces Phase 1 with the planning council (§C3). Everything else in `SKILL.md` — the graph
slice, the doc slice, the 3c payload, the gate runner, the 5b reviewer, the commit policy — is used
unchanged.

**The split:** Codex proposes a plan independently, Claude synthesizes, Codex critiques the
synthesis, Claude decides. Then Codex implements and Claude reviews. Claude is the coordinator,
synthesizer, reviewer and final decision-maker throughout; Codex supplies a second independent
judgment at planning time and does the implementation. The 5b reviewer stays a Claude subagent on
purpose — an executor grading its own diff is the weakest possible critic.

Behaviour below was verified against `codex-cli 0.146.0`.

---

## C1. Preflight — run before the council

Only C1.1 stops the run: with no `codex` binary there is no backend. The rest print and continue,
because C4 names `CLAUDE.md` in every payload and their failure modes degrade the run rather than
break it. **Never edit `~/.codex/config.toml` yourself** — print the fix and let the operator decide.
A C1.1 stop needs no cleanup: it fires before the run has created a single artifact.

1. **`codex` on PATH** — `codex --version`. Missing → stop:
   ```
   /forge-orchestrate codex needs the Codex CLI. Install it, then re-run.
   ```
2. **Codex must read this project's `CLAUDE.md`.** Codex auto-loads `AGENTS.md`, not `CLAUDE.md`,
   and the 3c payload deliberately omits the engineering-discipline rules because a Claude subagent
   auto-loads the project's `CLAUDE.md` for free. Two things get it to Codex, and they are not
   equivalent: the config key below makes it **auto-load** (in a repo with no `AGENTS.md`), while
   C4's prepended line makes every dispatch **read it on instruction**. C4 always fires, so this is
   a warning, not a stop. Check for `project_doc_fallback_filenames` in `~/.codex/config.toml` (or
   `$CODEX_HOME/config.toml`). Missing → print and continue:
   ```
   Note: ~/.codex/config.toml has no project_doc_fallback_filenames, so Codex will not auto-load
   this project's CLAUDE.md. Every dispatch names the file explicitly, so its rules still reach
   Codex — but auto-loading is the stronger guarantee. To get it, add this line and re-run:

       project_doc_fallback_filenames = ["CLAUDE.md"]
   ```
3. **`AGENTS.md` shadow check — compare the two files, do not merely look for one.** The key is a
   *fallback*, and resolution is **first-match-wins**: with both files present only `AGENTS.md`
   reaches the model, and with `AGENTS.md` absent `CLAUDE.md` does. Verified against 0.146.0 with
   `codex debug prompt-input`, which renders the model-visible prompt as JSON without a model call —
   use it if you ever need to re-check this.

   No `AGENTS.md` in the repo root → nothing to do. If there is one, diff it:

   ```bash
   git diff --no-index --ignore-all-space -- CLAUDE.md AGENTS.md
   ```

   Empty → the shadowing is harmless; continue silently. Non-empty → print the drifted section
   headings and **continue**:

   ```
   Note: AGENTS.md shadows CLAUDE.md for Codex, and the two have drifted — Codex auto-loads only
   AGENTS.md. Drifted sections: [headings]. Every dispatch below names CLAUDE.md explicitly (C4),
   so its rules still reach Codex; but where the two disagree, AGENTS.md is the one Codex reads
   first. Reconcile them if that matters for this task.
   ```

   This is a warning rather than a stop because C4 delivers `CLAUDE.md`'s *content* by naming its
   path in every payload. What C4 cannot do is evict `AGENTS.md` from the auto-loaded context — so
   the diff is the only signal that two rule sets are in play, which is exactly why the check tests
   divergence and not existence. An existence check fires on identical files, gets overridden once,
   and is dead from then on.
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

**Dispatch in the background — every call, planning included.** The Bash tool's timeout defaults to
2 minutes and caps at 10. A trivial one-function edit measured **56s**; a real subtask exceeds both.
A killed execute subprocess leaves a half-written working tree that C5 would then hand to `git add`
— worse than a clean failure.

Planning calls are not reliably the short ones. A `sol`/`high` council-class call over this repo
**exceeded the 10-minute cap and was killed**; the identical call run detached finished normally. A
killed planning call is a burnt budget slot (C3 counts it), so run every dispatch — execute *and*
council — with `run_in_background: true` and read the result file when it lands. Do not give a
planning call a foreground timeout; there is no timeout value that is both safe and under the cap.

**Never `--ephemeral`** — it skips writing the session file, which kills the resume path in C6 and
council round 3. **Never `--dangerously-bypass-approvals-and-sandbox`** — `-s` is the whole safety
story here.

**Capture the session id — and note that background dispatch changes where you read it.** `codex
exec` prints `session id: <uuid>` on stdout before the turn starts. Detached, that stdout does not
come back to you, so **redirect it to a log file and read the id out of the log**:

```bash
codex exec [flags] - < <payload> > <log> 2>&1
```

Without this the id is never captured, and every path that depends on it silently degrades: C6 falls
back to a fresh dispatch with the full payload, and council round 3 falls back to a fresh session
that has forgotten its own findings. Both fallbacks work, so nothing errors — which is exactly why
this is easy to miss.

An execute dispatch stores the id in the subtask's existing `agent_id` slot for C6; council round 2
stores it in `council.round2_session_id` for round 3. Keep the two apart — they are different
sessions with different sandboxes.

**Never detect completion by grepping the log for text that appears in the payload.** `codex exec`
echoes the whole prompt to stdout before the turn starts, so a `FILES CHANGED` grep matches the
instruction that *asked* for the list, not the answer, and reports a finished run seconds after
dispatch. Any sentinel drawn from the payload self-matches this way. Wait on the background
dispatch's own exit; if you must poll the log, match the run trailer `^tokens used`, which only the
finished turn writes. For a council round the `-o` file appearing is the same signal.

**Worktree mode** — pass `-C <worktree abs path>` so Codex's working root is that checkout.

**Always pass `-m` explicitly, and on planning calls `-c 'model_reasoning_effort="high"'` too.**
Without them a dispatch silently inherits whatever `model =` and `model_reasoning_effort =` sit in
the user's `~/.codex/config.toml`, which makes the council's quality a function of an unrelated
setting. The two are one rule: an unset reasoning effort degrades a `sol` call as surely as the wrong
`-m` does, and neither failure is visible in the output. This replaces the model-selection paragraph
in Phase 4, which applies to Agent-tool workers only.

| Phase | Model | Why |
|---|---|---|
| **C3 council** | `gpt-5.6-sol`, `model_reasoning_effort="high"` | Every planning call — the independent proposal and both critiques. Planning is where a cheap pass fails silently: it returns criterion-wording nits instead of "this subtask is architecturally wrong for this codebase". It is also the cheap phase — read-only, ~2 minutes a call, and one caught plan flaw saves a whole execute round. |
| **C4 execute** | `gpt-5.6-terra` | Default. Mini-like tier, right for mechanical 1–2 file edits against a clear spec. |
| **C4 execute** | `gpt-5.6-sol` | For a subtask involving multi-file integration or design judgment — the same distinction Phase 4 draws for Claude workers. |

Execution keeps the explicit split above. `sol`/`high` is the planning tier, not a blanket default —
do not reach for it on a bounded mechanical subtask.

Tier descriptions are the CLI's own: `sol` is *"flagship … for hardest quality-first, coding, and
reasoning workflows"*, `terra` is *"mini-like … for balanced cost, latency, and quality"*.

The council row's model choice is a measured result, not a preference. Same fixture, same schema,
same prompt, on the audit this council's critique round grew out of: `terra`
returned criterion-wording nits, while `sol` caught that the plan's central term was undefined
("valid code" with no codes or rates specified), cross-checked the plan against the project
`CLAUDE.md`'s two-decimal rounding rule, and read the source to find that `checkout()` returns a dict
so "returns the discounted amount" names nothing real. Those are the findings that stop a wasted
execute round; the nits are not. Both ran ~2 minutes.

The `high` reasoning effort is a determinism requirement, not a measured one: it was not A/B'd
against the default. It is pinned so the same plan gets the same scrutiny on any machine. The flag
combination itself is verified — `-c 'model_reasoning_effort="high"'` alongside `-s read-only`,
`--output-schema`, `-o` and stdin is accepted by `codex-cli 0.146.0`.

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

## C3. Planning council — Phase 1b

Three Codex calls, hard cap, all read-only. Round 1 proposes a decomposition from the original task
**without seeing Claude's**; round 2 critiques Claude's synthesis of the two; round 3 verifies the
revision. Stop early the moment round 2 returns `verified` — that is the common case and it costs
two calls.

Round 2 is the audit this section used to describe, repointed from Claude's first draft at the
synthesis. Its schema, its carve-outs and its ordering-reconciliation rule are unchanged.

**What the extra round buys.** The old audit showed Codex the decomposition first, so its output was
always a reaction to Claude's framing. It could catch a path that does not exist; it could not
produce "this repo wants a different cut of the work". Round 1 exists to get that second cut before
either side has anchored.

**The cost, stated plainly.** The old loop allowed three audit passes and two repair rounds. The
council allows two critique passes and one repair round. Three calls is the budget; this is where it
is spent.

### Run identity and artifacts

Pick a short `run_id` at 1b.0 (a timestamp is fine) and give this run **its own directory**:

| Path (all under `graphify-out/.orchestrate_council_<run_id>/`) | Written at | Holds |
|---|---|---|
| `proposal_schema.json` | 1b.0 | round 1's output schema |
| `critique_schema.json` | 1b.0 | rounds 2–3's output schema |
| `payload<N>.txt` | each round | that round's prompt |
| `round<N>.json` | each round | that round's `-o` result |
| `round<N>.log` | each round | the dispatch's stdout — this is where the session id is |

The directory is not decoration. Cleanup removes **only this run's** directory (SKILL.md 6z). Never
delete by a bare `.orchestrate_*` glob and never "purge stale council scratch": a second
`/forge-orchestrate` may be mid-council in another session, and without a lock you cannot tell its
live payloads from a dead run's leftovers. Own your directory; leave every other one alone.

**Round 1 must not read another run's scratch.** `-s read-only` restricts writes, not reads — it is
not a read allowlist, and Codex can open anything in the repo. A leftover `payload2.txt` from an
earlier run holds an earlier *Claude synthesis*, and round 1 reading it defeats the independence this
whole section exists for. Since deleting it is unsafe, the round 1 payload forbids reading it
instead (see the instruction block below). That is an instruction, not an enforcement — the same
class of guarantee as "do not edit files", and the honest limit of what this design can promise.

### The schemas

Both go in before the first call. Every field is `required` and `additionalProperties` is `false` —
that strictness is the shape known to work with `--output-schema`, so keep it on any field you add.

**Round 1 — the independent proposal.** `..._proposal_schema.json`:

```json
{
  "type": "object",
  "properties": {
    "subtasks": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": { "type": "string" },
          "goal": { "type": "string" },
          "success_criterion": { "type": "string" },
          "depends_on": { "type": "array", "items": { "type": "string" } },
          "files_read": { "type": "array", "items": { "type": "string" } },
          "files_changed": { "type": "array", "items": { "type": "string" } }
        },
        "required": ["id", "goal", "success_criterion", "depends_on",
                     "files_read", "files_changed"],
        "additionalProperties": false
      }
    },
    "risks": { "type": "array", "items": { "type": "string" } },
    "assumptions": { "type": "array", "items": { "type": "string" } },
    "execution_order": { "type": "array", "items": { "type": "string" } }
  },
  "required": ["subtasks", "risks", "assumptions", "execution_order"],
  "additionalProperties": false
}
```

**Shape-valid is not usable.** This schema happily accepts an empty `subtasks`, two subtasks with the
same `id`, a `depends_on` naming an id that does not exist, a dependency cycle, and an
`execution_order` unrelated to the subtasks. Check all five after parsing. A proposal that fails any
of them is **degraded**, not flawed — it spent its call and round 1 is treated as absent (below).

**The prose fields carry the operator's own Codex instructions.** A `goal` or `problem` string comes
back wearing whatever house style the operator's **global `~/.codex/AGENTS.md`** imposes — greetings,
confidence tags, the lot. Observed in fixture runs, including one whose entire prompt was "Reply with
the single word ok" and which still opened with the operator's greeting rule. That global file is the
carrier, not `project_doc_fallback_filenames`: the key routes the *project's* `CLAUDE.md`, while the
style arrives from `~/.codex/AGENTS.md`, which on this machine is a verbatim copy of the operator's
global `~/.claude/CLAUDE.md`. Naming the wrong cause sends an operator to edit the wrong file.

The payloads ask for plain text (below), which handles the common case. Whatever still arrives styled
is cosmetic and is the operator's own configuration, so do not try to strip it: read these fields for
their content and write the plan in your own words, which 1b.2 already requires. Never paste a
proposal string straight into the printed decomposition.

**Rounds 2–3 — the critique.** `..._critique_schema.json`, unchanged from the audit it replaces:

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

### The calls

Rounds 1 and 2 are **fresh sessions**:

```bash
codex exec \
  -m gpt-5.6-sol \
  -c 'model_reasoning_effort="high"' \
  -s read-only \
  --output-schema graphify-out/.orchestrate_council_<run_id>_<which>_schema.json \
  -o graphify-out/.orchestrate_council_<run_id>_round<N>.json \
  - < graphify-out/.orchestrate_council_<run_id>_payload<N>.txt
```

Round 3 **resumes round 2's session** — parent flags before the subcommand, per C6:

```bash
codex exec -s read-only \
  resume <council.round2_session_id> \
  -m gpt-5.6-sol -c 'model_reasoning_effort="high"' \
  --output-schema graphify-out/.orchestrate_council_<run_id>_critique_schema.json \
  -o graphify-out/.orchestrate_council_<run_id>_round3.json \
  - < graphify-out/.orchestrate_council_<run_id>_payload3.txt
```

**Which rounds resume, and why it is not symmetric.** Round 2 is deliberately *fresh*: a session that
still remembers writing its own proposal grades the synthesis against that proposal and returns
preference as `flawed`, which is the exact anchoring round 1 exists to remove. Round 3 *resumes*
round 2 because that session's context is its own findings list, and "is my finding fixed" is the
only question worth asking it. The rule generalises: **resume when the prior context is a defect
list, never when it is a competing proposal.**

`council.round2_session_id` is stored separately from every subtask's `agent_id`. They are different
sessions with different sandboxes, and C6 must never resume a council session into `workspace-write`.

**`-s read-only` does not mean "cannot run tests".** It restricts filesystem writes by the commands
Codex issues; a read-only `pytest` invocation is still a command it can run. The no-execution
requirement is carried by the payload instruction below, not by the sandbox flag. What `-s
read-only` does guarantee is that no planning round mutates the repo.

### Round 1 payload — the independent proposal

**Claude's decomposition is not an input here, and does not exist yet.** Round 1 is dispatched before
1b.2 writes a candidate. That ordering, plus the template below having no slot for a decomposition,
is the whole enforcement mechanism.

The honest statement of the guarantee: *no byte of Claude's candidate reaches round 1, and no such
artifact exists while it runs.* It is not "Codex is uninfluenced by Claude" — Claude chooses the
slice key words, and Codex reads `CLAUDE.md` and `doc/` on its own. Do not claim the stronger thing.

Include, paths not pasted text:

- `[TASK]` verbatim, and `[TASK_CRITERIA]` when it exists.
- `[TASK_FEATURE]` when the task came from a v2 task-list block — the `Builds: Fn` link 5b's spec
  check needs later.
- `[BLAST_RADIUS]` and `[RULED_OUT]` when `[FROM_DIAGNOSIS]` is true, plus the
  `doc/diagnosis-<slug>.md` path itself. These are established facts about the task, not Claude's
  plan, and dropping them makes round 1 re-investigate branches the diagnosis already closed.
- Any assumption 0e resolved with the user. Same reasoning — it is the user's answer, not Claude's
  design.
- The path `graphify-out/graph.json`, the project's `CLAUDE.md`, and the `doc/*.md` paths that exist.
- The **task-level** slice's `FILES` list, labelled advisory: it was produced from Claude's choice of
  key words and is a starting point, not a boundary. `NO_MATCH` → say the graph had no specific match
  and name the universal docs only.
- These questions, verbatim:
  ```
  Propose your own decomposition of this task. Inspect the repository yourself — the file list
  above is an advisory starting point, not a boundary, and you are not bound by it.
  For each subtask give: a one-line goal, a success criterion that is verifiable as written,
  which earlier subtasks it depends on, and the files you expect to read and to change.
  Also return the risks and the assumptions you are making, and the order the subtasks must run.
  You are in a read-only sandbox and cannot run the test suite. That is expected — do not report
  it as a problem.
  Do not read anything under graphify-out/.orchestrate_council_*/ — those are another run's
  planning notes, and reading them would defeat the point of asking you independently.
  Do not run the test suite. Do not write code. Do not edit files.
  Every JSON string field must be plain descriptive text — no greetings, sign-offs or
  confidence tags.
  ```
  Note which carve-outs are absent. "Never flag paths a subtask CREATES" and "missing lint tooling
  is not a finding" belong to the critique rounds — round 1 is not auditing anything, and pasting
  audit carve-outs into a proposal prompt just tells it to look for flaws that are not there.

### Rounds 2–3 payload — the critique

Round 2 gets the synthesis: every subtask's goal, success criterion, deps and gate set, plus the
per-subtask slice `FILES` from 1b.4, `[TASK]`, `[TASK_CRITERIA]`, and **the project `CLAUDE.md`
path** — the same path round 1 carries. Without it the critique cannot do the thing this section
cites as its reason for running on `sol` ("cross-checked the plan against the project `CLAUDE.md`'s
two-decimal rounding rule"): a subtask that violates a project rule is only visible to a round that
was told where the rules live. C1.3's shadow warning applies here too — if an `AGENTS.md` shadows it,
naming the path is what still gets it read. Round 3, resuming, gets **only the revised subtasks and
the instruction below** — the session already holds everything else.

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
  Report defects only. Do not report stylistic preferences.
  Do not run the test suite. Do not write code. Do not edit files.
  Every JSON string field must be plain descriptive text — no greetings, sign-offs or
  confidence tags.
  ```
  All three carve-outs are load-bearing and were added after fixture runs. Without the first, the
  audit reports every new file the plan creates as a missing path and drowns the real findings.
  Without the second, every round emits a spurious "pytest could not start" finding. Without the
  third, any repo with no linter burns a round on "the lint gate is not verifiable" — which 5a
  already handles by design, so rewriting a subtask over it is pure waste.
- Round 3 appends, verbatim:
  ```
  For each finding you returned last turn, check the revised subtask text above and say whether
  it is now fixed. Verify against the text yourself — a claim that something was addressed is
  not evidence that it was. Do not raise new stylistic points.
  ```
  The verification is deliberately not "do not re-report resolved findings". A synthesis asserting
  that it fixed something must not be able to suppress the check on it.

### The budget — 3 calls, hard

`council.calls_used` increments on **every** dispatch, including one that returns nothing usable.
There is no retry: a wasted call is spent.

- Round 2 `verified` → **stop at 2 calls**, continue hands-off. The common case.
- Round 2 `flawed` → Claude revises **only the subtasks the findings name** (unaffected subtasks stay
  byte-identical), re-slices every subtask it revised, then round 3 verifies.
- Round 3 `verified` → continue hands-off.
- Round 3 `flawed` → Claude's plan is authoritative. It does **not** hand the plan back unfinished
  and it does **not** claim consensus: see "Claude finalises" below.

There is no 4th `codex exec`. An unbounded planning loop burns both quotas.

**Degraded rounds.** A missing, empty, unparseable, or semantically invalid `-o` file costs its call
and is never retried. What follows depends on which round died, and the difference matters:

| Round | Effect | Report as |
|---|---|---|
| 1 | No independent proposal. Claude plans alone — the pipeline still works, it just lost the second opinion. Continue to round 2 normally. | `degraded — proposal unusable` |
| 2 | The synthesis was never critiqued. Treat as **unresolved**, not as verified. | `degraded — critique unusable` |
| 3 | Round 2's findings stand unaddressed. Treat as **unresolved**. | `degraded — verification unusable` |

Silence is not agreement. A round that failed to answer must never be scored as `verified`.

### Claude finalises — always, in every terminal state

Before anything is dispatched to Phase 4, Claude produces the authoritative plan and, for each
objection still outstanding, either incorporates it or rejects it **with the repository evidence that
justifies the rejection**. "Codex disagreed and I ignored it" is not a terminal state.

**Ordering conflict.** The synthesis emits per-subtask `deps`. If a round's `execution_order`
disagrees with them, that disagreement is itself a finding Claude reconciles **in the decomposition**
— never a silent override at dispatch time. Two sources of truth for sequencing is a bug. This
reconciliation happens even when the final round is `flawed`, because execution can still proceed
after approval; leaving the order undefined there was the old flow's luxury, not this one's.
Phase 4 follows the reconciled `deps`; the **final** round's `execution_order` is the tiebreaker for
subtasks the deps leave unordered. Round 1's `execution_order` is advisory input to the synthesis
only — it never overrides anything.

**Then apply the approval policy:**

- Everything resolved → continue hands-off. No approval stop. This is the design's normal path.
- Anything unresolved — a round-3 `flawed`, or a degraded round 2 or 3 — → print the final plan, the
  unresolved objection, and the concrete risk it names, then **stop and ask before executing**.
  Do not begin execution on an objection the user has not seen.

The approval stop is a **pause, not an abort**: keep this run's council artifacts (SKILL.md 6z), the
user is expected to answer and continue.

---

## C4. Execute — Phase 4

```bash
codex exec -m gpt-5.6-terra -s workspace-write - < graphify-out/.orchestrate_payload_<n>.txt
```

Use `-m gpt-5.6-sol` instead when the subtask is multi-file integration or design judgment (C2).
Add `-C <worktree abs path>` in worktree mode, `--add-dir <path>` per C1.4. Run in the background per
C2.

**The payload is the Phase 3c payload, verbatim, with nothing removed** — that reuse is the point of
this design. Wrap it: one line before, two after.

Prepend:

```
Read ./CLAUDE.md before you start. It is this project's binding engineering rules and applies to
this subtask even if AGENTS.md is also present.
```

Append:

```
Do NOT git commit, do NOT create branches, do NOT run git add. The orchestrating session owns git.
List every file you created or modified, one path per line, under a final "FILES CHANGED:" heading.
```

The prepended line goes first for the reason 3c gives about its own read gate — a worker that skims
still hits it. It exists because C1.2's config key is not a guarantee: it is a *fallback*, so an
`AGENTS.md` in the repo root shadows `CLAUDE.md` entirely (C1.3), and the key itself lives in a file
this skill must not edit. Naming the path in the payload is the one delivery path that does not
depend on the operator's `~/.codex/config.toml`. It is Codex-only: Claude workers auto-load the
project's `CLAUDE.md`, which is why 3c omits it, and keeping the line here leaves the 3c payload
byte-identical across both backends.

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
council files into it. Match on the **path prefix**, not on exact filenames — `git status
--porcelain` can collapse the whole untracked directory into a single `?? graphify-out/` line.

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
| A dispatch looks finished seconds after it started | the completion check grepped for a string the payload also contains, and matched the echoed prompt | wait on the dispatch's exit, or match `^tokens used` (C2) |
| `error: unexpected argument '-s' found` on a retry | `-s`/`-C` placed after `resume` | move them before the subcommand (C6) |
| Codex reports a file it could not write | path outside the writable root | add `--add-dir <path>` and retry that subtask |
| Empty or unparseable `-o` file after a council round | run died before its final turn | it still counts against the 3 — degrade per C3, never score it `verified` |
| A council round's JSON parses but has duplicate ids, a dangling `depends_on`, a cycle, or an unrelated `execution_order` | schema validity is not semantic validity | same as unparseable — degraded, and the call is spent (C3) |
| Round 3 dispatched but `council.round2_session_id` was never captured | the session id was not read off stdout at round 2 | fall back to a fresh `codex exec` carrying round 2's findings in the payload; it is still call 3 |
| `git status` delta is empty after execute | **check the snapshot cwd first** (C5) — in worktree mode a bare `git status` reads the main checkout and always comes back empty. Otherwise Codex answered without editing, or was killed mid-run | fix the `-C`; if the cwd was right, re-dispatch with the goal restated |
| Reconciled delta contains files from an unrelated subtask | `[NO_COMMIT]`, no moving baseline | expected; 5b's out-of-scope note (single-tree `[NO_COMMIT]`) covers it |
| `NEEDS_CONTEXT` in the final message | Codex lacked a doc or file | supply it and re-dispatch via C6, same as a Claude worker |
