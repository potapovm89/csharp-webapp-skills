# Scale-Out and Resilience

## Multi-Replica Rules
A RealtimeHost with more than one replica requires **both**:

1. **Backplane** — fan-out of user/group messages between instances:

```csharp
builder.Services.AddSignalR().AddStackExchangeRedis(
    builder.Configuration.GetConnectionString("redisForRealtime")!,
    options => options.Configuration.ChannelPrefix = RedisChannel.Literal("realtime"));
```

2. **Connection pinning** — the backplane does not pin a connection to a server. Without negotiation pinning, the `/negotiate` request and the following connect request can land on different instances, and the second one fails. Pick one option and record it in the RealtimeHost settings doc (`docs/settings/<host>.md`, `project-documentation`):
   - **A. WebSockets only + skipped negotiation (default).** The server allows only WebSockets (`MapHub` option `Transports`), every client sets `skipNegotiation: true` / `SkipNegotiation = true` with the WebSockets transport. The connection is a single long-lived request, so no affinity is needed — this is the documented exception in the ASP.NET Core scaling guidance.
   - **B. Sticky sessions.** Required if any client needs negotiation (another transport, or a negotiate-based redirect). Configure LB session affinity (cookie) for the `/realtime` path; the management port and `/health` stay outside the affinity rules.

- A reconnect is a new connection (new negotiate under option B) and may land on any instance; that is fine because the client re-subscribes after every reconnect.
- One replica: no backplane needed, but the choice is written in the RealtimeHost settings doc in the same MR that adds the hub — the first replicated deployment must not silently break groups.
- The backplane is Redis/Valkey (`nosql-and-cache`): TTL rules do not apply to the backplane channel itself, but the connection string is validated on start (`configuration-secrets`).

## Rolling Deploys and Reconnects
Every deploy drops all connections — that is normal and must be invisible to the user:

- Clients enable automatic reconnect with bounded exponential backoff **with jitter** and **re-subscribe after reconnect** (group memberships die with the old connection; `clients.md`). When the retry policy gives up, the client shows the failure state and offers a manual retry.
- `terminationGracePeriodSeconds` covers connection drain; RealtimeHost has no long-running in-flight work besides sends — a grace period of 30–45 s is enough (`runtime-deployment`, graceful shutdown).
- A reconnect storm after a mass drop is bounded client-side (backoff + jitter) and server-side (`MaxConcurrentUpgradedConnections`, ingress rate limits on the hub path — `security`, web-hardening.md).
- After reconnect, the client treats the world as stale and refetches authoritative state (TanStack Query invalidation, `react-frontend`); pushes during the gap were lost by design ("delivery is a hint", SKILL.md principle 1).

## Proxies, Timeouts, Transports
- The ingress allows the `Upgrade`/`Connection` headers on the hub path and its read timeout exceeds the keep-alive: `KeepAliveInterval` (default 15 s) < ingress idle timeout; `ClientTimeoutInterval` (default 30 s) ≈ 2 × `KeepAliveInterval`, and the client's `serverTimeoutInMilliseconds` likewise. A proxy that silently kills idle sockets produces ghost connections — check its timeout config in the platform runbook.
- The hub path is exposed only through the ingress with authentication; the management port stays cluster-internal (`runtime-deployment`, kubernetes.md network policy: only ingress namespace + monitoring may reach it).
- Under option A browsers connect straight with a WebSocket and no CORS preflight happens. Under option B, if the SPA origin differs from the hub origin, the RealtimeHost needs a CORS policy for the negotiate request with explicit origins (`security`, web-hardening.md).

## Failure Modes
| Failure | Behavior |
| ------- | -------- |
| Backplane down | sends to users/groups can fail: the processor retry/DLQ path applies, and clients refetch through the REST API. `/health` reports `Degraded`. SignalR provides no durable replay to the browser after the outage. |
| Broker down | no new pushes; consumers recover via broker reconnection (`message-brokers`); outbox backlog alert stays authoritative (`observability`) |
| RealtimeHost down entirely | clients exhaust reconnects and show the failure state; data screens keep working through refetch — nothing is lost (`blazor-frontend` / `react-frontend` data states) |
| Poison push payload | straight to DLQ with alert (`message-brokers` consumer checklist), never sent raw to clients |

## Capacity
- Fan-out is multiplicative: one event to a `tenant_` group with N connected clients produces N sends. Estimate the worst case (all users of the largest tenant online) in the load test of the affected journey (`reliability-and-performance`).
- Scale on active connections (`signalr.server.active_connections`) and broker lag for the push consumers, not on CPU alone; connections per pod are the real limit (`observability`, metrics.md).
- Memory per connection is bounded by `MaximumReceiveMessageSize` and send buffers — validate with a soak test before raising limits, not after.
