# Modules

## Root Layout

```
src/
  Reminders.sln              — single solution for all .NET projects
  app/                       — frontend apps
  server/                    — backend APIs and services
  test/                      — all tests
blockchain/                  — Solidity contracts + Hardhat toolchain
infrastructure/              — nginx configs, k6 load tests, mssql scripts
```

---

## Backend: .NET API — `src/server/api/dotnet/Reminders.Api/`

Follows a layered DDD architecture via `Layers/` subdirectory:

| Layer | Path | Purpose |
|-------|------|---------|
| **Domain** | `Layers/Domain/` | `Reminder` entity, `Entity<TId>` base, repository/UoW interfaces |
| **Application** | `Layers/Application/` | `RemindersService`, `RemindersBlockchainService`, FluentValidation validators, AutoMapper profiles, ViewModels |
| **Data** | `Layers/Data/EntityFramework/` | `RemindersContext`, `Repository<T>`, `UnitOfWork`, provider-specific migrations (Postgres/SqlServer) |
| **CrossCutting** | `Layers/CrossCutting/` | DI wiring, `MachineNameMiddlewareExtensions` |

Key files:
- `Controllers/RemindersController.cs` — thin controller, all logic in service
- `Layers/Application/Services/RemindersService.cs` — CRUD orchestration + blockchain side-effects
- `Layers/Application/Services/RemindersBlockchainService.cs` — Nethereum wrapper for Ganache
- `Layers/Data/EntityFramework/Repository.cs` — generic base repository (soft-delete aware)
- `Program.cs` — startup, CORS, Swagger, `X-Server: dotnet` header

---

## Backend: Go API — `src/server/api/go/reminders-api/`

Simple 3-file structure:
- `cmd/app/main.go` — gin router setup, CORS, `X-Server: go` header, health endpoint
- `pkg/api/handler.go` — HTTP handlers
- `pkg/api/repository.go` — direct SQL against Postgres (`lib/pq`)
- `pkg/models/reminder.go` — Reminder struct

No ORM, no DI, no validation layer. Direct SQL queries.

---

## Backend: C++ API — `src/server/api/cpp/reminders-api/src/`

Uses **Crow** framework + **nlohmann/json** + libpq:
- `main.cpp` — Crow app setup, route registration
- `handler.cpp/h` — HTTP handlers (try/catch for all operations)
- `repository.cpp/h` — PostgresRepository with libpq
- `model.h` — Reminder struct with nlohmann JSON serialization

Does not integrate blockchain.

---

## Migration Runner — `src/server/services/dotnet/Reminders.MigrationsRunner/`

One-shot .NET console app that:
1. Applies provider-specific EF Core migrations (Postgres or SqlServer)
2. Exposes `/healthz` endpoint
3. Uses Polly retry (exponential backoff, configurable via `MigrationRunner__MaxRetryAttempts`)
4. Exits 0 on success, exits 1 on failure

---

## Frontend: React App — `src/app/reactjs/reminders-app/`

Next.js 14 app (App Router), TypeScript, Material UI:

| Path | Purpose |
|------|---------|
| `src/app/api/index.ts` | Raw fetch functions (getReminders, createReminder, etc.) |
| `src/app/api/hooks/index.ts` | TanStack Query hooks (`useReminders`, `useCreateReminder`, etc.) |
| `src/app/api/types/index.ts` | `Reminder`, `APIError`, `Errors`, `MutateResult<T>` |
| `src/app/hooks/useReminderContext/` | React context + `useReducer` for form state + action handlers |
| `src/app/hooks/useRemindersQueryClient/` | TanStack `QueryClientProvider` wrapper |
| `src/app/services/ValidationService.ts` | Client-side field validation (title, description, limitDate) |
| `src/app/reminder/[id]/` | Edit page (dynamic route) |
| `src/app/reminder/create/` | Create page |
| `src/app/reminder/list/` | List page |
| `src/app/components/` | `AlertError`, `ReminderDeleteModal`, `ReminderForm` |

API base URL comes from `NEXT_PUBLIC_API_BASE_URL` env var (build-time baked in for static export).

---

## Frontend: .NET MVC — `src/app/dotnet/Reminders.Mvc/`

ASP.NET Core 8 MVC app. Calls the API via `RemindersService` (HTTP client). Views use Razor + Bootstrap.

---

## Blockchain — `blockchain/`

Hardhat project (TypeScript):
- `contracts/Reminders.sol` — mapping-based reminder storage, owner validation
- `contracts/Lock.sol` — standard Hardhat demo contract
- `ignition/modules/` — Hardhat Ignition deployment modules
- `scripts/` — utility scripts (deploy, create, get, count, monitor)
- `test/` — Chai/Hardhat tests for both contracts

---

## Tests — `src/test/`

| Path | Framework | Type |
|------|-----------|------|
| `server/dotnet/Reminders.Application.Test/` | MSTest + Moq | Unit tests for RemindersService |
| `server/dotnet/Reminders.Api.Test/` | MSTest | Integration tests (requires live API) |
| `app/dotnet/Reminders.Mvc.Test/` | MSTest + Selenium | MVC UI tests (requires Chrome/Firefox) |
| `cypress/` | Cypress | E2E against deployed GitHub Pages app |
| React `*.test.ts(x)` files | Jest + RTL | Component/hook/API layer unit tests |
