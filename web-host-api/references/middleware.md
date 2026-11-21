# WebHost Middleware Pipeline Reference

## Pipeline Order
This is the single source of truth for the WebHost pipeline order; other skills link here. The sequence is strict:

```csharp
host.UseExceptionHandler();          // 1. AppExceptionHandler (RFC 9457 Problem Details, no stack traces)
host.UseForwardedHeaders();          // 2. Only behind a reverse proxy, with KnownProxies/KnownNetworks

if (!host.Environment.IsEnvironment("Local") && !host.Environment.IsEnvironment("Aspire"))
{
    host.UseHsts();                  // 3. HSTS outside local environments
}

host.UseSecurityHeaders();           // 4. Security headers
host.MapDefaultEndpoints(port);      // 5. Health & Alive checks on ServicePurpose:Port
host.UseRouting();                   // 6. Routing

if (!host.Environment.IsProduction())
{
    host.UseSwaggerConfiguration();  // 7. Swagger UI (non-production only)
}

host.UseHttpsRedirection();          // 8. HTTPS Redirection
host.UseCors(CorsPolicies.FRONTEND); // 9. CORS (between routing and auth; explicit origins from options)
host.UseAuthentication();            // 10. Authentication (Bearer / Keycloak)
host.UseAuthorization();             // 11. Authorization (fallback policy: authenticated user)
host.UseRateLimiter();               // 12. Rate limiting (after auth to partition by user)
host.MapControllers();               // 13. Controllers
```

Why the security-relevant positions matter (CORS, forwarded headers, rate limiting) is explained in `security`, web-hardening.md.

## Exception Handling
- Handled by `AppExceptionHandler` implementing `IExceptionHandler`.
- Emits RFC 9457 `ProblemDetails`:
  - `FluentValidation.ValidationException` -> 400 Bad Request with field errors.
  - `NotFoundException` -> 404 Not Found.
  - `UnauthorizedAccessException` -> 401 Unauthorized (not authenticated).
  - `ForbiddenException` -> 403 Forbidden (authenticated, not allowed). Add it to `Common/Exceptions` if missing.
  - Unhandled exceptions -> 500 Internal Server Error without leaking internal stack traces.

## Restful Filter: `SupportRestfulApiFilter`
Registered in `AddControllers(...)`:
- Action returning void/Task -> 204 No Content.
- Action returning null -> 404 Not Found.
