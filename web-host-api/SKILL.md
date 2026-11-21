---
name: web-host-api
description: >-
  Use this skill when developing or modifying WebHost REST API endpoints, ASP.NET Core controllers, middleware pipeline, Swagger / OpenAPI configurations, or REST request/response conventions.
  Triggers (RU): REST, эндпоинт, контроллер, Swagger, middleware, версионирование API, ProblemDetails.
---

# WebHost REST API

This skill covers the design and implementation of the `Itl.<Project>.WebHost` REST API application.

## Key Principles
1. **Thin Controllers**: Controllers contain no business logic. They accept HTTP inputs, build a request for the solution's Stories mechanism (MediatR command/query or Executor story context — see `cqrs-stories`; or a gRPC request in the gateway topology — see `solution-architecture`), dispatch it and return the result.
2. **Kebab-Case URLs**: All endpoints use kebab-case via `[KebabCaseNaming]`.
3. **Partitioning**:
   - `Controllers/User` -> Route `api/v1/[controller]`, Swagger group `user-api`, JWT Bearer auth.
   - `Controllers/Admin` -> Route `api/v1/admin/[controller]`, Swagger group `admin-api`, Keycloak Admin role auth.
   - These prefixes apply to new versioned APIs. Preserve published legacy routes; migrate them only through the compatibility/deprecation process in api-evolution.md.
4. **Mandatory Cancellation**: Every action method takes `CancellationToken ct` as its last parameter.
5. **Strict Middleware Order**: `UseExceptionHandler` -> `UseForwardedHeaders` (behind a proxy) -> `UseHsts` (non-local) -> security headers -> `MapDefaultEndpoints` -> `UseRouting` -> `UseSwaggerConfiguration` (non-prod) -> `UseHttpsRedirection` -> `UseCors` -> `UseAuthentication` -> `UseAuthorization` -> `UseRateLimiter` -> `MapControllers`.
6. **Security**: deny by default (fallback policy), ownership checked in Stories with owner ids taken from claims, rate limits on anonymous and expensive endpoints. See the `security` skill.

## Program.cs Minimum (check on every WebHost change)
The template may not contain all of this yet. If it is missing, add it (or report it to the user) — do not assume the template is compliant.

```csharp
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build());
builder.Services.AddRateLimiter(o => { /* named policies, see security/web-hardening.md */ });
builder.Services.Configure<ForwardedHeadersOptions>(/* known proxies */);
```

- [ ] `SetFallbackPolicy(...RequireAuthenticatedUser())` — plain `AddAuthorization()` is not enough.
- [ ] `AddRateLimiter` + `UseRateLimiter`, policies on anonymous and expensive endpoints.
- [ ] `UseForwardedHeaders` (behind a proxy), `UseHsts` (non-local), security headers middleware.
- [ ] Middleware order exactly as in principle 5.

## Subtopic References
- [middleware.md](./references/middleware.md) — Middleware pipeline, RFC 9457 Problem Details error handling, and `SupportRestfulApiFilter`.
- [controllers.md](./references/controllers.md) — Controller partitioning, parameter attributes, and action templates.
- [swagger.md](./references/swagger.md) — OpenAPI configuration on the built-in `Microsoft.AspNetCore.OpenApi` (documents, security schemes, UI), Swashbuckle legacy rules.
- [api-evolution.md](./references/api-evolution.md) — compatible vs breaking changes, URL versioning, deprecation headers, idempotent writes, error and pagination contracts.
