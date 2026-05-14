Create a CLAUDE.md file for this project using the raw goal description provided by the user.

Raw goal input: $ARGUMENTS

---

## Instructions

1. Rephrase and enhance the raw goal into 2–4 clear, professional sentences that describe:
   - What the system does
   - Who it's for (if inferable)
   - The key outcome or value it delivers
   Do not invent features not implied by the input. Keep it concise but complete.

2. Use the Write tool to create `CLAUDE.md` in the current working directory with exactly this content (replace the placeholder with your enhanced goal text):

```
# Final Goal

[Your enhanced goal text here — 2–4 polished sentences]

---

## Rules & Behaviour

Read and follow all rules in `initialprompt.md` before any action.

---

## Doc Reference

All project reference files are in the /doc folder.

### Always load these first (every session):
- doc/task-list.md — current tasks and phases
- doc/progress.txt — current task pointer
- doc/architecture.md — system design and layer rules

### Load only when the task requires it:
- doc/database-schema.md — when task touches entities, migrations, or queries
- doc/api-contract.md — when task touches endpoints or DTOs
- doc/domain-model.md — when task touches enums or business rules
- doc/solution-structure.md — when adding new files or projects
- doc/coding-standard.md — when generating any code
- doc/security.md — when task touches auth, roles, or data access
- doc/ui-guideline.md — when task touches UI or frontend

Do not load docs that are irrelevant to the current task.
```

3. After writing the file, print:

```
✅ CLAUDE.md created successfully!

The final goal has been written. Doc navigation guide is included.

Next steps:
1. Run /setupcode to generate the full /doc folder structure
2. Fill in all [FILL IN: ...] placeholders in the /doc files
3. Paste initialprompt.md into your AI chat to start your first session
```
