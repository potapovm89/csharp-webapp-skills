# Dapper Data Access Reference

## When to Use Dapper
Used where change tracking is not needed: high-throughput consumer hosts (KafkaHost, RabbitHost) and fast bulk operations.

## Architecture & Base Classes
- `BasePostgreDomainService<T, TId>` in `DataAccess/Dapper/Base`:
  - Automatically compiles SQL table name, column list, and SELECT templates from entity attributes (`[Table]`, `[Column]`, `[Key]`).
  - Key-type subclasses: `BaseLDomainService<T>` (long), `BaseGDomainService<T>` (Guid), `BaseSDomainService<T>` (string).
  - **Name collision**: the EF base classes have the same names but live in `DataAccess/EF/Base`. Always import the namespace matching the access technology of the database (`...DataAccess.Dapper.Base` here); never mix them in one service.
  - Out of the box provides `CreateAsync` (INSERT RETURNING id), `GetAsync`, `UpdateAsync`, and `DeleteAsync`.
  - `DeleteAsync` performs a **physical** `DELETE`. It is allowed only for technical tables without `ICanBeSoftDeleted` (queues, temporary buffers, idempotency journals). For business entities implement a soft delete (`UPDATE ... SET is_deleted = TRUE`) and filter `is_deleted = FALSE` in custom queries.

## Custom Queries
Always use parameterized queries with `CommandDefinition`, passing the active transaction and cancellation token:

```csharp
public class StoreOrderDomainService(DbConnection connection, ITransactionScope transactionScope) 
    : BaseLDomainService<StoreOrder>(connection, transactionScope), IStoreOrderDomainService
{
    public Task<StoreOrder?> GetAsync(string storeId, string orderId, CancellationToken ct)
    {
        string query = $"{_selectAll} WHERE order_id = @OrderId AND store_id = @StoreId";
        var command = new CommandDefinition(
            query,
            new { OrderId = orderId, StoreId = storeId },
            transaction: _transactionScope.Transaction,
            cancellationToken: ct);
        return _connection.QuerySingleOrDefaultAsync<StoreOrder>(command);
    }
}
```

## Rules
- SQL concatenation of user input is strictly prohibited; always use parameters.
- Query column names must match database `snake_case`.
- Manage transactions with `ITransactionScope`.
