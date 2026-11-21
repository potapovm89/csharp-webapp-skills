# Alerts, Budgets & Release Decisions

## Error-Budget Policy
Record the policy owner, measurement window, escalation, release hold/resume conditions and evidence in `docs/reliability/`. Use the **same SLI definition** as the dashboard. State how emergency security/reliability fixes are reviewed during a hold, who can grant a time-bound exception and how it is recorded. Do not disable all safety fixes automatically or invent an override when the approver is unavailable. A remaining availability budget does not permit data corruption or a security bypass.

During an authorized release, `delivery-and-supply-chain` checks the current policy and a bounded rollout observation: business smoke outcomes, error/latency/freshness signals, backlog and saturation. Define minimum sample size/traffic and maximum observation time. If observations or required telemetry are insufficient, hold for a decision; do not silently pass. Keep prior digest/config/schema compatibility from `runtime-deployment` and `database-migrations` available for an authorized rollback.

Policy reference: [Google SRE: example error-budget policy](https://sre.google/workbook/error-budget-policy/), reviewed 2026-11-14. Adopt a policy suited to the service, not an unconditional copy of another organization's release freeze.

## Burn-Rate Alert Example
For target 99.9%, the allowed bad-event ratio is 0.001. A multiwindow alert checks both sustained consumption and whether it is still happening. Example starting point for a 30-day window: page when burn exceeds 14.4 in both 1 h and 5 min. Under steady traffic, that spends about 2% of the budget in one hour. Event volume changes alter the exact consumed fraction; tune windows/severity to the service and on-call response.

The following illustrative Prometheus rules use a **custom** counter, not an automatically exported .NET metric. `business_journey_outcomes_total` is incremented once per eligible terminal outcome with `outcome="good"` or `"bad"`; initialize both outcome series for each bounded `(service, journey)` pair. Define async outcome timing separately as in sli-slo.md. Adapt names to the real exporter; do not guess its label conversion.

```yaml
groups:
  - name: business-journey-slo
    rules:
      - record: journey:bad_ratio:rate1h
        expr: sum by (service, journey) (rate(business_journey_outcomes_total{outcome="bad"}[1h])) / sum by (service, journey) (rate(business_journey_outcomes_total{outcome=~"good|bad"}[1h]))
      - record: journey:bad_ratio:rate5m
        expr: sum by (service, journey) (rate(business_journey_outcomes_total{outcome="bad"}[5m])) / sum by (service, journey) (rate(business_journey_outcomes_total{outcome=~"good|bad"}[5m]))
      - alert: JourneyBudgetBurnFast
        expr: (journey:bad_ratio:rate1h > 14.4 * 0.001) and on (service, journey) (journey:bad_ratio:rate5m > 14.4 * 0.001)
        for: 2m
        labels:
          severity: page
        annotations:
          summary: "Sustained error budget burn for {{ $labels.service }}/{{ $labels.journey }}"
          runbook: "docs/runbooks/journey-budget-burn.md"
```

The runbook value is a repository-relative example; in deployed alert annotations use a reachable URL for the operator. The example is not a complete alert suite: define slower-burn/ticket behavior and low-traffic rules, deduplicate overlapping alerts and add a separate missing-instrumentation/telemetry alert. A zero denominator produces no meaningful ratio; do not mask missing/NaN data with `or vector(0)` and call it healthy. Operational thresholds for outbox/DLQ/backup remain useful diagnostics and integrity alarms even when no availability SLO alert fires.

Source: [Google SRE: alerting on SLOs](https://sre.google/workbook/alerting-on-slos/), reviewed 2026-11-14. The threshold above is illustrative; the SLO, traffic assumptions and severity are project-specific.

## Validation and Alert Operations
- Test the real recording/alert rules with the backend's rule validator; for Prometheus use `promtool check rules` and `promtool test rules` fixtures. Do not claim compatibility with another metrics backend without its own validation.
- Cover sustained high burn, short spike that clears, low volume, no traffic, missing one/all outcomes, counter reset and two replicas with different traffic. Verify alert onset/clear timing, series matching, severity and deduplication.
- Bind a responsible team, dashboard and runbook; set escalation/silence limits. A configuration change must not notify real on-call channels as part of a fixture test.
- Runbooks specify customer impact, data/integrity checks, safe mitigation and when to stop/escalate. Alert thresholds and exclusions are reviewed when objectives or telemetry change.

Tool reference: [Prometheus rule unit tests](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/).
