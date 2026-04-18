Create an AI-assisted coding project template in the current working directory.

Use the Write tool to create each file listed below exactly as specified. Do not skip any file. After writing all files, print a summary of what was created.

---

## File 1: `initialprompt.md`

```
You are a [FILL IN: e.g. Senior .NET Solution Architect / Full Stack Developer / Backend Engineer] AI assistant.
Project root contains a /doc folder with the following files:
•	architecture.md
•	database-schema.md
•	domain-model.md
•	api-contract.md
•	solution-structure.md
•	security.md
•	coding-standard.md
•	ai-instruction.md
•	task-list.md
•	progress.txt
•	ui-guideline.md
•	llm-checklist.md

Your role is to generate code strictly according to these documents.
STRICT RULES:
1.	You must read and follow all documents in /doc before generating any code.
2.	You must follow solution-structure.md exactly. No structural changes allowed.
3.	You must not invent tables, fields, enums, or properties.
4.	You must update progress.txt after completing a task.
4a.	You must update doc/changelog.txt after every change, adding a new entry in the format: `Date | Change | Description`.
5.	You must implement ONLY the next incomplete task from task-list.md.
6.	[FILL IN: e.g. You must enforce WorkspaceId multi-tenancy everywhere. / You must enforce OwnerId scoping on all queries.]
7.	You must generate code only in the correct project folder according to solution-structure.md.
8.	You must follow coding-standard.md.
9.	You must confirm before implementing any task that no schema change is required.
Before writing any code: 
Step 1: List all documents you are using. 
Step 2: State which task from task-list.md you are implementing.
Step 3: List which project(s) will be modified. 
Step 4: Confirm no schema changes are required. 
Step 5: Then generate code.
Do not proceed if any rules are unclear. Ask for clarification first.
```

---

## File 2: `subsequentprompt.md`

```
Continue from progress.txt. Implement the next incomplete task only. After completing the task, update progress.txt summary and create an individual progress file in doc/Progress/ named Progress-N.txt (replace N with the task number). Update doc/changelog.txt with a new entry in the format: Date | Change | Description.
```

---

## File 3: `README.txt`

```
===================================================
  AI CODING TEMPLATE – SETUP & USAGE GUIDE
===================================================

COMPATIBLE WITH ANY AI AGENT
------------------------------
This template works with any AI coding assistant:
  - Claude (claude.ai, Claude Code, Cursor with Claude)
  - ChatGPT / GPT-4o (OpenAI)
  - Gemini (Google)
  - GitHub Copilot Chat
  - Windsurf, Cursor, Codeium, or any chat-based AI tool

No AI-specific configuration is required. Just paste the prompts.


BEFORE YOU START
----------------
Fill in all [FILL IN: ...] placeholders across the /doc files to match your project.
Files to fill in:
  doc/ai-instruction.md      – Replace [PROJECT_NAME]
  doc/architecture.md        – Tech stack, layers, naming conventions, rules
  doc/solution-structure.md  – Project/folder layout using your project name
  doc/database-schema.md     – Your tables/entities and fields
  doc/domain-model.md        – Enums, invariants, business rules
  doc/api-contract.md        – Your API endpoints
  doc/security.md            – Roles and auth rules
  doc/ui-guideline.md        – Layout and UX rules (skip if backend-only)
  doc/coding-standard.md     – Language/framework-specific standards
  doc/task-list.md           – Your phased task breakdown
  doc/changelog.txt          – Start date
  initialprompt.md           – AI role (e.g. Senior .NET Architect)

The only file you do NOT need to change: subsequentprompt.md


WORKFLOW
--------
1. INITIAL PROMPT
   Paste the full text of initialprompt.md into your AI chat.
   This gives the AI the full project context (reads all /doc files).

2. SUBSEQUENT PROMPTS
   For every follow-up session, paste subsequentprompt.md instead.
   The AI will continue from progress.txt and implement the next task.

3. AFTER EACH TASK COMPLETES
   a. progress.txt is updated with a short summary of what's done.
      Keep this file concise – it is sent to the AI every session and costs tokens.
   b. A new doc/Progress/Progress-N.txt is created for the completed task
      (replace N with the task number, e.g. Progress-1.txt).

4. REPEAT
   Keep calling subsequentprompt.md until all tasks in task-list.md are done.

5. NEW FEATURES
   Add new tasks to task-list.md under a new phase and continue.


TOKEN EFFICIENCY DESIGN
------------------------
- Full /doc files are read once (initial prompt only).
- Only progress.txt is sent on every subsequent prompt – keep it short.
- Detailed per-task logs go in doc/Progress/ – not sent unless needed.


TIPS FOR SPECIFIC AI TOOLS
----------------------------
Claude Code (CLI):
  Add a CLAUDE.md at the project root pointing to /doc files for
  automatic context loading on every session.

Cursor / Windsurf:
  Add a .cursorrules or .windsurfrules file referencing the /doc folder
  so the AI picks up your architecture rules automatically.

ChatGPT / Gemini:
  Paste initialprompt.md in a new chat, then attach or paste the relevant
  /doc files when asked. Use a Project (ChatGPT) or Gem (Gemini) to persist context.

GitHub Copilot Chat:
  Open the /doc folder in VS Code and reference files inline with #file
  when chatting (e.g. #file:doc/architecture.md).
```

