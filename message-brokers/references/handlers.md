# Consumer Message Processors Reference

## Unified Interface: `IProcessor<TKey, TMessage>`
All message handlers implement `IProcessor<TKey, TMessage>` from `Common/Services/ConsumerProcessor`:

```csharp
public interface IProcessor<TKey, TMessage>
{
    Task ExecuteAsync(ConsumedMessageDto<TKey, TMessage> message, CancellationToken ct = default);
}
```

The processor never references broker-specific types (`ConsumeResult` or `BasicDeliverEventArgs`). It only works with `ConsumedMessageDto<TKey, TMessage>`.

`ConsumedMessageDto` must carry the standard headers (`message-id`, `message-type`, `message-version`, `correlation-id`) in addition to key and payload. If the solution's DTO has only key and payload, extend it before relying on `message-id` for deduplication.

## Processor Lifecycle Pattern
1. **Validate the envelope**: known `message-type`, supported `message-version`. Otherwise throw a dedicated poison-message exception → the worker sends the message to the DLQ without retries.
2. **Deserialize** `message.Message` (JSON) into a contract DTO from `Contracts`. A deserialization failure is a poison message as well.
3. **Dispatch to a Story.** The processor is a host entry point: it builds a command (payload + consumer name + `message-id`) and sends it through the solution's Stories mechanism. It does not inject domain services, `ITransactionScope` or the inbox service — those belong to the Story (`solution-architecture`: hosts use DataAccess only for DI registration). A `ValidationException` from the Story is rethrown as a poison-message exception.
   ```csharp
   var saved = await sender.Send(new AddOrderStatusCommand
   {
       Status = status,
       Consumer = CONSUMER_NAME,
       MessageId = message.MessageId
   }, ct);
   ```
4. **In the Story: persist transactionally with deduplication** — in one transaction: the inbox insert or the unique-key insert (see "Inbox" in `outbox.md`), the business changes (mapping via extension methods, `csharp-code-style`), and outbox rows for any messages it publishes:
   ```csharp
   await using (await transactionScope.BeginAsync(ct))
   {
       try
       {
           if (!await inboxDomainService.TryRegisterAsync(CONSUMER_NAME, message.MessageId, ct))
           {
               logger.LogInformation("Message {MessageId} already processed", message.MessageId);
               return;
           }

           // business changes + outbox rows
           await transactionScope.CommitAsync(ct);
       }
       catch
       {
           await transactionScope.RollbackAsync(CancellationToken.None);
           throw;
       }
   }
   ```
5. **Required side effects** (business notifications and mandatory audit records) are durable: persist them in the same transaction or write an outbox event there. Do not replace mandatory delivery with catch-and-log after commit. Only explicitly optional telemetry may fail after commit without failing the already committed operation; document that distinction.
6. **Let processing failures propagate.** The worker decides between retry, retry topic/queue and DLQ and only then acknowledges or stores the offset. Never turn a failure of required processing into a successful return by catching and logging it. Optional telemetry from step 5 is the explicit exception.

## Checklist for a New Consumer
- [ ] Idempotent: inbox or unique business key, covered by an integration test that delivers the same message twice.
- [ ] Poison messages and unsupported versions go to the DLQ (integration test).
- [ ] Transient failures are retried, then dead-lettered; the offset/ack happens only after that.
- [ ] Messages published by the consumer go through the outbox.
- [ ] Trace context is extracted from headers.
- [ ] Topic/queue, DLQ, key semantics and versions are documented in `docs/contracts/`.
