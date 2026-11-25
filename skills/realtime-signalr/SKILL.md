---
name: realtime-signalr
description: >-
  Use this skill when adding or changing real-time push to clients with SignalR: the dedicated RealtimeHost, hub structure and message contracts, JWT/BFF authentication and per-group authorization, the outbox→broker→push pipeline, scale-out with the Redis/Valkey backplane (WebSockets-only with skipped negotiation, or sticky sessions), client reconnection, observability and limits. Triggers (RU): SignalR, realtime, хаб, пуш, вебсокет, группы, backplane, RealtimeHost.
---

# Real-Time Push (SignalR, RealtimeHost)

Real-time updates live in the dedicated `Itl.<Project>.RealtimeHost` (in `Hosts/`): one host owns all client connections, hub methods and the push pipeline. WebHost, BlazorHost and the SPA frontends never host hubs (a BFF host may only proxy the hub path, see [clients.md](./references/clients.md)). Every rule of `solution-architecture` applies to it as to any host: thin hub methods, mandatory host elements (`AddEnvironmentConfigurations`, `AddServiceDefaults`, `MapDefaultEndpoints`, validated options), its own Dockerfile, architecture tests.

## Choose the Client and Replica Profile

1. Find the client auth profile in the project decision, host registration and ingress. Read [clients.md](./references/clients.md#authentication-profiles). With direct OIDC, use the configured access-token factory and limit query-token acceptance to the hub path. With a BFF, use the authenticated same-origin proxy and test its origin restrictions and server-side token forwarding.
2. Find the intended RealtimeHost replica count and transport in the deployment settings. One replica needs no backplane. For several replicas, read [scale-out-and-resilience.md](./references/scale-out-and-resilience.md#multi-replica-rules), use the Redis/Valkey backplane and verify the configured transport: WebSockets with skipped negotiation for all clients, or session affinity when clients negotiate.
3. Read [hubs-and-contracts.md](./references/hubs-and-contracts.md). Trace the changed event from the committed state to the authorized group and the client's authoritative refetch. Record the client, transport and replica profile used. If the change depends on an undecided profile, resolve it before wiring the affected branch.

**Scope example** — A REST endpoint reads current state and history for the authenticated tenant. Keep that endpoint as the authoritative recovery path and use push as a trigger to reread the state. Verify delivery and reconnect through a hub fixture.

## Core Principles

1. **Delivery is a hint.** The push path has no durable replay to the browser. A broker retry can send the same logical event again if the first hub send succeeded before completion was recorded. Clients handle repeated hints idempotently and fetch authoritative state via the normal API (TanStack Query refetch / HTTP).
2. **Deny by default applies to hubs too.** The fallback authorization policy closes the hub; the hub carries its own policy. Joining a group is an authorization decision: the hub asks a Story whether the caller may follow the entity and adds the connection only on success. Identifiers that define access come from claims, never from client input.
3. **Push pipeline goes through the outbox.** A business change publishes its domain event through the transactional outbox (`message-brokers`); the RealtimeHost consumes it (`IProcessor`) and forwards a mapped Contracts DTO to a group or user. Hub methods never read the database directly and never publish to brokers.
4. **Contracts are typed.** Outbound JSON payloads are DTOs from `Itl.<Project>.Contracts/Realtime`. React generates types from schemas registered in OpenAPI. Blazor references Contracts directly. Hub method names, group names and payload versions are documented in `docs/contracts/realtime/`. In the gateway topology, proto (`Protos`) is used only between RealtimeHost and RpcHost.
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

## Completion Evidence

- In an isolated test with real hub connections, deny an unauthenticated connection and a valid caller's subscription to another owner's group. An authorized subscriber receives the expected payload from the changed event. Verify delivery separately from any polling or refetch fallback.
- Redeliver the same broker event after its completion record is durable and assert one hub send. Separately inject a failure after the hub send and before durable completion, then replay the event through the consumer. Deliver both hints with the same event identity. Assert that an expected visible action, such as a toast, occurs once and that the client shows the authoritative API state. Exercise both cases in the applicable processor and client tests.
- Force a new connection and change the authoritative state while it is offline. Assert that the client re-subscribes through authorization, refetches the missed state and receives a later event. Where access can change, verify that a revoked subscriber is denied on re-subscription.
- For multi-replica changes, connect through one instance and publish through another using the configured transport. Interrupt the backplane, assert the documented degraded health and authoritative API response, and verify delivery of a new event after recovery. A push missed during the outage can be recovered as current state through the API. [SignalR Redis backplane](https://learn.microsoft.com/en-us/aspnet/core/signalr/redis-backplane?view=aspnetcore-10.0#redis-server-errors)
- For a shared Redis/Valkey backplane, verify that the prefix is set and scoped to the deployment. Prove isolation with identical group names in two deployments. For a Blazor lifecycle change, fail or cancel the first connection and verify that a later explicit start succeeds without duplicate handlers.
- Record the test commands, auth and transport profiles, replica count and observed outcomes. Report any applicable check that could not run.