---

## File 4: `doc/ai-instruction.md`

```
# AI Development Rules

You are assisting in developing [PROJECT_NAME].

STRICT RULES:
1. Only implement tasks listed in task-list.md.
2. Do NOT modify database-schema.md unless explicitly instructed.
3. Do NOT invent new fields.
4. Always check architecture.md before coding.
5. If unsure, ask before implementing.
6. [FILL IN: e.g. Never skip WorkspaceId / TenantId filtering on queries.]
7. Always update progress.txt after completing a task.

When generating code:
- Follow the architecture defined in architecture.md.
- Use proper dependency injection.
- Add minimal necessary comments only.
- Do not refactor unrelated files.

Before coding:
1. Restate which task you are implementing.
2. List files that will be modified.
3. Confirm no schema changes are required.
4. Then generate code.
```

---

## File 5: `doc/architecture.md`

```
# Architecture – [PROJECT_NAME]

## Tech Stack
- [FILL IN: e.g. ASP.NET Core 8 Web API / Node.js + Express / Django REST]
- [FILL IN: e.g. EF Core / Prisma / SQLAlchemy]
- [FILL IN: e.g. PostgreSQL / MS SQL / MySQL / MongoDB]
- [FILL IN: e.g. JWT Authentication]
- [FILL IN: e.g. Redis (Caching – Phase 2)]
- [FILL IN: e.g. React / Vue / Blazor (Frontend)]

## Architecture Style
- [FILL IN: e.g. Clean Architecture / MVC / Layered]
- [FILL IN: e.g. CQRS pattern]
- [FILL IN: e.g. Repository + Unit of Work]
- [FILL IN: e.g. Multi-tenant (scoped by TenantId / WorkspaceId / OwnerId)]

## Project Layers
- [FILL IN: e.g. Domain / Application / Persistence / Infrastructure / API]

## [FILL IN: e.g. Multi-Tenant Rule / Scoping Rule]
[FILL IN: e.g. All entities MUST include WorkspaceId. No query is allowed without WorkspaceId filtering.]

## Naming Convention
- [FILL IN: e.g. Entities: PascalCase]
- [FILL IN: e.g. DB tables: Singular]
- [FILL IN: e.g. Primary key: Id]
- [FILL IN: e.g. Foreign keys: {EntityName}Id]

## Non-Negotiable Rules
- [FILL IN: e.g. No direct DbContext access in Controller]
- [FILL IN: e.g. All business logic must go through Application/Service layer]
- [FILL IN: e.g. No hardcoded type values – use enums or config]
```

---

## File 6: `doc/solution-structure.md`

