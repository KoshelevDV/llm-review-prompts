# Example: PROJECT_CONTEXT for a .NET Blazor Project

This is a filled example of the `[PROJECT_CONTEXT]` slot for a fictional but realistic enterprise project.
Copy, adapt, and paste into any prompt from this collection.

---

```
## PROJECT_CONTEXT

### Project Overview
PortalHR is an internal HR management system for a mid-size enterprise (~2000 employees).
It handles employee lifecycle (onboarding/offboarding), leave management, performance reviews,
and payroll data synchronization with an external ERP (SAP S/4HANA).
Used by HR staff (~50 users) and employees (~2000, read-only self-service).

### Tech Stack
- .NET 8 (C# 12), ASP.NET Core 8
- Blazor Server (not WebAssembly) — all state server-side
- Entity Framework Core 8, PostgreSQL 16 (Npgsql provider)
- MediatR 12 (CQRS), FluentValidation 11, AutoMapper 12
- Polly 8 (resilience for SAP integration)
- Keycloak 24 (OIDC/OAuth2, integrated via Microsoft.Identity.Web)
- Serilog → Elasticsearch (structured logging)
- xUnit + Testcontainers (integration tests with real PG instance)

### Architecture
Clean Architecture with CQRS split:
- Domain/ — entities, value objects, domain events, no external dependencies
- Application/ — Commands, Queries, Validators, Handlers (business logic lives here only)
- Infrastructure/ — EF Core DbContext, repositories, SAP client, email service
- Web/ — Blazor pages, components, controllers (API endpoints for external integrations)

Layer dependency rule: Web → Application → Domain. Infrastructure implements Application interfaces.
Domain never references Application or Infrastructure.

### Key Rules

1. **No raw SQL** — all DB access via EF Core LINQ. Exceptions: reporting queries use stored procedures
   called via `ExecuteSqlRaw` with *parameterized* inputs only. No string interpolation in SQL ever.

2. **Controllers and Blazor pages are thin** — no business logic. Pages call MediatR commands/queries only.
   Logic in a page component = automatic BLOCKING in review.

3. **No `.Result` or `.Wait()` on async tasks** — always `await`. This causes deadlocks in Blazor Server's
   synchronization context. No exceptions.

4. **Use `Result<T>` for expected failures** — `Result<T>` from domain layer wraps success/failure.
   Never `throw` for expected errors (user not found, validation failed, etc.).
   Only throw for unexpected/infrastructure failures (DB unreachable, etc.).

5. **All public API endpoints must have `[Authorize]` or explicit `[AllowAnonymous]`** — no unannotated
   endpoints. Reviewed by security scanner in CI, but also flagged in code review.

6. **Repository pattern is mandatory** — no direct `DbContext` injection outside Infrastructure layer.
   Application layer only sees `IXxxRepository` interfaces.

7. **Sensitive fields must use `[PersonalData]` attribute** — GDPR requirement.
   Fields: name, email, phone, national ID, salary, address. Missed `[PersonalData]` = BLOCKING.

8. **SAP integration calls must use `IEmpServiceClient`, wrapped in Polly retry+circuit breaker.**
   Direct `HttpClient` usage for SAP = BLOCKING. The client handles retries, timeouts, and logging.

9. **No magic strings for claim types** — use `ClaimTypes.*` constants or project's `AppClaims` static class.

10. **Migrations must not contain data migrations** — schema changes only. Data migrations = separate scripts
    in `/scripts/migrations/` reviewed by DBA.

### Known Constraints / Pitfalls

- EF Core change tracking is **enabled by default**. For read-only queries (Queries in CQRS), always add
  `.AsNoTracking()`. Forgetting it causes memory bloat under load.

- Blazor Server uses a **single SignalR circuit per user**. Heavy synchronous operations block the UI thread.
  Use `InvokeAsync` and avoid `Thread.Sleep`. Background jobs go through `IBackgroundTaskQueue`.

- Keycloak token refresh is handled automatically by the middleware. Do NOT manually refresh tokens
  or check expiry in application code — this was a source of a production incident.

- `EmployeeId` is an `int` in the domain but a `string` UUID in Keycloak. The mapping lives in
  `KeycloakEmployeeMapper`. Don't assume they're the same type anywhere else.

- SAP sync is **eventually consistent** — HR data in PortalHR may lag SAP by up to 5 minutes.
  Do not write code that assumes real-time SAP consistency.

- The `Leave` bounded context and `Payroll` bounded context are **intentionally separate**.
  They communicate via domain events (`LeaveApprovedEvent`), never via direct service calls.
  Cross-context direct calls = BLOCKING architectural violation.

### Out of Scope for Code Review

- Blazor CSS/styling — handled by design system, not reviewed in MRs
- EF Core migration files — reviewed by DBA separately, not by this prompt
- `/tests/` directory SQL scripts — DBA-owned, excluded from developer review
- Third-party library internals referenced in diffs (Polly, MediatR source)
```
