# Metrics & Tracing Reference

## OpenTelemetry Wiring
Wired in all hosts via `builder.AddServiceDefaults()`:
- `AddAspNetCoreInstrumentation()`, `AddHttpClientInstrumentation()`, `AddRuntimeInstrumentation()`, `AddGrpcClientInstrumentation()`.
- Database: `.AddMeter("Npgsql", "Microsoft.EntityFrameworkCore")` for metrics and `.AddSource("Npgsql")` for SQL spans. Npgsql and EF Core emit them natively; without these lines they are silently dropped.
- Application meters and activity sources (outbox, consumers, workers) are registered by name in the same place: a `Meter`/`ActivitySource` that is not added with `AddMeter`/`AddSource` exports nothing.
- Quartz instrumentation in WorkerHost: `cfg.AddQuartzInstrumentation()`.

## Health Check Endpoints
Exposed via `host.MapDefaultEndpoints(port)`:
- `/health` — readiness check; `/alive` — liveness check.
- Bound to the dedicated internal management port from `ServicePurpose:Port` and marked `AllowAnonymous()` (the fallback authorization policy must not close them).

## Naming
- Follow OpenTelemetry semantic conventions: lowercase, dot-separated namespaces, snake_case inside a segment: `outbox.messages.oldest_pending_age`, `messaging.consumer.duration`.
- Units in the instrument (`s`, `{message}`, `By`), not in the name. Durations are histograms in seconds.
- Tags are low-cardinality: broker, destination, message type, outcome. Never ids, user data or error messages.
- One `Meter` per area, created through `IMeterFactory` in a singleton class (e.g. `OutboxMetrics`), with the meter name as a public constant used in `AddMeter`.

## Database Metrics (built-in)
| Source | What to watch |
| ------ | ------------- |
| Npgsql (`db.client.connection.*`, `db.client.operation.duration`) | Pool usage close to `Maximum Pool Size`, pending connection requests, slow commands |
| EF Core (`Microsoft.EntityFrameworkCore`) | Active contexts, query cache misses, optimistic concurrency failures |

## Messaging Metrics (application)
| Instrument | Type | Tags | Alert |
| ---------- | ---- | ---- | ----- |
| `outbox.messages.pending` | gauge | — | growing for minutes |
| `outbox.messages.oldest_pending_age` | gauge, s | — | > 5 min: relay stopped or broker unavailable |
| `outbox.messages.published` | counter | broker, message_type | — |
| `outbox.messages.failed` | counter | broker, message_type, outcome (`retry`/`exhausted`) | any `exhausted` |
| `outbox.publish.duration` | histogram, s | broker, message_type | p99 growth |
| `messaging.consumer.messages` | counter | destination, outcome (`processed`/`retried`/`dead_lettered`) | any `dead_lettered` |
| `messaging.consumer.duration` | histogram, s | destination, outcome | p99 growth |

Gauges that need a database query (backlog size, oldest age) are refreshed by a background loop at a fixed interval and only read from memory in the observable callback — never query the database inside the callback.

## Journey and Recovery Signals
- Implement the measurement boundary and outcome definitions from [reliability-and-performance](../../reliability-and-performance/SKILL.md). Use unsampled bounded-cardinality events/counters and threshold-capable histograms; record eligible work that times out or never completes, not only successful completions. Do not average per-pod percentages or percentiles.
- Missing/partial telemetry and zero/low traffic are explicit states. Preserve infrastructure/outbox/DLQ diagnostics alongside SLO burn-rate alerts; diagnostic thresholds are not themselves a business SLO. The example five-minute outbox threshold above is tuned to the project's processing deadline, not a universal target.
- Export recovery coverage/backup/archive health, overdue drills and key/storage availability status from the approved platform integration (`data-recovery`). No backup payloads, credentials or personal data are exported. Each actionable alert has an owner and runbook.
- Validate the actual backend's recording/alert rules and fixtures as described in [alerts-and-budgets.md](../../reliability-and-performance/references/alerts-and-budgets.md); do not claim a backend-compatible query from syntax alone.

## Custom Traces
- Create an `ActivitySource` for consumers and workers and register its name with `AddSource`:
  ```csharp
  private static readonly ActivitySource ActivitySource = new("Orders.Consumer");
  ```
- Consumer activities continue the producer's trace from the `traceparent` header (see `message-brokers`).
- Record failures with `activity?.AddException(ex)` and `activity?.SetStatus(ActivityStatusCode.Error)`; the OpenTelemetry `RecordException` extension is obsolete.