```
# Solution Structure – [PROJECT_NAME]
Stack: [FILL IN: e.g. ASP.NET Core 8 + EF Core + MS SQL / Node.js + Prisma + PostgreSQL]
Architecture: [FILL IN: e.g. Clean Architecture + CQRS + MediatR / Layered MVC]

---

# Root Structure

[ProjectName].sln  (or package.json / pyproject.toml / etc.)

/src
    [ProjectName].[Layer1]       # e.g. Domain
    [ProjectName].[Layer2]       # e.g. Application
    [ProjectName].[Layer3]       # e.g. Persistence
    [ProjectName].[Layer4]       # e.g. Infrastructure
    [ProjectName].[Layer5]       # e.g. API
    [ProjectName].[Layer6]       # e.g. Contracts (optional)
    [ProjectName].[Layer7]       # e.g. Worker (optional)

/tests
    [ProjectName].[Layer1].Tests
    [ProjectName].[Layer2].Tests
    [ProjectName].IntegrationTests

---

# Layer Dependency Rule (STRICT)
# [FILL IN: Define which layers can depend on which. Example:]
# API → Application → Domain
# Infrastructure → Application
# Persistence → Domain
# Domain → (no dependency)

---

# [ProjectName].[Layer1] (e.g. Domain – Core Business Rules)
# [FILL IN: No external dependencies allowed.]

[ProjectName].[Layer1]
│
├── Common
│   ├── [FILL IN: e.g. BaseEntity.cs]
│   └── [FILL IN: e.g. AuditableEntity.cs]
│
├── Enums
│   └── [FILL IN: e.g. StatusType.cs / RoleType.cs]
│
├── Entities
│   └── [FILL IN: e.g. User.cs / Order.cs / Product.cs]
│
└── Interfaces
    └── [FILL IN: e.g. IRepository.cs / IUnitOfWork.cs]

---

# [ProjectName].[Layer2] (e.g. Application – Use Cases)

[ProjectName].[Layer2]
│
├── Common
│   ├── Interfaces
│   │   └── [FILL IN: e.g. ICurrentUserService.cs / ICacheService.cs]
│   └── Exceptions
│       └── [FILL IN: e.g. NotFoundException.cs / ForbiddenException.cs]
│
├── [Feature1]
│   ├── Commands
│   │   └── [FILL IN: e.g. CreateFeature1 / UpdateFeature1]
│   └── Queries
│       └── [FILL IN: e.g. GetFeature1 / GetFeature1ById]
│
└── [Feature2]
    ├── Commands
    └── Queries

---

# [ProjectName].[Layer3] (e.g. Persistence – Database Layer)

[ProjectName].[Layer3]
│
├── Context
│   └── [FILL IN: e.g. ApplicationDbContext.cs]
│
├── Configurations
│   └── [FILL IN: one config file per entity]
│
├── Repositories
│   └── [FILL IN: e.g. UserRepository.cs / UnitOfWork.cs]
│
└── Migrations

---

# [ProjectName].[Layer4] (e.g. Infrastructure – External Services)

[ProjectName].[Layer4]
│
├── [FILL IN: e.g. Identity / Auth]
├── [FILL IN: e.g. Caching]
├── [FILL IN: e.g. Email]
├── [FILL IN: e.g. Storage]
└── DependencyInjection.cs

---

# [ProjectName].[Layer5] (e.g. API – Presentation Layer)

[ProjectName].[Layer5]
│
├── Controllers
│   └── [FILL IN: one controller per resource]
│
├── Middleware
│   └── [FILL IN: e.g. ExceptionMiddleware.cs / AuthMiddleware.cs]
│
└── Program.cs / Startup.cs / index.ts / app.py

---

# Enterprise Rules (Non-Negotiable)
1. [FILL IN: e.g. All entities must include TenantId for multi-tenancy.]
2. [FILL IN: e.g. All delete operations must be soft delete.]
3. [FILL IN: e.g. No controller may access the database directly.]
4. [FILL IN: e.g. All write operations must go through the service/command layer.]
5. [FILL IN: e.g. No business logic in Controllers.]
6. [FILL IN: e.g. Use validation library for all input commands.]

---

# AI Instruction

The AI must:
- Follow this structure strictly.
- Not invent new projects or layers.
- Not merge layers.
- Not introduce cross-layer dependencies.
- Generate code only inside the correct project folder.

Any deviation must be rejected.
```

---

## File 7: `doc/database-schema.md`

