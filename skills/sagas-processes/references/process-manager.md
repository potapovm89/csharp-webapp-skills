# Process Manager (Default Pattern)

## Schema

```dbml
Table process_instances {
  id bigint [pk, increment]                // NumericKeyEntity / BaseEntity (domain-modeling)
  process_type int [not null]              // enum: OrderFulfillment = 1, ...
  state int [not null]                     // explicit state machine enum
  entity_id bigint [not null]              // aggregate the process is about
  next_wake_at_utc timestamptz [null]      // due time and claim lease; NULL in terminal and operator-only states
  attempts int [not null, default: 0]      // attempts of the current step, reset on transition
  is_deleted boolean [not null, default: false]
  create_date timestamptz [not null]
  update_date timestamptz [null]
}

Table process_events {
  id bigint [pk, increment]
  process_id bigint [not null, ref: > process_instances.id]
  step int [not null]                      // enum
  outcome int [not null]                   // Completed | Failed | Compensated
  detail text [null]                       // client-safe description, no PII dump
  create_date timestamptz [not null]
}
```

- Two tables, created by migration. Indexes: partial `(process_type, next_wake_at_utc) WHERE next_wake_at_utc IS NOT NULL AND is_deleted = FALSE` (terminal processes drop out of the scan and the stuck gauge) and `(process_id)` on events (`database-migrations`).
- Timestamps come from one source per entity (`domain-modeling`, entity.md). A process instance is a system record without an author, so it uses the `BaseEntity` columns (`create_date`/`update_date`); never mix them with the `IKnowCreator`/`IKnowUpdater` columns.
- Optimistic concurrency uses the PostgreSQL system column `xmin`, mapped on the entity (`postgresql-data-access`, concurrency.md). It is not a declared column and does not appear in the DBML. A concurrent claim or a user command that changes the process state conflicts → 409/skip, never a silent overwrite.
- Per-step business payloads are not stuffed into the instance row: each step's data lives in the aggregate it belongs to, the instance carries ids. A JSONB column is allowed only for small diagnostic context, documented in `docs/database/` (`postgresql-data-access`).

## States and Steps

```csharp
public enum OrderFulfillmentState
{
    /// <summary>Процесс создан, ожидает резервирования товара.</summary>
    PendingStockReservation = 1,
    /// <summary>Товар зарезервирован, ожидает списания средств.</summary>
    PendingPayment = 2,
    /// <summary>Средства списаны, ожидает отгрузки.</summary>
    PendingShipment = 3,
    /// <summary>Завершён успешно.</summary>
    Completed = 4,
    /// <summary>Компенсация выполнена, процесс закрыт.</summary>
    Compensated = 5,
    /// <summary>Остановлен, требуется оператор.</summary>
    Failed = 6
}
```

- One enum per process type in `Domain/Enums`; allowed transitions are checked in the domain service (`TransitionAsync` refuses invalid transitions — a transition matrix, not `if`s scattered across stories). Entering a terminal state (`Completed`, `Compensated`, `Failed`) sets `next_wake_at_utc = NULL`.
- XML docs are mandatory on every state (`csharp-code-style`).
- Track dispatch reservations and an explicit reconciliation/manual state when an external outcome can remain unknown. Reconciliation can schedule a status lookup. Clear the wake time and alert on an operator-only state so the scheduler does not claim it.

## Steps and Compensations

The example uses Executor Stories and a provider that accepts an idempotency key. With MediatR the same body lives in a command handler (`cqrs-stories` — one mechanism per solution).

```csharp
// One step = one Story, idempotent, dispatched by the runner or by an external event consumer.
public class ChargePaymentStory(...) : IStory<ChargePaymentStoryContext, long>
{
    private const string PROCESS_NAME = "order-fulfillment";

    public async Task<long> ExecuteAsync(ChargePaymentStoryContext context, CancellationToken ct)
    {
        await validator.ValidateAndThrowAsync(context, ct);
        var stepKey = $"{context.ProcessId}:{ProcessStep.ChargePayment}";

        // 0. Cheap pre-check without a transaction: the other trigger may have completed the step already.
        if (!await processDomainService.IsInStateAsync(context.ProcessId, OrderFulfillmentState.PendingPayment, ct))
        {
            return context.ProcessId;
        }

        // 1. External call OUTSIDE the transaction (no HTTP inside ITransactionScope — postgresql-data-access).
        //    The provider idempotency key is derived from (process, step): a re-run after a crash between
        //    the charge and step 2 returns the same charge instead of charging twice (api-clients).
        var charge = await paymentClient.ChargeAsync(context.OrderId, context.Amount, idempotencyKey: stepKey, ct);

        await using (await transactionScope.BeginAsync(ct))
        {
            try
            {
                // 2. Local dedup: false = the step is already recorded (duplicate run, or the other trigger won).
                if (!await inboxDomainService.TryRegisterAsync(PROCESS_NAME, stepKey, ct))
                {
                    return context.ProcessId;
                }

                // 3. Business effect + state transition + "next step" message in the SAME transaction.
                await paymentDomainService.MarkPaidAsync(context.OrderId, charge.ChargeId, ct);
                await processDomainService.TransitionAsync(context.ProcessId,
                    OrderFulfillmentState.PendingPayment, OrderFulfillmentState.PendingShipment,
                    ProcessStep.ChargePayment, ProcessOutcome.Completed,
                    nextWakeAtUtc: timeProvider.GetUtcNow() + options.Value.ShipmentTimeout, ct);
                await outbox.AddAsync(OutboxMessageFactory.Create("shipment-requested", payload), ct);

                await transactionScope.CommitAsync(ct);
            }
            catch
            {
                await transactionScope.RollbackAsync(CancellationToken.None); // a cancelled ct must not skip the rollback
                throw;
            }
        }

        return context.ProcessId;
    }
}
```

