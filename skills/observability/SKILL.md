---
name: observability
description: >-
  Use this skill when adding or configuring structured logging with Serilog, OpenTelemetry tracing and metrics, ActivitySource instrumentation, or health check endpoints (/health, /alive).
  Triggers (RU): логи, Serilog, метрики, трейсы, OpenTelemetry, health, алерты, мониторинг.
---

# Observability (Logging & Telemetry)

This skill covers Serilog structured logging, OpenTelemetry tracing, custom meters, and health checks across all host applications.

## Key Principles
1. **Structured Logging Only**: Never interpolate strings into logs. Use named placeholders (`_logger.LogInformation("Entity {Id} saved", id)`).
2. **English Messages**: All log descriptions are written in English.
3. **Exception as First Argument**: `_logger.LogError(ex, "Failed to process message")`.
4. **No Sensitive Leaks**: Passwords, tokens, and PII must never be written to logs.
5. **Standardized OpenTelemetry**: Every host calls `builder.AddServiceDefaults()` and `host.MapDefaultEndpoints(port)`.
6. **Measurement contract**: [reliability-and-performance](../reliability-and-performance/SKILL.md) defines journey SLI/SLO, numerator/denominator, targets/windows and alert policy; this skill implements the counters/histograms, dashboards and export. [data-recovery](../data-recovery/SKILL.md) defines backup/archive/drill coverage signals. Do not infer business success from probes or sampled traces alone.

## Subtopic References
- [logging.md](./references/logging.md) — Serilog configuration, log levels, and structured templates.
- [metrics.md](./references/metrics.md) — OpenTelemetry wiring (incl. Npgsql/EF Core meters), naming, outbox and consumer metrics with alerts, ActivitySource tracing, health check endpoints.
