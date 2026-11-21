# SLI/SLO Specification

## Define the Observation Boundary First
An SLI is a measured indicator; an SLO is its target over an agreed window; an SLA adds external commitments and consequences and must not be invented by an implementation agent. Specify whether events are HTTP attempts at the ingress, logical user operations, scheduled runs or accepted business messages. Do not mix those denominators or let internal retries inflate success counts.

| Journey type | Eligible unit | Example good outcome (choose actual values with the owner) |
|---|---|---|
| Interactive command | Valid authorized operation at the user-facing boundary | Correct durable result returned before its deadline; no duplicate business effect |
| Read | Eligible read request | Correct/authorized response within the latency and freshness bound |
| Async processing | Unique accepted business item/cohort | Correct completion within its processing deadline |
| Scheduled sweep | Scheduled execution/business items due by a time | Required work completed before the business deadline |

For every definition, record:
- Owner, business impact, scope (service/journey/region), load assumptions and target/window; use separate objectives for availability, latency, freshness and integrity when needed rather than a vague “healthy” metric.
- Exact inclusion/exclusion and outcome rules. Explicitly decide invalid/unauthorized user inputs and client cancellations; include server-generated 5xx/timeouts and overload rejections affecting legitimate work. Do not blanket-exclude all 4xx (e.g. capacity-induced 429), planned maintenance or dependency failures. Changes to exclusions are reviewed prospectively, never used to rewrite a missed objective after an incident.
- Clock/source, event completion timing, histogram bucket boundaries and data retention. An ingress request never seen by the app requires ingress/synthetic evidence; an app-only counter cannot prove external availability. Avoid aggregation that averages per-pod percentages; aggregate event counts at the agreed boundary.
- Missing telemetry, zero traffic, low sample size, resets and partial rollout behavior. No events means undefined/insufficient evidence, not automatic 100%. Synthetic probes can supplement low-volume services but must be identified separately from real-user events.

## Calculations
For event-based objectives with eligible events E, good events G, bad events B = E - G and target T (a fraction):

```text
SLI = G / E                         (defined only when E > 0)
allowed_bad_events = (1 - T) * E
remaining_budget = allowed_bad_events - B
burn_rate = (B / E) / (1 - T)       (requires E > 0 and 0 < T < 1)
```

Example only: 1,000,000 eligible requests at target 99.9% allow 1,000 bad requests. If 700 are bad, 300 remain. This event budget is not a promise of a fixed number of downtime minutes; traffic volume varies. For a time-based SLI, define eligible time slices and their good/bad rule separately.

Latency may be “fraction of eligible requests with a successful result within 500 ms” (illustrative threshold), not merely average duration. Define whether slow successful requests count against availability, latency or both; never let fast failures make a latency-only chart look like business success. Histogram buckets must support the chosen threshold; do not average p95/p99 values across pods.

Async deadlines need cohort/age handling: count an accepted item once when its deadline can be evaluated, keep unresolved overdue work in the bad outcome and avoid dropping it from the denominator because it never completed. Distinguish original acceptance from retries/replay; include a “stuck item” test. Queue lag alone does not prove correct end-to-end completion.

## Example Document Fields
In `docs/reliability/<journey>.md`, include business scope/owner; measurement boundary; eligible/good/bad/excluded events; target/window; exact counters/histogram/query; no/low-data rules; workload/data assumptions; dependency and overload behavior; load/failure evidence; dashboard/alert/runbook links; budget/release policy; review date. Mark an unresolved target explicitly and obtain the needed decision before claiming the objective is met.

## Query Verification
Use small controlled series/event fixtures with known totals, including mixed outcomes, a slow success, timeout/429, missing series, zero traffic, reset and an unfinished async item. Verify weighted aggregation, units, cohort boundaries and the chosen denominator. A load test percentile is useful acceptance evidence but does not substitute for the live SLI's measurement contract.
