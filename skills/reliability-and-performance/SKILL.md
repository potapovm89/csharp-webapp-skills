---
name: reliability-and-performance
description: >-
  Use this skill when defining non-functional requirements, SLI/SLO and error budgets, burn-rate alerts, release observation criteria, load/capacity tests, overload control or dependency-failure behavior. Triggers (RU): надёжность, NFR, SLI, SLO, error budget, burn rate, нагрузка, производительность, деградация, отказоустойчивость.
---

# Reliability & Performance Requirements

Translate critical user journeys into measurable acceptance criteria. `observability` implements telemetry; this skill defines what it means and how it drives verification/release decisions. Read [sli-slo.md](./references/sli-slo.md) for objectives, [alerts-and-budgets.md](./references/alerts-and-budgets.md) for alert/release policies, and [load-and-failure-tests.md](./references/load-and-failure-tests.md) when testing capacity or failure behavior.

## When to Use (and When Not)
- **Use** for a new or changed critical user journey, a hot-path or capacity change, retry/timeout/overload behavior, or release observation criteria.
- **Skip** for routine changes inside an existing journey that already has an SLO — reuse its definition; do not create a new document per endpoint.

## Quick Checklist
1. Name the journey, its owner and the user-visible good/bad outcome.
2. Pick the SLI from real counters (not CPU or health probes) and write the target as an open decision if the owner has not set it.
3. Define deadlines, retries, limits and degradation for each dependency the change touches.
4. Add or update the alert + runbook; verify the query on fixture data.
5. Run the relevant load/failure scenario in a non-production environment and record measured results.

## Workflow
1. Identify critical journeys, acknowledged business outcomes, owners, expected load/data size, dependency failure modes and business impact. Start with the changed scenario; a small internal endpoint does not automatically need a new SLO if an existing journey covers it.
2. Write or update `docs/reliability/<journey>.md`: eligible events, good/bad outcome definition, observation boundary, measurement source, latency/freshness limits, target, rolling/calendar window, exclusions, low/no-traffic handling, load assumptions and owner. Targets come from product/operations decisions and evidence; do not impose example values or promise 100% availability.
3. Trace each SLI to the code or event producer that records its numerator and denominator. Verify the name available in the telemetry backend, units, labels and emission boundary. Pin the executable query and its expected results on fixture data. Use bounded-cardinality telemetry with `observability`. Unsampled counters/events measure business outcomes. Use sampled traces, probes, CPU and memory for diagnosis. Mark the SLI unverified if the source is missing or cannot be queried.
4. Define failure budgets and bounded behavior: end-to-end deadlines, retries/idempotency, queue/concurrency limits, backpressure, load shedding and optional-dependency degradation. Reuse `api-clients`, `message-brokers`, `worker-host-jobs` and `runtime-deployment`; do not create a competing retry/locking mechanism.
5. Verify with representative load and failure tests in an authorized non-production environment. Record latency distributions, throughput, error/freshness ratios, resource saturation, backlog drain and recovery; include dropped/unstarted test iterations and accepted work that never completed.
6. Connect alerts to a runbook and the approved error-budget policy. Define a separate bounded rollout observation window and rollback/hold conditions for `delivery-and-supply-chain`; a short canary does not prove a month-long SLO.

**Example from `it-letter`** — `docs/reliability/message-delivery.md` names `WorkerDelivery` as the freshness metric, but the implementation and export must be verified before using it. The `ConsumerMetrics` outcome counter measures broker intake. A delivery freshness SLI needs evidence from accepted-message timestamps and confirmed provider delivery, with overdue messages still represented in the denominator.

## Requirement Record
| Item | Evidence to provide |
|---|---|
| Journey/objective | Owner, business impact, target/window and denominator/exclusion rules |
| Measurement | Source implementation, exported metric/event name, exact query, units, labels, aggregation and missing-data behavior |
| Capacity | Normal/peak load, concurrency, dataset/skew, limits and test environment differences |
| Failure behavior | Dependency timeouts, degradation, overload response, retry/idempotency and recovery |
| Acceptance | Reproducible load/failure scenario, actual measurements and pass/fail thresholds |
| Operations | Dashboard, alert/runbook, error-budget release policy, review date |

`data-recovery` owns RPO/RTO, backup and restore proof. Link those objectives when data loss/restoration affects the journey; neither a high availability percentage nor replica count demonstrates recoverability. Security/integrity violations are not automatically acceptable just because an availability error budget remains.

## Completion Evidence
- [ ] For each changed SLI, record the source reference, exact backend query and fixture result. Use an input with known outcomes and assert the numerator and denominator separately. For a delivery fixture with five eligible messages past their deadline and two on-time deliveries, expect `2/5` when the remaining three are late, failed or unfinished.
- [ ] Run the query with no traffic and missing or partial telemetry. Assert the documented no-data/unknown result. For counters that can reset, test a reset without counting false successes.
- [ ] Relevant load, dependency-failure and overload cases have measured outcomes and bounded abort conditions.
- [ ] A runbook and accountable owner exist; release checks consume the same definitions and documented evidence.

Reference approach, reviewed 2026-11-14: [Google SRE: implementing SLOs](https://sre.google/workbook/implementing-slos/). The resulting targets and release policy are project decisions, not a generic SLA or certification claim.
