# Patterns

## .NET API — Error Handling

All exceptions bubble up to `UseRemindersExceptionHandler` (global middleware in `Program.cs`):

1. **`RemindersApplicationException`** (business errors) → mapped via `ToHttpStatusCode()` extension:
   - `ValidationStatus.NotFound` → HTTP 404
   - `ValidationStatus.IdsDoNotMatch` → HTTP 409
   - Other → HTTP 400

2. **`FluentValidation.ValidationException`** → HTTP 422, with `properties` dict of field errors

3. Everything else → HTTP 500 `"Internal Server Error."`

Error response shape (`ErrorDetails`):
```json
{ "statusCode": 404, "message": "Not found.", "properties": null }
```

## .NET API — Soft Delete

Deletion is **not a physical DELETE**. `Entity<TId>.Delete()` sets `IsDeleted = true`, then `remindersRepository.Update()` is called. All `Get()` queries filter `!reminder.IsDeleted`. The `Exists()` method also returns `false` for soft-deleted records.

## .NET API — Validation Pattern

FluentValidation with `RuleSet("Insert", ...)`:
- `Insert` rule set: `IsDone` must be `false`
- Default rules (always): `LimitDate.Date > DateTime.UtcNow.Date`

`Insert` path calls `validator.Validate(..., options => options.IncludeRuleSets("*"))` — applies all rule sets.
`Edit` path calls `ValidateAndThrow()` — applies only default rules (no rule set filtering).

**Important**: `LimitDate` must be strictly *after* today, not equal to today.

## .NET API — Service Layer Conventions

- `IsDone` is always forced to `false` on insert (caller cannot set it).
- `Id` is always forced to `Guid.Empty` on insert (prevents ID spoofing).
- Edit validates that the URL `id` param matches the body `ReminderViewModel.Id` — mismatch throws `IdsDoNotMatch`.
- Blockchain calls use `.Result` (blocking async) — intentional but introduces risk of deadlocks in some contexts.
- Every mutating operation calls `unitOfWork.Commit()` after the repository call.

## .NET API — Layered Namespace Pattern

Namespace prefixes: `Reminders.Domain.*`, `Reminders.Application.*`, `Reminders.Infrastructure.*`
- `GlobalUsings.cs` imports common namespaces project-wide.
- `[ExcludeFromCodeCoverage]` used on DI wiring classes.

## Go API — Error Handling

Pattern: check error → return JSON error response immediately (no global middleware).
```go
if errors.Is(err, ErrorReminderNotFound) {
    c.IndentedJSON(http.StatusNotFound, gin.H{"message": "..."})
    return
}
```

## C++ API — Error Handling

All handlers wrapped in `try/catch (const std::exception& e)` → `errorResponse(500, ...)`. JSON parse errors caught separately with `catch (const json::parse_error&)` → 400.

## React — State Management

Two-layer state:
1. **TanStack Query** — server state (cache, fetching). Wrapped in `ReminderQueryProvider`.
2. **React Context + useReducer** — form state. `RemindersContext` via `RemindersContextProvider`.

Reducer actions: `SET_REMINDER`, `UPDATE_REMINDER`, `CLEAR_REMINDER`.

`useContextSelector` (from `use-context-selector`) is used instead of `useContext` to avoid unnecessary re-renders.

## React — Validation Pattern

Double validation: client-side first (`ValidationService`), then server-side errors from API response.

`onCreateReminder`/`onUpdateReminder` both:
1. Run `ValidationService.validateTitle/Description/LimitDate`
2. If errors → set local errors state, return `Fail`
3. Else → call `mutateAsync`, check `result.errors`
4. Return `Success` | `Fail`

## React — API Layer

`fetch` is used directly (no axios). Mutations return `MutateResult<T>` — either `result` (success) or `errors` (failure). Never throws on non-2xx; callers check `result.errors` presence.

## React — Reminder Display Transforms

`mapReminder()` in `src/app/api/index.ts` adds two computed fields:
- `limitDateFormatted` — UTC date formatted as `YYYY-MM-DD`
- `isDoneFormatted` — `"Yes"` | `"No"`

## Nginx — Load Balancing

Round-robin across all three API backends. No sticky sessions. The `X-Server` response header (`dotnet` / `go` / `cpp`) identifies which backend answered. This matters for debugging inconsistencies.

## Docker Compose — Shared Resources

YAML anchors used for shared config:
- `&shared-deploy` / `*shared-deploy` — CPU/memory limits (0.25 cpu, 0.5GB by default; overridden in override.yml for dev)
- `&dotnet-args` / `*dotnet-args` — build args for .NET containers
- `&dotnet-volumes` / `*dotnet-volumes` — shared volume mounts
