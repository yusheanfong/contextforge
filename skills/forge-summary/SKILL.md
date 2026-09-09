---
name: forge-summary
description: Plain-English catch-up in two modes (/forge-summary). `session` — the default — recaps what THIS session has done, what is next, and what is still open, checked against git and doc/. `project` explains what the codebase is, from its README, docs and graph. Report-only — never writes, never commits, never judges the code. Triggers include "/forge-summary", "/forge-summary project", "catch me up", "where are we", "recap this session", "what have we done so far", "summarize what we did", "what is this codebase", "explain this repo".
argument-hint: "[session | project]"
allowed-tools: Read, Grep, Glob, Bash
---

# /forge-summary — Plain-English Catch-Up

Two questions, two modes:

- **`session`** (the default) — *what have we done, and what is next?* Reads this session, then
  checks it against what git and `doc/` actually say.
- **`project`** — *what is this codebase?* Reads the README, the docs and the graph. Never reads the
  session.

Reads `$ARGUMENTS` as the mode.

## CONTRACT (non-negotiable guarantees)

- **Writes nothing, ever.** `Write` and `Edit` are both absent from `allowed-tools`. Unlike
  `/forge-diagnose` — which permits `Write` because a handoff file has to be produced and restrains
  it with prose — there is no permitted write here at all. The output is chat text and nothing else.
- **Never commits, checks out, stashes, or touches a ref.** Every `git` call below is a read.
- **Nothing slow, nothing with side effects.** No build, no test suite, no graph rebuild, no
  `graphify` call, no network. A summary that takes a minute is not a summary.
- **No graph required.** Uses `graphify-out/` when it exists, works fine without it.
- **Plain English is the deliverable.** No phase numbers, no gate names, no internal vocabulary, no
  jargon from this file. A sentence that only makes sense to someone who has read this skill is a
  defect — rewrite it.
- **One screen, five sections.** Too long? Cut the least decision-relevant line. Never add a sixth
  section.
- **Never invent.** Every line traces to something in this session or observed on disk. What you
  cannot source, you drop — or you say plainly that you don't know.

---

## PHASE 0: Resolve the mode

Take the leading bare word of `$ARGUMENTS`. It is a subcommand, the same shape as
`/forge-contextmap sync` and `/forge-orchestrate codex` — not a flag.

- `project` → `[MODE] = project`
- `session`, or nothing at all → `[MODE] = session`
- anything else → `[MODE] = session`, and print exactly one line first:
  ```
  ℹ️ Ignored "<word>" — /forge-summary takes `session` (the default) or `project`.
  ```

Then run **only** that mode's section. Do not announce the mode; the output says which it is.

---

## SESSION MODE — what we have done, and what is next

### S1. Recall this session

From the conversation so far, gather: what was asked, what was decided, what was actually changed or
run, what is still pending. This is the primary source — nothing on disk records it.

**If the session has no history yet** (this is the first thing invoked), stop and print exactly:

```
Nothing to recap yet — this session has no history.
Try /forge-summary project for what this codebase is.
```

Do not substitute a repo-state summary under a session heading.

If earlier context was compacted or truncated, that is a real limit — disclose it in *Grounded on*
rather than presenting a partial recap as complete.

### S2. Ground the recall

Recall says what was *intended*; these say what actually *happened*. Run each as a separate call
(`||` does not exist in PowerShell 5.1) and ignore failures — a non-git directory fails all four,
which is a fact to report, not an error to fix:

```bash
git branch --show-current
```
```bash
git status --porcelain
```
```bash
git log --oneline -5
```
```bash
git diff --stat
```

Then use **Glob** to see which of these exist, and the **Read tool** to read them — never `tail`,
`head` or `cat`, none of which exist as cmdlets on Windows:

- `doc/progress.txt` and `doc/changelog.txt` — the last entries only
- `doc/task-list.md` — the first unchecked task whose `Depends on` are all done
- whatever this session was working *on*, when there is one: the active plan file under the
  harness's plan directory (`~/.claude/plans/`), `doc/plan-*.md`, `doc/diagnosis-*.md`,
  `doc/release-readiness.md`

Read nothing else. This is a recap, not an investigation — do not open source files to check whether
a change was correct. **Where the recall and the repo disagree, the repo wins and the disagreement is
named in the output.**

