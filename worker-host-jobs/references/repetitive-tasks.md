# Repetitive (Interval) Tasks Reference

## Configuration
In `Jobs:<ScheduleName>`:
```json
"Jobs": {
  "PaymentStatusSchedule": {
    "Interval": "00:01:00"
  }
}
```

## Simple Schedule Setup
Translates to Quartz Simple Schedule:
```csharp
t.WithSimpleSchedule(x => x.WithInterval(options.Interval.Value).RepeatForever());
```

## Guidelines
- Prefer intervals for recurring sweeps (polling Redis keys, batching queue messages) where wall-clock alignment is unnecessary.
- Protect against execution overlapping by using `[DisallowConcurrentExecution]` if processing time might exceed interval. It works only within one scheduler; across replicas see [clustering.md](./clustering.md).
- Use `WaitForJobsToComplete = true` for graceful shutdown.
