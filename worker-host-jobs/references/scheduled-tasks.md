# Scheduled Tasks (Cron) Reference

## Configuration
In `Jobs:<ScheduleName>`:
```json
"Jobs": {
  "SendStatusSchedule": {
    "Cron": "0 0 3 * * ?"
  }
}
```

## Quartz Cron Expression Format (7 Fields)
Format: `[Seconds] [Minutes] [Hours] [DayOfMonth] [Month] [DayOfWeek] [Year (optional)]`
- `0 0 3 * * ?` — Every day at 03:00:00 UTC.
- `0 0/15 8-18 ? * MON-FRI` — Every 15 minutes during work hours on weekdays.
- Note: One of `DayOfMonth` or `DayOfWeek` MUST be `?`.

## Guidelines
- Always validate cron expressions in unit tests via `CronExpression.IsValidExpression(expr)`.
- Cron expressions must be configured per environment in JSON.
