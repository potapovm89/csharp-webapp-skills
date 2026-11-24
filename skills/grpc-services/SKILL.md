---
name: grpc-services
description: >-
  Use this skill when developing or modifying gRPC services in RpcHost, defining .proto contracts and protobuf messages in Protos, configuring gRPC interceptors, or mapping gRPC error status codes.
  Triggers (RU): gRPC, proto, RpcHost, интерцептор, совместимость контрактов, дедлайны.
---

# gRPC Services & Protobuf (RpcHost)

This skill covers implementing gRPC endpoints in `Itl.<Project>.RpcHost` and defining contracts in `Itl.<Project>.Protos`.

## Architecture
- Contracts live in `Protos` and are compiled to C# via `Grpc.Tools`.
- `RpcHost` implements generated `*.ServiceBase` classes.
- Services are thin adapters dispatching through the solution's chosen mechanism: `IStoryExecutor` or MediatR `ISender` (see `cqrs-stories`).
- Pass `context.CancellationToken` as the execution method's token argument and to all async calls; do not put it in the StoryContext. Confirm the current Executor interface before changing an older solution.

```csharp
public class OrderRpcService(IStoryExecutor storyExecutor) 
    : OrderService.OrderServiceBase
{
    public override async Task<CreateOrderResponse> CreateOrder(
        CreateOrderRequest request, 
        ServerCallContext context)
    {
        var result = await storyExecutor.Execute(new CreateOrderStoryContext
        {
            Request = request
        }, context.CancellationToken);

        return new CreateOrderResponse { OrderId = (UUIDModel)result };
    }
}
```

For MediatR, map the proto request to a Contracts DTO, put it in the existing command and call `await sender.Send(command, context.CancellationToken)`. Keep proto types out of MediatR commands. Ownership identifiers come from the authenticated caller as described in `security`, not directly from an untrusted request.

## Authentication and Program.cs Pipeline

Read the project's approved policy, ADR, host registration and effective deployment boundary before editing. Use the [service-to-service rules](../security/SKILL.md#changing-service-to-service-authorization) to select the applicable branch:

| Approved profile | What to wire or preserve |
|---|---|
| Application JWT with service client credentials | Use the registration example below and the project's issuer, audience and caller/method policies. |
| mTLS or workload authorization | Use the approved application or platform boundary. Verify caller identity, method permissions, protected delegation and denied direct access. Wire only the controls required by that profile. |
| Scoped policy-approved exception | Keep the mechanism, scope and method policy recorded in the ADR. Extend the existing enforcement for the changed method and verify equivalent controls using the actual configuration. |

If the policy, ADR and configuration disagree, resolve the affected boundary before editing its registration. A new method in an established host uses its existing approved profile.

**Example from `it-letter`** — ADR 0001 selects service tokens for the gateway. `RpcHost` registers `RpcServiceAuthInterceptor` outside `AppExceptionInterceptor`. When adding a method under this decision, update and verify the applicable caller/method policy without adding a second JWT profile. Use the integration test pattern that exercises missing credentials and denied methods on the configured host.

### Application JWT Example

```csharp
var rpcAuth = builder.Configuration.GetRequiredSection("RpcAuthentication");
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = rpcAuth["Authority"]
            ?? throw new InvalidOperationException("RpcAuthentication:Authority is required");
        options.Audience = rpcAuth["Audience"]
            ?? throw new InvalidOperationException("RpcAuthentication:Audience is required");
        options.RequireHttpsMetadata = !(builder.Environment.IsEnvironment("Local")
            || builder.Environment.IsEnvironment("Aspire"));
        options.MapInboundClaims = false;
        options.TokenValidationParameters.ValidateIssuer = true;
        options.TokenValidationParameters.ValidateAudience = true;
        options.TokenValidationParameters.ValidateLifetime = true;
        options.TokenValidationParameters.ValidateIssuerSigningKey = true;
    });

builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build())
    .AddPolicy("OrdersRpcCaller", policy => policy
        .RequireAuthenticatedUser()
        .RequireClaim("azp", "orders-webhost")); // Example Keycloak service client allowlist
```

Use validated options for the JWT settings (`configuration-secrets`). Reject blank or invalid authority and audience at startup. Verify the trusted client claim mapping and scopes in Keycloak and use the actual service principals permitted to call these methods. Check tenant delegation separately from token validity.

```csharp
host.MapDefaultEndpoints(port);
host.UseRouting();
host.UseHttpsRedirection();
host.UseAuthentication();
host.UseAuthorization();

if (!host.Environment.IsProduction())
{
    host.MapGrpcReflectionService(); // Reflection enabled in non-production
}

host.MapGrpcService<OrderRpcService>().RequireAuthorization("OrdersRpcCaller");
```

`MapDefaultEndpoints` must mark only internal health endpoints `AllowAnonymous()`. Protect reflection under the selected profile and keep it disabled in production. Register `AppExceptionInterceptor` and the remaining interceptors as described below.

## Subtopic References
- [protobuf.md](./references/protobuf.md) — Protobuf naming conventions, message structure, imports, wire compatibility rules (reserved fields, versioned packages), deadlines and retries.
- [interceptors.md](./references/interceptors.md) — Interceptor order, error handling (`AppExceptionInterceptor`), and metric interceptors.

## Completion Evidence

- Record the approved profile, source of the decision and actual method enforcement. For a changed contract, generate the client and verify compatibility with the previous contract where applicable.
- Through the configured boundary, verify that an allowed caller reaches the changed method and a valid caller without method permission receives the documented denial. Missing or invalid identity must be rejected before the Story runs. At the application gRPC boundary expect `Unauthenticated` or `PermissionDenied` as applicable. At a transport-enforced boundary, assert the expected transport rejection and absence of handler invocation.
- For JWT, include expired and wrong-audience tokens. For a platform boundary, test direct access without enforcement. For a scoped exception, use that profile's actual test credentials and assert the configured method restrictions. Verify tenant delegation when present, and record the commands, results and unavailable checks.
