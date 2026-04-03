# Testing

## Test Suite Overview

| Suite | Framework | Count | Location |
|-------|-----------|-------|---------|
| React unit | Jest + React Testing Library | ~61 tests | `src/app/reactjs/reminders-app/src/` (co-located `*.test.ts(x)`) |
| Blockchain | Hardhat + Chai | 10 tests | `blockchain/test/` |
| .NET unit | MSTest + Moq | ~20 tests | `src/test/server/dotnet/Reminders.Application.Test/` |
| .NET integration | MSTest | ~3 tests | `src/test/server/dotnet/Reminders.Api.Test/` (requires live API) |
| MVC Selenium | MSTest + Selenium | small | `src/test/app/dotnet/Reminders.Mvc.Test/` |
| E2E Cypress | Cypress | ~5 spec files | `src/test/cypress/` |

---

## React / Jest

**Run:**
```bash
cd src/app/reactjs/reminders-app
npm test                        # watch mode
npm test -- --coverage          # with coverage report
npm test -- --watchAll=false    # CI mode (single run)
```

**Config:** `jest.config.ts` — uses `nextJest`, `jsdom` env, `clearMocks: true`, `collectCoverage: true`

**Setup:** `jest.setup.js` — runs before each test file

**Alias mapping** (from `moduleNameMapper`):
- `@/app/*` → `src/app/*`

**Test files** follow `.test.ts` / `.test.tsx` naming convention and live next to the file they test.

**Shared mocks:** `src/app/util/testMocks.ts`

---

## Blockchain / Hardhat

**Run:**
```bash
cd blockchain
npm test
```

Tests use Hardhat's in-memory EVM. No external node needed.

---

## .NET Unit Tests (`Reminders.Application.Test`)

**Run:**
```bash
cd src
dotnet test src/test/server/dotnet/Reminders.Application.Test
```

**CI command** (with coverage, excluding infra):
```bash
dotnet test -c Release ./src/test/server/dotnet/Reminders.Application.Test \
  /p:CollectCoverage=true \
  /p:CoverletOutput=TestResults/ \
  /p:CoverletOutputFormat=lcov \
  /p:Exclude="[Reminders.Infrastructure.CrossCutting]*,[Reminders.Infrastructure.Data.EntityFramework]*" \
  /p:ExcludeByFile="**/*.Designer.cs,**/*Entity.cs"
```

**Pattern:** `[TestInitialize]` sets up Moq mocks. `GetRemindersService()` factory method returns service with mocked dependencies. Blockchain service mock always returns `"0x1234567890abcdef"` for transactions.

**Test class:** `ReminderServiceUnitTest` — covers Insert, Edit, Delete, Get, GetById scenarios including validation failures.

---

## .NET Integration Tests (`Reminders.Api.Test`)

**Requires live API on `http://localhost:5000`.**

```bash
# Start API first
docker compose up postgres ganache -d
cd src/server/api/dotnet/Reminders.Api
dotnet run

# Then run tests
cd src/test/server/dotnet/Reminders.Api.Test
dotnet test
```

`ApiIntegrationTest` runs a full CRUD flow in sequence: `Insert()` → `Edit()` → `Delete()` as a single `[TestMethod]`.

`EnsureDatabaseAvailableTests` and `MigrateRemindersDatabaseTests` verify DB availability and migration state.

---

## E2E Cypress

**Run:**
```bash
cd src/test/cypress
npm run cy:open         # interactive (Electron)
npm run cy:run          # headless

# Override base URL for local:
CYPRESS_baseUrl=http://localhost:3000 npm run cy:run
```

**Default target:** `https://jumperck.github.io/Reminders` (deployed GitHub Pages)

**Spec files:** `reminder-create.cy.js`, `reminder-delete.cy.js`, `reminder-edit.cy.js`, `reminders-list.cy.js`, `reminders-integration.cy.js`

**Fixtures:** `reminders.json`, `single-reminder.json`, `empty-reminders.json`

---

## GitHub Actions CI

Workflows in `.github/workflows/` — triggered on PRs to `main`, path-filtered:

| Workflow | Trigger path | What runs |
|----------|-------------|-----------|
| `dotnet-pull-request.yml` | `src/server/api/dotnet/**`, `src/app/dotnet/**`, `src/test/server/dotnet/**` | restore, build, unit tests with coverage |
| `react-pull-request.yml` | (React paths) | Jest unit tests |
| `go-pull-request.yml` | Go API paths | go build + test |
| `blockchain-pull-request.yml` | `blockchain/**` | Hardhat tests |
| `pull-request-check.yml` | Infrastructure / compose files | validation check |
| `cypress-e2e.yml` | (non-PR, deploy) | E2E against GitHub Pages |
| `dotnet-code-coverage.yml` | main branch | coverage + Coveralls upload |
| `deploy-pages.yml` | main branch | Next.js static export to GitHub Pages |

---

## Coverage

- React: ~99.6% (reported in README)
- .NET: tracked via Coveralls badge (linked from README)
- Excluded from .NET coverage: infrastructure/DI classes, EF Designer files, `Entity.cs`
