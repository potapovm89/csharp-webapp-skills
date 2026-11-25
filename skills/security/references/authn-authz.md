# Authentication & Authorization Reference

## Token Validation (JWT Bearer / Keycloak)
All of these stay `true` outside local development; never disable them to "make tests pass" — tests use the mocked authentication scheme instead.

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,
    ValidateAudience = true,          // audience = this API's client id
    ValidateLifetime = true,
    ValidateIssuerSigningKey = true,
    ClockSkew = TimeSpan.FromSeconds(30),
    ValidAlgorithms = [SecurityAlgorithms.RsaSha256]
};
options.RequireHttpsMetadata = !(builder.Environment.IsEnvironment("Local") || builder.Environment.IsEnvironment("Aspire"));
options.MapInboundClaims = false;     // keep original claim names ("sub", "realm_access")
```

- Signing keys come from the identity provider's metadata (Keycloak `.well-known`) or from secrets; never from source code.
- Keycloak protocol mappers treat dots in `claim.name` as JSON nesting: `http://schemas.microsoft.com/.../role` becomes a nested object, not a role claim. Use flat names (`roles`, Keycloak's realm `roles` scope) and let `JwtBearer` map them (`MapInboundClaims`) or set `RoleClaimType` explicitly; escape dots (`\.`) only when a dotted name is unavoidable. A test decodes a real token and checks the claim the API actually reads.
- Short-lived access tokens; refresh tokens are stored hashed and revocable.

## Policies and the Fallback Policy
```csharp
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build())
    .AddPolicy(AuthorizationPolicies.ADMIN, p => p.RequireRole(AuthenticationRoles.Admin))
    .AddPolicy(AuthorizationPolicies.STORE_OWNER, p => p.RequireClaim(AppClaims.STORE_ID));
```

- The fallback policy makes every endpoint (controllers, minimal APIs, gRPC) require authentication unless it has `[AllowAnonymous]`.
- Use named policies (`[Authorize(Policy = ...)]`), not role strings scattered over controllers. Policy and role names are constants.
- Health endpoints (`/health`, `/alive`) on the internal port are anonymous; they expose no details.

## Resource Ownership (prevents IDOR)
- The controller passes the **principal-derived** owner id into the command (read from a claim via a `ClaimsPrincipal` extension in `Common`), never `[FromHeader] string storeId` from an end-user request.
- A header with an owner id is acceptable only when it is set by a trusted gateway after authentication and the service is not reachable directly; document this in `docs/roles/`.
- The Story loads the entity filtered by owner (`WHERE store_id = @StoreId AND id = @Id`); a missing row returns 404.
- Admin endpoints that act on any owner are separate endpoints under the admin policy, not a flag in a user endpoint.

## Error Mapping
`AppExceptionHandler` / `AppExceptionInterceptor` map:

| Exception | HTTP | gRPC |
| --------- | ---- | ---- |
| not authenticated (`UnauthorizedAccessException`) | 401 | `Unauthenticated` |
| authenticated but not allowed (`ForbiddenException`) | 403 | `PermissionDenied` |
| `NotFoundException` (also for objects of another owner) | 404 | `NotFound` |

If `ForbiddenException` is missing in the solution, add it to `Common/Exceptions` with both mappings.

## Service-to-Service
- Use the enforcement boundary selected by the project policy and ADR. The reference profiles are application JWT with service client credentials and platform-enforced mTLS/workload authorization. A policy-approved exception follows the same checks described in [the security skill](../SKILL.md#changing-service-to-service-authorization).
- Internal gRPC/HTTP calls use TLS and authenticate the caller. Verify that direct access cannot bypass enforcement, caller/method permissions and forwarded identity or tenant delegation are protected, and rotation/revocation and negative access tests are evidenced. A namespace/private IP alone does not establish service identity.
- Each service has its own credentials for PostgreSQL, Kafka, RabbitMQ, S3 and Redis, with access only to its own databases, topics/vhosts and buckets.
- The migration runner uses a DB user that may change schema; the application user may only read/write data.
