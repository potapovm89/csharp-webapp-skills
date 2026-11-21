# Logging Reference

## Core Rules
- Always use DI `ILogger<T>`. Never use `Console.WriteLine`.
- Configured via Serilog using shared files in `shared/configs/serilog.{env}.json`.
- Log messages must be in **English**.
- Always use **structured logging** with named placeholders; never use string interpolation:
  ```csharp
  // Correct
  _logger.LogInformation("Processed order {OrderId} for customer {CustomerId}", orderId, customerId);

  // Prohibited
  _logger.LogInformation($"Processed order {orderId}");
  ```
- Always pass exceptions as the first argument:
  ```csharp
  _logger.LogError(ex, "Failed to process order {OrderId}", orderId);
  ```
- Strictly avoid logging passwords, auth tokens, personal data, or credit card info.

## Serilog Configuration
A host registers the shared config files (`AddEnvironmentConfigurations`) and runs Serilog from `serilog.{env}.json`. Baseline for deployed environments:

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
      }
    },
    "Enrich": ["FromLogContext"],
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "formatter": "Serilog.Templates.ExpressionTemplate, Serilog.Expressions",
          "template": "{@t:yyyy-MM-dd HH:mm:ss.fff} [{@l:u3}] [{TraceId}] {SourceContext}: {@m}\n{@x}"
        }
      },
      {
        "Name": "OpenTelemetry",
        "Args": { "endpoint": "http://localhost:4317", "protocol": "Grpc" }
      }
    ]
  }
}
```

- Local (`Local`, `Aspire`) may use `Debug`/`Information` on the console sink; deployed environments run `Information` with framework overrides above. Per-environment differences live in `serilog.{env}.json`, never in code.
- Machine-collected output must be structured: the OpenTelemetry sink exports logs via OTLP with `TraceId`/`SpanId` attached. Plain-text files are not a telemetry backend in deployed environments.
- EF Core command logging (`Microsoft.EntityFrameworkCore.Database.Command`) is `Warning` in deployed environments, `Information` only locally — query text contains personal data.

## Levels
| Level | Use |
| ----- | --- |
| Debug | local diagnosis only; production behavior must not depend on it |
| Information | business events: order created, job started/finished with counts |
| Warning | recoverable anomalies: retry, fallback, degraded optional dependency, message bound for DLQ |
| Error | operation failed and requires attention; always with the exception object |
| Critical | process cannot continue |

- Log once, at the boundary that makes the decision; rethrow without wrapping so a single failure produces a single entry. Log-and-return in a processor is prohibited (`message-brokers`).
- Do not use levels to sort logs for humans ("just to see it higher") — the level is a filtering contract for dashboards and alerts.

## Correlation (log ↔ trace)
- Log inside the active activity (request, consumer message, job execution): Serilog + the OpenTelemetry sink attach `TraceId`/`SpanId` automatically; the `[TraceId]` template property makes it visible locally.
- `builder.AddServiceDefaults()` wires ASP.NET Core, HttpClient and gRPC instrumentation. A log emitted outside any activity has no trace id — worker jobs create an `ActivitySource` activity first (`metrics.md`, Custom Traces).
- Propagate `traceparent` through broker headers (`message-brokers`) so consumer logs continue the producer's trace instead of starting a new one.

## Sensitive Data & Redaction
- Never log: passwords, tokens, cookies, Authorization headers, personal data, card data; external request/response bodies in error logs (`api-clients`).
- Destructure deliberately: `{OrderId}` for scalars; `{@Entity}` only when the entity is guaranteed clean. A raw DTO may contain personal data — log identifiers, not payloads.
- Exception messages may embed input (e.g. an invalid URL). Log the exception object with a static message template and parameter values you control; sanitize the message when the exception can carry user input.
- If a property must be masked, mask before logging (a small helper or a Serilog.Expressions filter), not by filtering in the sink afterwards.

## Scopes
- Wrap a unit of work in `ILogger<T>.BeginScope(new Dictionary<string, object> { ["OrderId"] = orderId })`; add scope properties to the console template when useful.
- Do not duplicate scope properties into the message template — pick one place.