```
# Database Schema
# [FILL IN: Define one section per table/collection. Use the format below.]
# Remove placeholder sections that don't apply to your project.

## [Entity1]
- Id ([FILL IN: e.g. Guid / int / ObjectId])
- [FILL IN: field name] ([FILL IN: type, e.g. nvarchar(255) / string / int])
- [FILL IN: field name]
- CreatedDate
- [FILL IN: e.g. IsDeleted (bool) – for soft delete]

## [Entity2]
- Id
- [Entity1]Id (FK → [Entity1])
- [FILL IN: field name]
- [FILL IN: field name]
- CreatedDate

## [Entity3]
- Id
- [Entity2]Id (FK → [Entity2])
- [FILL IN: field name]
- [FILL IN: field name]

## [Entity4]  (optional)
- Id
- [FILL IN: fields]

# Notes:
# - [FILL IN: e.g. All tables must include a WorkspaceId / TenantId column.]
# - [FILL IN: e.g. All deletes are soft deletes via IsDeleted flag.]
# - Do NOT add tables or fields not listed here.
```

---

## File 8: `doc/domain-model.md`

```
# Domain Model Rules

## [FILL IN: Enum name, e.g. StatusType / RoleType / CategoryType]
- [FILL IN: enum value 1, e.g. Active]
- [FILL IN: enum value 2, e.g. Inactive]
- [FILL IN: enum value 3]
- [FILL IN: add more as needed]

## [FILL IN: Entity1] Invariants
- [FILL IN: e.g. [Entity1] must belong to a [ParentEntity]]
- [FILL IN: e.g. [Entity2] must belong to a [Entity1]]
- [FILL IN: add more structural rules]

## Business Rules
- [FILL IN: e.g. Cannot delete a record if child records exist (soft delete only)]
- [FILL IN: e.g. [FieldName] must be unique within [scope]]
- [FILL IN: e.g. Only one [X] of type [Y] allowed per [Entity]]
- [FILL IN: add more business rules]
```

---

## File 9: `doc/api-contract.md`

```
# API Contract
# [FILL IN: Define all endpoints. Group by resource. Use REST conventions.]

## Auth
POST /api/auth/login
POST /api/auth/register
[FILL IN: e.g. POST /api/auth/refresh-token]

## [FILL IN: Resource 1, e.g. Users / Workspaces / Organizations]
GET    /api/[resource1]
POST   /api/[resource1]
GET    /api/[resource1]/{id}
PUT    /api/[resource1]/{id}
DELETE /api/[resource1]/{id}

## [FILL IN: Resource 2, e.g. Projects / Boards]
GET    /api/[resource2]/{[parentId]}
POST   /api/[resource2]
GET    /api/[resource2]/{id}
PUT    /api/[resource2]/{id}
DELETE /api/[resource2]/{id}

## [FILL IN: Resource 3]
POST   /api/[resource3]
PUT    /api/[resource3]/{id}
DELETE /api/[resource3]/{id}

## [FILL IN: Nested resource, e.g. Item values / Attachments]
PUT /api/[resource2]/{parentId}/[subresource]/{id}

# Notes:
# - [FILL IN: e.g. All endpoints except /auth require JWT in Authorization header.]
# - [FILL IN: e.g. All list endpoints support ?page=&pageSize= query params.]
```

---

## File 10: `doc/security.md`

```
# Security Model

## Roles
- [FILL IN: e.g. Owner]
- [FILL IN: e.g. Admin]
- [FILL IN: e.g. Member]
- [FILL IN: e.g. Viewer]

## Rules
- [FILL IN: e.g. All queries must be filtered by TenantId / WorkspaceId / OwnerId]
- Policy-based authorization
- JWT required for all endpoints
- [FILL IN: e.g. Soft delete for all entities – no hard deletes]
- Audit log required for:
    - [FILL IN: e.g. Delete operations]
    - [FILL IN: e.g. Permission changes]
    - [FILL IN: e.g. Sensitive data access]
```

---

## File 11: `doc/ui-guideline.md`

```
# UI Guidelines

## Layout
- [FILL IN: e.g. Left Sidebar: Navigation / Workspace list]
- [FILL IN: e.g. Top Bar: User profile + Notifications]
- [FILL IN: e.g. Main Area: Primary content view]

## Views
- [FILL IN: e.g. List (default)]
- [FILL IN: e.g. Grid / Kanban / Calendar / Table]
- [FILL IN: e.g. Detail / Modal view]

## UX Rules
- [FILL IN: e.g. Inline editing for data cells]
- [FILL IN: e.g. Lazy loading for large data sets]
- [FILL IN: e.g. Confirm dialogs before destructive actions]
- [FILL IN: e.g. Toast notifications for success/error feedback]
```

