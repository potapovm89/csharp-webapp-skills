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
Every RpcHost has an explicit authenticated/authorized caller boundary, including internal callers. Application JWT is the reference default below. An already enforced platform mTLS/workload policy can replace duplicate JWT per `security`, authn-authz.md; document/test the no-bypass and delegation guarantees. Example with JWT client credentials (adapt configuration names and the allowed service client to the solution):

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

Use validated options for these settings in the solution (`configuration-secrets`); reject blank/invalid authority and audience at startup. Verify the actual trusted client claim mapping and scopes in Keycloak; replace the example allowlist with the service principals permitted to call these methods. A valid token alone does not authorize arbitrary tenant delegation. With mTLS, configure certificate authentication and equivalent per-service/per-method policies instead.

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

`MapDefaultEndpoints` must mark only internal health endpoints `AllowAnonymous()`; reflection remains authenticated and is disabled in production. Register `AppExceptionInterceptor` and the remaining interceptors as described below. Test missing/expired/wrong-audience tokens and an authenticated but unauthorized service client.

## Subtopic References
- [protobuf.md](./references/protobuf.md) — Protobuf naming conventions, message structure, imports, wire compatibility rules (reserved fields, versioned packages), deadlines and retries.
- [interceptors.md](./references/interceptors.md) — Interceptor order, error handling (`AppExceptionInterceptor`), and metric interceptors.
