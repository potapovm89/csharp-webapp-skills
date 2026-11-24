# Load, Capacity & Failure Acceptance

## Test Contract
Record the target journey and SLI, normal/peak arrivals, concurrency, payload mix, dataset size/skew, cache state, topology/replicas and resource limits. Include realistic database indexes, tenant skew and broker partitions. State environment differences and what the result can establish; Aspire/Testcontainers correctness tests alone do not prove production capacity.

Use tools already approved by the project (`postgresql-data-access`, performance.md: k6/JMeter), pin versions and run only in authorized non-production environments. Do not install a new tool or provision paid resources without the required authorization. Define max load, duration, data/storage limits, abort metrics, a stop owner and cleanup scope. Use synthetic/masked data and test sinks; block production endpoints and external paid/irreversible side effects.

## Useful Scenarios
| Scenario | Observable acceptance |
|---|---|
| Normal/peak steady load | Required throughput and outcome/latency/freshness targets at realistic size |
| Ramp/burst and recovery | Queues/concurrency stay bounded; explicit load shedding; capacity recovers after the burst |
| Soak | Memory, connection use and backlog stabilize; no leak or accumulating uncompleted work |
| Slow/unavailable dependency | Bounded deadlines and retries, circuit/degradation behavior and no retry storm |
| Redis failure/cold cache | Correct bounded DB fallback, concurrency/stampede limits across replicas |
| Broker interruption/consumer restart | Durable pending work, duplicates handled, eventual drain within the agreed deadline |
| Rolling shutdown | New work stops, in-flight work drains or retries safely; no duplicate business effect |

Select applicable scenarios by risk; changes to hot paths, shared infrastructure, retry limits or resource profiles require the corresponding evidence. Never accept silently lost business work as “successful degradation.” Keep mandatory authorization/integrity checks active during overload.

## Measuring the Test
- Measure client/edge and service observations, success/error ratios, latency distribution, completed/accepted counts, timeouts, dropped/unstarted generator iterations and expected freshness. Do not discard errors from a latency report without an explicit separate error measure.
- Check that the load generator sustained the intended arrival rate. A saturated closed-loop generator can reduce offered load and hide latency/overload; choose an appropriate model and report that limitation rather than claiming the target RPS passed.
- Record CPU/memory/GC, connection pools, query durations, broker lag/queue age and outbox pending/exhausted metrics as explanations. CPU below a threshold is not a user-journey SLO.
- For asynchronous workloads, measure from original acceptance through correct completion, including retries, and continue until backlog drains or the agreed timeout fails. An empty input queue alone does not prove business completion.
- Compare before/after with the same workload/tool/environment inputs when evaluating a regression. Do not average percentiles across pods or runs; preserve distributions or calculate the relevant aggregate from observations.

## Failure and Release Evidence
For each fault, specify injection boundary, expected degraded behavior, time/retry/concurrency budgets, pass/fail thresholds, abort condition, restoration step and integrity checks. A failure experiment should restore normal behavior in the test environment without enabling production access. Capture the measured recovery after removing the fault.

Store the scenario/config version, source revision and image digest, timestamps, inputs, raw-results reference and sanitized conclusion in `docs/reliability/` or access-controlled CI evidence. Report incomplete/aborted tests as such. Required failures or unknown outcomes hold the affected release gate (`delivery-and-supply-chain`); they are not waived by unit tests or a green readiness probe.
