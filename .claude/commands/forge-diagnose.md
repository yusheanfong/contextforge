# /forge-diagnose — Diagnose Without Changing Code + Cross-Session Handoff

This command is implemented as a skill: `.claude/skills/forge-diagnose/`.

Invoke the **forge-diagnose** skill: read `.claude/skills/forge-diagnose/SKILL.md` (project copy;
fall back to `~/.claude/skills/forge-diagnose/SKILL.md`) and follow it exactly — its CONTRACT and
PHASE 0–6 drive the session. Pass `$ARGUMENTS` through unchanged (the issue to diagnose).