---

## File 12: `doc/coding-standard.md`

```
# Coding Standards

- [FILL IN: e.g. Use async/await everywhere / no synchronous DB calls]
- [FILL IN: e.g. Validate all inputs using FluentValidation / Joi / Zod / class-validator]
- [FILL IN: e.g. Use MediatR / service layer for all commands and queries]
- [FILL IN: e.g. Use DTOs – never expose entities directly in API responses]
- [FILL IN: e.g. Always pass CancellationToken through async call chains]
- [FILL IN: e.g. Use dependency injection for all services]
- [FILL IN: e.g. No magic strings – use constants or enums]
```

---

## File 13: `doc/task-list.md`

```
# Master Task List

## Phase 1 – Core

[ ] Setup solution/project structure
[ ] Setup [FILL IN: architecture layers / framework boilerplate]
[ ] Configure [FILL IN: database + ORM, e.g. EF Core + MS SQL / Prisma + PostgreSQL]
[ ] Create [FILL IN: Entity 1] + migration
[ ] Create [FILL IN: Entity 2] + migration
[ ] Create [FILL IN: Entity 3] + migration
[ ] Implement [FILL IN: Entity 1] CRUD
[ ] Implement [FILL IN: Entity 2] CRUD
[ ] Implement [FILL IN: Entity 3] CRUD
[ ] [FILL IN: additional core task]

## Phase 2 – Advanced

[ ] [FILL IN: e.g. Role-based authorization]
[ ] [FILL IN: e.g. File upload / storage]
[ ] [FILL IN: e.g. Real-time / WebSocket integration]
[ ] [FILL IN: e.g. Email notifications]
[ ] [FILL IN: e.g. Audit logging]

## Phase 3 – Scaling

[ ] [FILL IN: e.g. Caching layer (Redis)]
[ ] [FILL IN: e.g. Performance indexing]
[ ] [FILL IN: e.g. Pagination optimization]
[ ] [FILL IN: e.g. Background job processing]
```

---

## File 14: `doc/llm-checklist.md`

```
# Pre-Code Checklist

Before any code generation, confirm:

[ ] Architecture rules followed (see architecture.md)
[ ] No schema invention – only fields in database-schema.md
[ ] [FILL IN: e.g. TenantId / WorkspaceId scoping enforced on all queries]
[ ] Correct project layer / folder targeted (see solution-structure.md)
[ ] No business logic in controllers / handlers
[ ] Task scope respected – only one task at a time
```

---

## File 15: `doc/changelog.txt`

```
# Changelog

## Format
Date | Change | Description

## [FILL IN: YYYY-MM-DD]
- Initial project setup
- Created all management files
```

---

## File 16: `doc/progress.txt`

```
# Progress

## Current Status
Project not started.

## Summary
Waiting to begin Phase 1: Foundation
```

---

## File 17: `doc/Progress/Progress-N.txt`

```
# Task Progress - Task N
# Replace N with the task number (e.g. Progress-1.txt, Progress-2.txt)

## Task
[Task name from task-list.md]

## Status
[ ] In Progress  [ ] Completed

## Files Modified
-

## Notes
```

---

After writing all 17 files, print:

```
✅ ContextForge template created successfully!

Files created:
  initialprompt.md
  subsequentprompt.md
  README.txt
  doc/ai-instruction.md
  doc/architecture.md
  doc/solution-structure.md
  doc/database-schema.md
  doc/domain-model.md
  doc/api-contract.md
  doc/security.md
  doc/ui-guideline.md
  doc/coding-standard.md
  doc/task-list.md
  doc/llm-checklist.md
  doc/changelog.txt
  doc/progress.txt
  doc/Progress/Progress-N.txt

Next steps:
1. Fill in all [FILL IN: ...] placeholders in the /doc files
2. Paste initialprompt.md into your AI chat to start your first session
3. Use subsequentprompt.md for every session after that
```
