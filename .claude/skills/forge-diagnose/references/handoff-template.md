# Handoff Template

The output is a **prompt addressed to the next agent**, not a report for a human. Write it in the
imperative. The next session pastes this file's contents and starts fixing immediately.

## Anchor on symbols, not lines

Every location is written as `path` → `functionName()` or `ClassName.method`, with `(~line N)` as a
**hint only**. Line numbers drift between the diagnosis session and the fix session; a stale number
sends the next agent re-exploring, which defeats the entire purpose of the handoff. This rule
applies to sections 3, 5 and 8 alike.

## Template

Fill every section — if one is genuinely empty, write "none" and say why. The `<!-- ... -->` lines
below are instructions to YOU: act on them, then remove them. They never appear in the written
handoff.

**No guesses in this document.** Only `[Certain]` (verified) and `[Likely]` (inference from
evidence you cite) appear. Anything you could not verify goes in section 6b as an open question —
never dressed up as a finding. A handoff containing a guess sends the next session exploring, which
defeats its purpose.

````markdown
# Diagnosis: [ISSUE — one line]

Diagnosed [DATE] by /forge-diagnose. Status: root cause identified, fix NOT applied.

## 1. Task

Fix [ISSUE] in `[path]` → `[symbol]`.

## 2. Do not re-explore

This diagnosis is complete. The files in section 5 are the full relevant set — do not search the
codebase for the root cause again, and do not re-investigate anything in section 6. If section 6b
lists open questions, resolve those first — they are the only sanctioned unknowns. Then read the
files listed, apply the fix in section 7, verify with section 9.

## 3. Root cause

[Confidence tag] [What is actually wrong, in 1-3 sentences.]

Evidence:
- `[path]` → `[symbol]` (~line N) — [what this line/function does wrong or proves]
- `[path]` → `[symbol]` (~line N) — [next link in the chain]

Trace: [origin of the bad value] → [what passes it along] → [where it surfaces as the symptom].

## 4. Reproduction

```bash
[exact command]
```

Expected failing output:
```
[shortest decisive output, verbatim]
```

<!-- If it could not be reproduced, replace the above with: -->
Not reproducible in this environment because [reason]. The diagnosis below is [Likely], not
[Certain] — confirm it before fixing.

## 5. Relevant files

- `[path]` → `[symbol]` — [its role in the bug, one line]
- `[path]` → `[symbol]` — [its role]

Nothing else in the codebase is relevant to this fix.

## 6. Ruled out — do not re-investigate

- **[Hypothesis]** — eliminated: [the evidence that killed it]
- **[Hypothesis]** — eliminated: [the evidence that killed it]

## 6b. Open questions — resolve BEFORE fixing

<!-- Unknowns the diagnosis session could not verify without changing code. Usually "none". Never move an
     unverified claim into section 3 to make this section empty. -->
- [What is unknown] — [why it couldn't be verified] — [how to resolve it: command to run, thing
  to check, question for the user]

## 7. Proposed fix

[Specific enough to execute. Name the file and the symbol. State what changes and to what. If there
is a design choice the next session must make, say so explicitly rather than picking silently.]

## 8. Blast radius

Other code touching what section 7 changes:
- `[path]` → `[symbol]` — [how it depends on the changed behavior]

<!-- Or: "No other call sites — [symbol] is only used by [caller]." -->

## 9. Verification

- **Before the fix:** `[command]` fails with `[output]`
- **After the fix:** the same command passes
- **Regression check:** `[full test command]` — nothing else breaks
- [If no test covers this: write the failing test first. Test file: `[path]`]

## 10. Execute with

- Multi-file fix or one that needs CI gates and commits → `/forge-orchestrate [task description]`
- One-file surgical fix → apply section 7 directly, then run section 9
````
