# Running Jobs on Several Replicas

## The Problem
With the default in-memory job store every replica of WorkerHost has its own scheduler. With 3 pods every job fires 3 times. `[DisallowConcurrentExecution]` only prevents overlap **inside one scheduler**; it does nothing across replicas.

Choose one approach explicitly for every WorkerHost and record it in `docs/settings/workerhost.md`. Link the applicable architectural decision when the strategy changes:

| Approach | When |
| -------- | ---- |
| **A. Single replica** (`replicas: 1`, deployment strategy `Recreate`) | Jobs are cheap, a short gap during redeploy is acceptable. Simplest. |
| **B. Quartz clustered ADO.NET job store** (PostgreSQL) | Several replicas coordinate acquisition of scheduled firings through the shared store. Default for production; recovery/retries can execute work again, so business effects remain idempotent. |
| **C. Distributed lock inside the job** (PostgreSQL advisory lock) | In-memory store kept, a few jobs that must not run in parallel across pods. |
| **D. Row-level work distribution** (`FOR UPDATE SKIP LOCKED`) | All replicas may run the sweep; each transaction claims a disjoint batch and commits its business effect and processed marker together. |

Independently of the approach, **jobs stay idempotent**: misfire recovery, node failover and retries can run a firing again.

## B. Clustered Job Store (Quartz.NET 4.x)
Schema: the Quartz tables are created **by a migration** like any other schema change (take the official PostgreSQL script for the exact Quartz version from the Quartz.NET repository and add it as a migration). Do not use `ProvisionSchema()` outside local environments.

```csharp
builder.Services.AddQuartz(q =>
{
    q.UsePersistentStore(store =>
    {
        store.UsePostgres(ds => ds.ConnectionStringName = "postgresdbForAPI");
        store.UseSystemTextJsonSerializer(_ => { });
        store.UseClustering(c =>
        {
            c.CheckinInterval = TimeSpan.FromSeconds(10);
            c.CheckinMisfireThreshold = TimeSpan.FromSeconds(20);
        });
        store.ConfigureStore(o =>
        {
            o.TablePrefix = "qrtz_";
            o.StoreJobDataAsStrings = true;   // readable, version-tolerant job data
            o.MisfireThreshold = TimeSpan.FromMinutes(1);
        });
    });
});
```

Notes (checked against Quartz.NET 4.7):
- `UseClustering` turns on database locking and generates a unique instance id per node automatically; do not hard-code `SchedulerId`.
- The 3.x API (`UseProperties`, `SimpleInstanceIdGenerator`, `quartz.jobStore.clustered` properties) does not apply to 4.x — check the Quartz version in `Directory.Packages.props` before writing configuration.
- All nodes must have synchronized clocks (NTP); clustering relies on timestamps.
- `[DisallowConcurrentExecution]` becomes cluster-wide with a persistent store.
- Keep job data small (ids, not payloads); the store serializes it into the database.
- Graceful shutdown: `AddQuartzHostedService(o => o.WaitForJobsToComplete = true)` and the pod termination grace period longer than the longest job.

## C. PostgreSQL Advisory Lock
The job depends on an infrastructure abstraction from `Common` (e.g. `IDistributedJobLock`), not on DataAccess types: the lock implementation owns its dedicated connection. Business work goes through Stories, one scope and transaction per item.

```csharp
public class PaymentStatusJob(IServiceScopeFactory scopeFactory, IDistributedJobLock jobLock, ILogger<PaymentStatusJob> logger) : IJob
{
    public async ValueTask Execute(IJobExecutionContext context, CancellationToken ct)
    {
        await using var handle = await jobLock.TryAcquireAsync(JobLockKeys.PAYMENT_STATUS, ct);
        if (handle is null)
        {
            logger.LogInformation("Job {Job} is already running on another node", nameof(PaymentStatusJob));
            return;
        }

        // page through ids with a query, then per item: new scope -> sender.Send(new AdvancePaymentStatusCommand { ... })
    }
}
```

Infrastructure jobs without business rules (outbox cleanup, relay) call an infrastructure service in `Common` instead of a Story.

- Lock keys are `long` constants in one class; never compute them from strings at runtime without a fixed hash.
- Without the helper: `pg_try_advisory_xact_lock` inside one transaction (released on commit/rollback — the whole run must fit in it), or `pg_try_advisory_lock` on a dedicated connection with `pg_advisory_unlock` in `finally`.

## Distributing Work Between Replicas
A job that processes many rows can let all replicas work in parallel instead of electing one: select a batch with `FOR UPDATE SKIP LOCKED` inside a transaction, process it, commit. A row lock excludes another worker only while the transaction is active; rollback/crash permits later processing again. Make the business change and processed-state update atomic; required notifications go through the outbox. Do not perform external business I/O while holding these locks. When each item needs external calls, use the lease variant: claim a batch with `UPDATE … SET <due_at> = @now + @lease … WHERE id IN (SELECT … FOR UPDATE SKIP LOCKED) RETURNING …`, commit, then process each item in its own transaction; an expired lease makes the item claimable again (`sagas-processes`, process-manager.md).

## Misfires and Time Zones
- Cron schedules are evaluated in UTC unless a time zone is set explicitly; set it explicitly for business-hour schedules (`.InTimeZone(...)`).
- Choose a misfire instruction per trigger: `WithMisfireHandlingInstructionDoNothing()` for periodic sweeps (the next run catches up), `FireAndProceed`/`FireNow` when every run matters.