- The step Story makes the external call. For providers without deduplication, use the durable reservation and outcome rules in [the main skill](../SKILL.md#external-effect-with-an-unknown-outcome). Document how the provider outcome can be correlated and who resolves it when evidence is unavailable. An optimistic pre-check alone does not reserve an external send.
- **Compensation map** is declared next to the state machine, in one static structure: failure in `PendingPayment` → `ReleaseStock`; failure in `PendingShipment` → `RefundPayment`, then `ReleaseStock`. Compensations run in reverse completion order over completed steps only and are separate Stories with the same idempotency/transaction rules (step key `<processId>:<compensation>`).
- A compensation that is itself an external effect follows the forward-step rules. `RefundPayment` is a new provider operation, not an undo of the charge: it gets its own provider idempotency key, and its outcome is applied in a transaction. If it still fails after bounded attempts, the process goes to `Failed` and an operator decides (`data-recovery` principle: external effects are not rolled back silently).
- An effect that has no compensation at all (a shipment handed to a carrier) gets no compensation entry — the process stops in `Failed`/manual state if it must be undone, and the runbook covers it.

## The Runner (WorkerHost)

The runner is a Quartz job that only dispatches Stories (`worker-host-jobs`); the claim query and the step logic live in Stories and domain services.

1. **Claim with a lease.** A `ClaimDueProcesses` Story runs one short transaction and commits it:
   ```sql
   UPDATE process_instances
   SET next_wake_at_utc = @now + @lease, attempts = attempts + 1
   WHERE id IN (
       SELECT id FROM process_instances
       WHERE next_wake_at_utc <= @now AND is_deleted = FALSE
       ORDER BY next_wake_at_utc
       LIMIT @batch
       FOR UPDATE SKIP LOCKED)
   RETURNING id, process_type, state, attempts;
   ```
   `@now` comes from `TimeProvider`, never `now()` in SQL: tests drive time with `FakeTimeProvider`, and direct clock reads are banned. The row lock protects only the claim; after commit the moved `next_wake_at_utc` (the lease) keeps other replicas away. This is clustering approach D with a lease, because steps make external calls and must not hold row locks (`worker-host-jobs`, clustering.md).
2. **Dispatch.** For each claim the job invokes the Story for the recorded state, with bounded parallelism. A completed step schedules the next one. After a lease expires, a reservation with an unrecorded result goes to reconciliation unless a retry with the same provider key and payload is still permitted.
3. **Resolve at exhaustion.** For a confirmed failure, the Story enters compensation when required, or records `Failed` and raises the applicable alert. For an unknown result, schedule reconciliation or request the documented operator decision. Exhaustion never turns an unconfirmed outcome into a failure or a compensation trigger.

- Timeouts are data, not code: each transition sets the next `next_wake_at_utc`. The lease must exceed the step's worst-case duration (external call timeout × retries).
- Misfire semantics for the runner trigger: `DoNothing` — the next scan catches up (`worker-host-jobs`, clustering.md).
- Steps may also be triggered by external events (a payment webhook arrives): the event consumer dispatches the same step Story. The runner and the consumer converge through the pre-check, the inbox key and the transition check — whoever commits first wins, the other becomes a no-op.

## Metrics and Runbook
| Instrument | Alert |
| ---------- | ----- |
| `process.active` gauge by type (non-terminal states) | unbounded growth |
| `process.stuck` gauge (`next_wake_at_utc` overdue > X min) | any sustained |
| `process.failed` counter by type, outcome | any — Failed needs an operator |

- Gauges refresh from a background loop like outbox metrics (`observability`, metrics.md) — never query the database in the observable callback.
- Monitor unresolved operator-only states separately from the overdue wake-time gauge. Every process type has a runbook in `docs/runbooks/` that explains how to inspect `process_events`, resume a stuck step and resolve an unknown external outcome.

## Tests (minimum set)
- A step applies its effect, transition and outbox row in one transaction; a duplicate invocation does not duplicate the effect, and the provider receives the same idempotency key.
- Two runners claiming concurrently get disjoint batches; an expired lease makes the instance claimable again.
- Advance `FakeTimeProvider` to the deadline. At exhaustion assert compensation/failure only for a confirmed failure. An unresolved external outcome must remain in reconciliation or manual handling with no compensation.
- Compensation runs in reverse order over completed steps only and is idempotent.
- Simulate a crash between the external call and the transaction. Verify recovery with the same provider key and a single effect. Without provider deduplication, verify the reconciliation or manual state before any new send.
- An invalid transition (`TransitionAsync` from a wrong state) throws — a unit test of the domain service (`testing`, unit-testing.md).

## Process List Discipline
- Adding a process type requires an ADR: states, steps, compensations, non-compensable effects, timeout and lease policy, owner (`project-documentation`).
- Deleting a process type: stop starting new ones, drain/complete or explicitly archive instances — a dangling process without a runner is a stuck-process alert forever.
