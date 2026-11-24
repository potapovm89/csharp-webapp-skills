---
name: worker-host-jobs
description: >-
  Use this skill when developing or modifying background jobs, recurring tasks, cron schedules, or batch processors in WorkerHost using Quartz.NET, including running jobs on several replicas (clustered job store, distributed locks, misfires).
  Triggers (RU): фоновая задача, джоба, Quartz, cron, расписание, несколько реплик, кластер.
---

# WorkerHost Background Jobs

This skill covers the development and orchestration of background workers and scheduled tasks in `Itl.<Project>.WorkerHost` using Quartz.NET.

Before editing a job, read the host-to-Stories topology in the project's AGENTS/ADR, then check the replica strategy in `docs/settings/workerhost.md` and the effective deployment configuration. For several replicas, read [clustering.md](./references/clustering.md). Resolve conflicting settings before changing coordination.

**Example from `it-letter`** — the delivery job delegates to `MessageDeliveryProcessor`, which dispatches through `IStoryExecutor`. The claim Story uses a domain service that selects rows with `FOR UPDATE SKIP LOCKED`. The job schedules and coordinates work. Stories own the business operation and its data access.

## Architecture
- Jobs implement `Quartz.IJob` with suffix `Job` (e.g. `OrderExpirationJob`).
- Quartz.NET **4.x**: `IJob.Execute` returns `ValueTask` and receives a `CancellationToken` parameter. Pass that token to every async call. The 3.x signature `Task Execute(IJobExecutionContext context)` does not compile.
- Jobs are thin orchestrators. Business work goes through the chosen Stories mechanism or an approved RpcHost gateway. Stories call domain services. The job may call a `Common` infrastructure abstraction for coordination, such as a distributed lock.
- Always implement idempotency so retries and overlapping runs do not duplicate work.
- Process large data volumes page by page (e.g. 250 items at a time) to prevent memory spikes.
- **Several replicas**: with the default in-memory store every replica fires every job. `[DisallowConcurrentExecution]` alone does not help across pods. Every WorkerHost must use one documented approach: single replica, Quartz clustered ADO.NET store, a PostgreSQL advisory lock, or row-level work distribution with `FOR UPDATE SKIP LOCKED`. See [clustering.md](./references/clustering.md).
  - The choice is written in `docs/settings/workerhost.md` (section «Работа на нескольких репликах») in the same MR that adds the first job, with one line per job: approach and why duplicates are harmless or impossible.
  - Row-level distribution (`FOR UPDATE SKIP LOCKED`) makes parallel runs safe but does not replace this decision: document it as the chosen approach.
- If scheduled business work changes data and notifies another service, the Story writes the business change and the outbox row in the same transaction (see `message-brokers`).
- In-process dispatch uses `IMediator.Send` or `IStoryExecutor.Execute` as selected by the solution (`cqrs-stories`). The example uses Executor Stories.

```csharp
[DisallowConcurrentExecution]
public class OrderExpirationJob(
    IStoryExecutor storyExecutor,
    ILogger<OrderExpirationJob> logger) : IJob
{
    public async ValueTask Execute(IJobExecutionContext context, CancellationToken cancellationToken)
    {
        logger.LogInformation("Starting stale orders expiration sweep");
        await storyExecutor.Execute(new ExpireStaleOrdersStoryContext(), cancellationToken);
    }
}
```

## Program.cs Registration
```csharp
builder.Services.AddQuartzJob<OrderExpirationJob>(
    builder.Configuration.GetRequiredSection("Jobs:OrderExpirationSchedule").Bind);

builder.Services.AddQuartzHostedService(opt =>
{
    opt.WaitForJobsToComplete = true; // Wait for active jobs on shutdown
});

builder.Services.AddOpenTelemetry().WithTracing(cfg => cfg.AddQuartzInstrumentation());
```

## Subtopic References
- [scheduled-tasks.md](./references/scheduled-tasks.md) — 7-field Quartz cron expressions and guidelines.
- [repetitive-tasks.md](./references/repetitive-tasks.md) — Interval-based tasks and concurrency prevention.
- [clustering.md](./references/clustering.md) — Running jobs on several replicas: clustered job store (Quartz 4.x), advisory locks, work distribution with SKIP LOCKED, misfires and time zones.

## Completion Evidence

- Trace the job and its host-side helpers to the Story or approved gateway. Verify that host code does not resolve or call a business domain service. Record the dispatch path and applicable architecture or integration check.
- Verify the configured replica strategy. For a single replica, check the rendered deployment and retry of a firing. For a shared Quartz store or distributed lock, use two schedulers/workers and verify acquisition or exclusion as configured. For `SKIP LOCKED`, run concurrent claims against the same data and assert disjoint item IDs.
- Retry the changed operation in an isolated test and assert one business effect per item. Record commands, the strategy and observed results or any unavailable check.
