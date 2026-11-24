---
name: realtime-signalr
description: >-
  Use this skill when adding or changing real-time push to clients with SignalR: the dedicated RealtimeHost, hub structure and message contracts, JWT/BFF authentication and per-group authorization, the outbox→broker→push pipeline, scale-out with the Redis/Valkey backplane (WebSockets-only with skipped negotiation, or sticky sessions), client reconnection, observability and limits. Triggers (RU): SignalR, realtime, хаб, пуш, вебсокет, группы, backplane, RealtimeHost.
---

# Real-Time Push (SignalR, RealtimeHost)

Real-time updates live in the dedicated `Itl.<Project>.RealtimeHost` (in `Hosts/`): one host owns all client connections, hub methods and the push pipeline. WebHost, BlazorHost and the SPA frontends never host hubs (a BFF host may only proxy the hub path, see [clients.md](./references/clients.md)). Every rule of `solution-architecture` applies to it as to any host: thin hub methods, mandatory host elements (`AddEnvironmentConfigurations`, `AddServiceDefaults`, `MapDefaultEndpoints`, validated options), its own Dockerfile, architecture tests.

## Core Principles

1. **Delivery is a hint, not a guarantee.** A SignalR push is at-most-once to a connected client. Authoritative state is always fetched via the normal API (TanStack Query refetch / HTTP); the push only triggers or accelerates it. No business rule may depend on a client having received a message.
2. **Deny by default applies to hubs too.** The fallback authorization policy closes the hub; the hub carries its own policy. Joining a group is an authorization decision: the hub asks a Story whether the caller may follow the entity and adds the connection only on success. Identifiers that define access come from claims, never from client input.
3. **Push pipeline goes through the outbox.** A business change publishes its domain event through the transactional outbox (`message-brokers`); the RealtimeHost consumes it (`IProcessor`) and forwards a mapped Contracts DTO to a group or user. Hub methods never read the database directly and never publish to brokers.
4. **Contracts are typed and generated.** Outbound payloads are DTOs from `Itl.<Project>.Contracts/Realtime`; browsers always receive JSON. Their schemas are included in the OpenAPI document the frontend generates its client from (see [hubs-and-contracts.md](./references/hubs-and-contracts.md)). Hub method names, group names and payload versions are documented in `docs/contracts/realtime/`. In the gateway topology, proto (`Protos`) is used only between RealtimeHost and RpcHost.
5. **Scale-out is explicit.** More than one replica requires the Redis/Valkey backplane **and** a connection-pinning answer: WebSockets-only clients with skipped negotiation (default) or sticky sessions at the ingress. See [scale-out-and-resilience.md](./references/scale-out-and-resilience.md).

## Program.cs Minimum (check on every RealtimeHost change)

```csharp
builder.Services.AddSignalR(options =>
{
    options.MaximumReceiveMessageSize = 16 * 1024;   // lower than the 32 KB default: clients send ids, files go through REST + S3
    options.MaximumParallelInvocationsPerClient = 1; // the default; do not raise without a load test
    options.EnableDetailedErrors = false;            // never leak internals to clients
});

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        // ... standard validation (grpc-services shows the full block) ...
        // Browsers cannot set an Authorization header on a WebSocket: accept the short-lived
        // access token from the query string, only on the hub path.
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];
                if (!string.IsNullOrEmpty(accessToken)
                    && context.HttpContext.Request.Path.StartsWithSegments("/realtime"))
                {
                    context.Token = accessToken;
                }
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build());

builder.WebHost.ConfigureKestrel(k => k.Limits.MaxConcurrentUpgradedConnections = realtimeOptions.MaxConnectionsPerInstance);

host.MapDefaultEndpoints(port);
host.UseRouting();
host.UseAuthentication();
host.UseAuthorization();
host.MapHub<OrdersHub>("/realtime/orders", o =>
{
    o.Transports = HttpTransportType.WebSockets;   // no long polling / SSE: stable behind load balancers
    o.CloseOnAuthenticationExpiration = true;      // the token is checked at connect; close when it expires
}).RequireAuthorization("OrdersRealtime");
```

- The token ends up in the URL of the WebSocket handshake: short-lived tokens only, no query-string logging anywhere (access logs included), and the path check stays narrow (`security`, authn-authz.md).
- With `CloseOnAuthenticationExpiration` the client reconnects with a fresh token from its token factory; the reconnect re-subscribes like any other ([clients.md](./references/clients.md)).
- Probes: `/health` includes the backplane connection (reported `Degraded`, not `Unhealthy`, see the failure modes); `/alive` stays process-only.

## Subtopic References
- [hubs-and-contracts.md](./references/hubs-and-contracts.md) — hub structure, user and group addressing, authorization of subscriptions, contracts and their generation, the outbox→broker→push pipeline, limits, metrics.
- [scale-out-and-resilience.md](./references/scale-out-and-resilience.md) — backplane and connection pinning, rolling deploys, reconnect storms, failure modes, capacity.
- [clients.md](./references/clients.md) — React and Blazor (BFF and direct OIDC) clients: connection lifecycle, token handling, re-subscription, wiring pushes into query/state, testing.