**A report is not an observation.** If a subagent, another command, or a gate run *told* you a
result, attribute it ("the test run reported 12 passing") rather than stating it as fact you saw.

### S3. Print

```
Now
  [1–2 lines: where things stand]

Done
  [what actually changed — one line each]

Next
  [the immediate next actions — 1–3 lines]

Open
  [decisions waiting on you, blockers, risks — or "nothing open"]

Grounded on
  [the sources actually read, one line]
```

Nothing about the codebase at large belongs here. That is what `project` is for.

---

## PROJECT MODE — what this codebase is

### P1. Read what exists

Never read the session in this mode. Glob first, then read only what is there:

- `README.md` — first, and usually the best source
- `doc/prd.md` (the goal and the feature list), `doc/architecture.md`, `doc/solution-structure.md`
- `graphify-out/GRAPH_REPORT.md` when present — counts and the biggest files
- when there is no `doc/`: whatever manifests exist — `package.json`, `pyproject.toml`, `go.mod`,
  `Cargo.toml`, `*.csproj`, `.claude-plugin/*.json`
- what is being worked on now:
  ```bash
  git log --oneline -10
  ```

Never blind-read source files, and never walk the whole tree. A missing source is not a failure —
skip it and name it in *Grounded on*.

### P2. Print

```
What it is
  [1–2 lines: what this project does, for whom]

How it's built
  [language, framework, key dependencies — what is actually in the manifests]

How it's laid out
  [the handful of directories that matter, and what lives in each]

Where it stands
  [what recent commits are working on; maturity if the docs say]

Grounded on
  [the sources actually read, and the notable ones that were missing]
```

Name real things — real files, real dependencies, real commands. Words like "modern", "robust" and
"scalable" carry no information; a claim you cannot point at gets dropped.

---

## PLAN MODE

Nothing here is restricted, because nothing here is written. Run in full and print the summary as
normal chat output.

- Do **not** spawn exploration or planning subagents. The reads above are the whole job.
- Do **not** write a plan file and do **not** call `ExitPlanMode`. This answers a question; it does
  not propose work, so there is nothing to approve.
- The six-heading plan shape the other skills emit does not apply here.

---

## RED FLAGS — stop and go back

| Thought | Reality |
|---------|---------|
| "Let me open the source to check that change was right" | That's a review, not a recap. `/forge-audit` and `/forge-orchestrate`'s gates judge code; this command doesn't. |
| "The docs don't exist — I'll offer to scaffold them" | `/forge-contextmap` writes docs. This one only reads them. Say what was missing and stop. |
| "It probably passed" | Then it doesn't go in. Say what was observed, or say you don't know. |
| "I'll mention the pipeline stage it stopped at" | The reader asked for plain English. Say what happened, not which stage. |
| Reaching for `Write` or `Edit` | Neither is in `allowed-tools`, and neither is in scope. |
| "One more section would make this clearer" | Five. A sixth section is the thing that makes it unreadable. |
| Session mode drifting into what the codebase is | That's `project`. Different question, different run. |

---

## VERIFY WHEN DONE

1. `/forge-summary`, `/forge-summary session` and `/forge-summary project` each resolve to the right
   mode, and an unrecognized word prints the ignore line and still runs session mode.
2. Every "Done" line matches something in `git log`, `git status` or this session's own record.
3. In a repo with no `doc/` and no `graphify-out/`, both modes still print and name what was missing.
4. `git status --porcelain` is identical before and after the run — nothing was written.

---

## NOTES

- **Boundary with `/forge-contextmap`:** contextmap *writes* the docs and *builds* the graph.
  `/forge-summary project` only reads and explains them. Overlapping words, opposite verbs.
- **Boundary with `/forge-audit`:** audit judges the code and returns a delete-list.
  `/forge-summary` judges nothing.
- **Session mode is the default on purpose.** It is the question you cannot answer any other way —
  the project question can always be answered later from the same files.
- **It carries no copy of the shared `plan-mode` block.** That block exists to say which phases are
  blocked because they write. Nothing here writes, so there is nothing to block, and a verbatim copy
  would be a page describing restrictions that do not apply.
