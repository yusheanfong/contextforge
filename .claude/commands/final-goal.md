Take the raw goal description and set up a complete AI coding project template in the current directory.

Raw goal input: $ARGUMENTS

---

## Step 1 — Enhance the goal

Rephrase the raw goal into 2–4 polished sentences describing:
- What the system does
- Who it's for (if inferable)
- The key outcome or value it delivers

Do not invent features not implied by the input.

---

## Step 2 — Present the plan and ask for confirmation

Show the user:

**Enhanced goal:**
[your enhanced goal text from Step 1]

**CLAUDE.md will contain:**
- Final goal (your enhanced goal)
- Role placeholder (user fills in their tech stack / AI expertise level)
- Doc reference guide (which files Claude loads every session vs. on demand)
- 10 strict rules (with 1 customizable placeholder)
- Before/After every task checklists

**Doc folder will be created with 12 files:**
- doc/architecture.md
- doc/solution-structure.md
- doc/database-schema.md
- doc/domain-model.md
- doc/api-contract.md
- doc/security.md
- doc/ui-guideline.md
- doc/coding-standard.md
- doc/task-list.md
- doc/changelog.txt
- doc/progress.txt
- doc/Progress/Progress-N.txt

Then ask: **"Type YES to create all files, or describe any changes you'd like."**

Do NOT proceed to Step 3 until the user explicitly confirms with YES (case-insensitive). If the user requests changes, revise the enhanced goal or adjust the plan and re-present. Wait for YES.

---

## Step 3 — Create CLAUDE.md

Use the Write tool to create `CLAUDE.md` in the current working directory. Replace `[Enhanced goal text]` with your enhanced goal from Step 1:

```
# Final Goal

[Enhanced goal text — 2–4 polished sentences]

---

## Role

You are a [FILL IN: e.g. Senior .NET Solution Architect / Full Stack Developer / Backend Engineer] AI assistant helping build this project.

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

---

## Strict Rules

1. Only implement the next incomplete task from task-list.md — one task per session.
2. Do NOT modify database-schema.md unless explicitly instructed.
3. Do NOT invent tables, fields, enums, or properties — use only what is defined.
4. Follow architecture.md and solution-structure.md exactly — no structural deviations.
5. Follow coding-standard.md for all generated code.
6. No business logic in controllers or handlers.
7. [FILL IN: e.g. Enforce WorkspaceId / TenantId scoping on all queries.]
8. Generate code only in the correct project folder per solution-structure.md.
9. Do not refactor unrelated files.
10. If unsure, ask before implementing.

---

## Before Every Task

1. State which task you are implementing (from task-list.md).
2. List which /doc files you loaded and why.
3. List the files that will be modified.
4. Confirm no schema changes are required.
5. Then generate code.

---

## After Every Task

- Update doc/progress.txt — current task pointer only (keep it short).
- Create doc/Progress/Progress-N.txt (N = task number) with a brief task summary.
- Append one line to doc/changelog.txt: `YYYY-MM-DD | Task N | Description`.
```

---

## Step 4 — Create /doc folder files

Use the Write tool to create each file below exactly as specified.

### File 1: `doc/architecture.md`

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

### File 2: `doc/solution-structure.md`

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

### File 3: `doc/database-schema.md`

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

### File 4: `doc/domain-model.md`

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

### File 5: `doc/api-contract.md`

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

### File 6: `doc/security.md`

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

### File 7: `doc/ui-guideline.md`

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

### File 8: `doc/coding-standard.md`

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

### File 9: `doc/task-list.md`

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

### File 10: `doc/changelog.txt`

```
# Changelog

## Format
Date | Change | Description

## [FILL IN: YYYY-MM-DD]
- Initial project setup
- Created all management files
```

### File 11: `doc/progress.txt`

```
# Progress

CURRENT TASK: 1 — Setup solution/project structure
LAST COMPLETED: None — project not started

> Full per-task logs are in doc/Progress/Progress-N.txt
```

### File 12: `doc/Progress/Progress-N.txt`

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

## Step 5 — Print success message

After writing all files, print:

```
✅ ContextForge setup complete!

Files created:
  CLAUDE.md            ← auto-loaded every session (contains all rules)
  doc/architecture.md
  doc/solution-structure.md
  doc/database-schema.md
  doc/domain-model.md
  doc/api-contract.md
  doc/security.md
  doc/ui-guideline.md
  doc/coding-standard.md
  doc/task-list.md
  doc/changelog.txt
  doc/progress.txt
  doc/Progress/Progress-N.txt

Next steps:
1. Fill in all [FILL IN: ...] placeholders in CLAUDE.md — especially Role (line 9) and Rule 7
2. Fill in all [FILL IN: ...] placeholders across the /doc files
3. Start a new Claude session — CLAUDE.md loads automatically, no paste needed
4. Claude will read task-list.md, progress.txt, and architecture.md every session
```
