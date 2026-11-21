# Web Hardening Reference

## Pipeline Order (WebHost)
The full order is defined once in `web-host-api`, middleware.md. Security-relevant constraints:
- `UseExceptionHandler` first, so no exception escapes without the ProblemDetails mapping.
- `UseForwardedHeaders` before anything that reads the scheme or client IP (HSTS, HTTPS redirection, rate limiting by IP), and only with trusted proxies configured.
- `UseCors` between `UseRouting` and `UseAuthentication`, so preflight requests are answered without a token.
- `UseRateLimiter` after `UseAuthorization`, so limits can be partitioned by user.

## CORS
- Allowed origins come from options (`CorsOptions.AllowedOrigins`), per environment.
- Never `AllowAnyOrigin()`; never combine wildcard origins with `AllowCredentials()`.
- Allow only the methods and headers the frontend uses.

```csharp
builder.Services.AddCors(o => o.AddPolicy(CorsPolicies.FRONTEND, p => p
    .WithOrigins(corsOptions.AllowedOrigins)
    .WithMethods("GET", "POST", "PUT", "DELETE")
    .WithHeaders("Authorization", "Content-Type", "Idempotency-Key")));
```

## Rate Limiting
Mandatory for anonymous endpoints (login, registration, password reset) and expensive ones (PDF/report generation, search).

```csharp
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.AddPolicy(RateLimitPolicies.PER_USER, context => RateLimitPartition.GetFixedWindowLimiter(
        context.User.FindFirst("sub")?.Value ?? context.Connection.RemoteIpAddress?.ToString() ?? "anonymous",
        _ => new FixedWindowRateLimiterOptions { PermitLimit = 100, Window = TimeSpan.FromMinutes(1) }));
});

[EnableRateLimiting(RateLimitPolicies.PER_USER)]
public class PaymentController : ControllerBase { ... }
```

Limits are options, not constants. Behind a proxy, partition by the real client IP only after `UseForwardedHeaders` is configured with trusted proxies.

## Security Headers
A small middleware in `Common/Api/Middlewares` adds to every response:

| Header | Value |
| ------ | ----- |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` (or CSP `frame-ancestors 'none'`) |
| `Referrer-Policy` | `no-referrer` |
| `Content-Security-Policy` | for HTML-serving hosts (Blazor/Razor): restrict `default-src 'self'` |
| `Cache-Control` | `no-store` for responses with personal data or tokens |

## Request Limits
- Keep Kestrel's default body limit or lower it; raise it only on specific upload endpoints with `[RequestSizeLimit]`.
- Set `[RequestFormLimits]` for multipart uploads.
- Paging endpoints cap the page size (e.g. ≤ 100) in the validator.

## Error Responses
- RFC 9457 Problem Details; in production no exception type, message of unexpected exceptions, stack trace or SQL.
- Validation errors may return field names and messages; they must not echo secrets or full input values.
- Swagger UI and gRPC reflection are disabled in production.
