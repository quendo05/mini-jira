# Mini Jira – Claude Code Guide

Jira-inspired task management app. Full-stack: ASP.NET Core 10 backend + React 19 frontend, orchestrated with .NET Aspire.

---

## Project Layout

```
Mini-Jira---Web-Services-Project/
├── src/
│   ├── MiniJiraAspire.AppHost/       # .NET Aspire orchestrator (start here for local dev)
│   ├── MiniJiraAspire.Server/        # Backend – single ASP.NET Core project
│   │   ├── Endpoints/                # Minimal API groups (thin HTTP layer)
│   │   ├── Features/                 # Vertical slices: Commands/ + Queries/ per feature
│   │   ├── Models/                   # Domain entities + DTOs
│   │   ├── Persistence/              # EF Core DbContext, migrations, repositories
│   │   ├── Services/Auth/            # JWT token service, password hasher
│   │   ├── docker-compose.yml        # PostgreSQL container
│   │   └── README.md                 # DB setup instructions
│   ├── frontend/                     # React 19 + TypeScript + Vite
│   │   ├── src/features/             # Feature folders (auth, tasks, projects, epics, …)
│   │   ├── src/components/           # Reusable UI components (Shadcn/Radix)
│   │   ├── src/pages/                # Page-level components
│   │   └── src/store/                # Jotai atoms
│   ├── MiniJira.Test/                # Unit tests (xUnit + Moq)
│   └── MiniJira.IntegrationTests/    # Integration tests (ASP.NET Mvc.Testing)
└── docs/arc.md                       # Architecture deep-dive
```

---

## Running the Project

### Prerequisites
- .NET 10.0 SDK
- Node.js `^20.19.0` or `>=22.12.0`
- Docker (for PostgreSQL)

### 1. Database setup (one-time)

Create `src/MiniJiraAspire.Server/.env`:
```
POSTGRES_USER=<user>
POSTGRES_PASSWORD=<password>
POSTGRES_DB=<db-name>
ConnectionStrings__DefaultConnection=Host=localhost;Port=<port>;Database=<db-name>;Username=<user>;Password=<password>
```

Then start the container from `src/MiniJiraAspire.Server/`:
```bash
docker compose up -d
```

### 2. Apply migrations (first run or after schema changes)
```bash
# run from src/MiniJiraAspire.Server/
dotnet ef database update
```

### 3a. Start everything via Aspire (recommended)
```bash
# run from src/MiniJiraAspire.AppHost/
dotnet run
```
Aspire orchestrates the backend, frontend (Vite), and database together.

### 3b. Start services individually
```bash
# Backend (port 5413)
cd src/MiniJiraAspire.Server && dotnet run

# Frontend (port 5173)
cd src/frontend && npm run dev
```

---

## Common Commands

| Task | Command | Directory |
|------|---------|-----------|
| Start Aspire (all services) | `dotnet run` | `src/MiniJiraAspire.AppHost/` |
| Start backend only | `dotnet run` | `src/MiniJiraAspire.Server/` |
| Start frontend only | `npm run dev` | `src/frontend/` |
| Build frontend | `npm run build` | `src/frontend/` |
| Lint frontend | `npm run lint` | `src/frontend/` |
| Format frontend | `npm run biome:format` | `src/frontend/` |
| Add EF migration | `dotnet ef migrations add <Name>` | `src/MiniJiraAspire.Server/` |
| Apply EF migrations | `dotnet ef database update` | `src/MiniJiraAspire.Server/` |
| Run unit tests | `dotnet test` | `src/MiniJira.Test/` |
| Run integration tests | `dotnet test` | `src/MiniJira.IntegrationTests/` |
| Stop DB container | `docker compose down` | `src/MiniJiraAspire.Server/` |

API docs (dev only): `http://localhost:5413/swagger` or `/scalar`

---

## Architecture

Combines **Onion Architecture**, **CQRS**, **MediatR**, **Vertical Slices**, and **Repository Pattern**. See `docs/arc.md` for details.

Key rules:
- **Endpoints are thin** – they only call `mediator.Send(command)` and return the result
- **Features own their logic** – each feature in `Features/` has `Commands/` and `Queries/` subfolders with one handler per use case
- **Domain entities are intentionally anemic** – business rules live in handlers, not entities
- **DTOs cross the boundary** – entities never leave the server; handlers return `record` DTOs
- **Repositories are the DB adapter** – handlers depend on interfaces (`ITaskRepository`), not EF Core directly

```
HTTP Request
  → Endpoint (Endpoints/)
    → IMediator.Send(command/query)
      → Handler (Features/<Feature>/Commands or Queries)
        → IRepository (Persistence/Repositories/)
          → EF Core + PostgreSQL
```

---

## API Endpoints

| Group | Prefix | Operations |
|-------|--------|------------|
| Auth | `/api/auth` | Login, Register |
| Projects | `/api/projects` | CRUD |
| Tasks | `/api/tasks` | CRUD + `PATCH /{id}/status`, `PATCH /{id}/priority`, `PATCH /{id}/assign-user`, `PATCH /{id}/assign-epic` |
| Epics | `/api/epics` | CRUD |
| Comments | `/api/comments` | CRUD |
| Users | `/api/users` | List, role management |

Task query params: `?search=`, `?status=`, `?priority=`, `?assigneeId=`, `?epicId=`, `?projectId=`

---

## Tech Stack

**Backend**
- ASP.NET Core 10 Minimal APIs
- MediatR 14 (CQRS)
- Entity Framework Core 10 + Npgsql (PostgreSQL)
- JWT Bearer auth (`Microsoft.AspNetCore.Authentication.JwtBearer`)
- OpenTelemetry, Swagger/Scalar

**Frontend**
- React 19 + TypeScript + Vite 8
- React Router 7, TanStack Query 5, Jotai
- Tailwind CSS 4, Shadcn/ui (Radix UI), Zod
- Biome (formatter/linter)

**Infrastructure**
- .NET Aspire (local orchestration)
- PostgreSQL in Docker
- GitHub Actions: `tests.yml` (CI), `discord.yml` (notifications)

---

## Git Commit Guidelines

- One-line commit messages only — no body, no bullet points
- No `Co-Authored-By` trailer

---

## Coding Guidelines

- **Keep implementations small and focused** – each handler, component, or function does one thing. If it grows too large, split it.
- **Write smart, readable code** – use idiomatic patterns and well-named identifiers; avoid overly verbose solutions when a clean, concise one exists.
- **No code duplication** – before writing something, check if a handler, repository method, or component already does it. Extract shared logic rather than copy-pasting.
- **Follow existing patterns** – match the style of the surrounding code (e.g. if existing handlers use a certain error-handling approach, do the same).
- **No unused abstractions** – don't add helpers, base classes, or utilities unless they are actually reused. Three similar lines is better than a premature abstraction.
- **No speculative code** – only implement what is currently needed; no "maybe useful later" logic, flags, or fallbacks.

---

## Adding a New Feature

1. Create `Features/<FeatureName>/Commands/<ActionName>Command.cs` with the command record and handler
2. Create `Features/<FeatureName>/Queries/<QueryName>Query.cs` with the query record and handler
3. Create or update the repository interface in `Persistence/Repositories/I<Feature>Repository.cs`
4. Implement the repository method in the EF Core implementation
5. Register in `Program.cs` if needed (repositories are usually auto-registered)
6. Add a new endpoint group in `Endpoints/<FeatureName>/<Feature>Endpoints.cs` and map it in `Program.cs`
7. If schema changed: `dotnet ef migrations add <Name>` then `dotnet ef database update`
