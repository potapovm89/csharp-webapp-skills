---
name: worker-host-jobs
description: >-
  Use this skill when developing or modifying background jobs, recurring tasks, cron schedules, or batch processors in WorkerHost using Quartz.NET, including running jobs on several replicas (clustered job store, distributed locks, misfires).
  Triggers (RU): фоновая задача, джоба, Quartz, cron, расписание, несколько реплик, кластер.
---

# WorkerHost Background Jobs

This skill covers the development and orchestration of background workers and scheduled tasks in `Itl.<Project>.WorkerHost` using Quartz.NET.

## Architecture
- Jobs implement `Quartz.IJob` with suffix `Job` (e.g. `OrderExpirationJob`).
- Quartz.NET **4.x**: `IJob.Execute` returns `ValueTask` and receives a `CancellationToken` parameter. Pass that token to every async call. The 3.x signature `Task Execute(IJobExecutionContext context)` does not compile.
- Jobs are thin orchestrators delegating business logic to Stories and domain services.
- Always implement idempotency so retries and overlapping runs do not duplicate work.
- Process large data volumes page by page (e.g. 250 items at a time) to prevent memory spikes.
- **Several replicas**: with the default in-memory store every replica fires every job. `[DisallowConcurrentExecution]` alone does not help across pods. Every WorkerHost must use one documented approach: single replica, Quartz clustered ADO.NET store, a PostgreSQL advisory lock, or row-level work distribution with `FOR UPDATE SKIP LOCKED`. See [clustering.md](./references/clustering.md).
  - The choice is written in `docs/settings/workerhost.md` (section «Работа на нескольких репликах») in the same MR that adds the first job, with one line per job: approach and why duplicates are harmless or impossible.
  - Row-level distribution (`FOR UPDATE SKIP LOCKED`) makes parallel runs safe but does not replace this decision: document it as the chosen approach.
- A job that changes data and notifies other services writes to the outbox in the same transaction (see `message-brokers`).
- Jobs dispatch to Stories through the solution's mechanism (`cqrs-stories`): `IMediator.Send` or `IStoryExecutor.Execute`. The example uses Executor Stories.

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
