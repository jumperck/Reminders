# Gotchas

## Blockchain: chainId is hardcoded to 0

In `RemindersService.cs`, every blockchain read/write uses `chainId = 0`. This is marked with `// TODO`. All reminder mutations use the *same* blockchain slot. There is no per-reminder blockchain ID tracking — the blockchain integration is supplementary logging, not a source of truth.

## Blockchain: `.Result` on async — blocking calls

`RemindersBlockchainService` methods are `async Task<T>`, but `RemindersService` calls them with `.Result`. This is a synchronous block on an async operation. It works in this context (not an ASP.NET synchronization context issue with .NET 8 Kestrel) but is a code smell and can cause deadlocks in other hosting scenarios.

## Delete is soft, not hard

`DELETE /api/reminders/{id}` does **not** remove the row. It sets `IsDeleted = true` and calls `Update()`. If you query the raw database table, deleted reminders will still appear. The `Get()` methods filter `!IsDeleted`, and `Exists()` returns `false` for soft-deleted records. The Go and C++ APIs perform **physical deletes** — behavior diverges between backends.

## Go/C++ APIs don't integrate blockchain

Only the .NET API writes to Ganache. When nginx round-robins to Go or C++ backends, no blockchain event is emitted. The three APIs are not functionally equivalent.

## SQL Server migration warnings are expected

Both Postgres and SQL Server migration classes exist. When running with Postgres (default), the runner logs SQL Server migration failures — this is expected and harmless per README. The runner filters by namespace (`.Postgres.` or `.SqlServer.`) before applying.

## `NEXT_PUBLIC_API_BASE_URL` is baked in at build time

The React app is exported statically for GitHub Pages. `NEXT_PUBLIC_API_BASE_URL` is passed as a Docker build arg. For local dev, it defaults to `http://localhost:9999`. To run against a different API, you must rebuild the Docker image.

## `LimitDate` must be strictly after today

The validator checks `LimitDate.Date > DateTime.UtcNow.Date` — today's date fails validation. Reminder list tests use `.AddDays(1)` minimum.

## Nginx has no health-check waiting for APIs

In `docker-compose.yml`, nginx `depends_on` the 3 API containers but only via `service_started` (not `service_healthy`). APIs may not be fully up when nginx starts receiving traffic. In practice, Docker's container start ordering is usually fast enough, but integration tests or quick smoke tests post-startup may see transient failures.

## Integration tests require a running API on port 5000

`Reminders.Api.Test` hits `http://localhost:5000` directly. Running `dotnet test` on this project without a live API will fail every test. These are not self-contained tests.

## Selenium tests require browser drivers

`src/test/app/dotnet/Reminders.Mvc.Test/Drivers/` includes Windows `.exe` drivers (chromedriver, geckodriver). These tests are effectively Windows-only and are skipped in CI.

## The MVC app doesn't run APIs — it calls nginx

MVC `ApiOptions__BaseUrl` points to `http://reminders-nginx:9999`. If you run only `--profile mvc`, the nginx/API stack is not started, so MVC will fail to load reminders. Use `--profile all` or ensure API stack is up.

## `Exists()` detaches entity from EF context

`Repository.Exists()` calls `GetAsNoTracking()` which explicitly detaches the entity. If you then try to update an entity that was checked via `Exists()`, you must re-fetch it. This is done correctly in `Delete()` — it calls `remindersRepository.Get(id)` (tracking) after `Exists()`.

## React's `clearReminder()` must be called before navigation

The list page calls `clearReminder()` before pushing to create/edit routes. If skipped, the previous form state bleeds into the new page because `RemindersContextProvider` wraps the whole app.

## `POSTGRES_USER` is `root` (not `postgres`)

The default Postgres superuser in this project is `root` (set via `POSTGRES_USER: root`). Connection strings use `Username=root`. Don't expect the standard `postgres` user to work.

## Ganache mnemonic is static

The fixed mnemonic `"candy maple cake sugar pudding cream honey rich smooth crumble sweet treat"` produces deterministic addresses. `Blockchain__PrivateKey` in `docker-compose.override.yml` is the private key of account 0 from this mnemonic — it's a well-known dev key, never use in production.

## `.NET` solution is at `src/Reminders.sln`

`dotnet restore` and `dotnet build` must target `./src`, not the repo root. The sln is at `src/Reminders.sln`.
