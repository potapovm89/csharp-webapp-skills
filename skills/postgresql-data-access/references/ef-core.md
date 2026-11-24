# Entity Framework Core Data Access Reference

## Registration
```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseNpgsql(builder.Configuration.GetConnectionString("postgresdbForAPI"), optionsBuilder =>
    {
        optionsBuilder.MigrationsAssembly("Itl.<Project>.EntityFramework.Migrations");
    });
});
```

## Domain Services
Direct usage of `DbContext` in business logic is prohibited. All operations pass through domain services in `DataAccess/EF`:
- Base classes: `BaseLDomainService<T>` (long), `BaseGDomainService<T>` (Guid), `BaseSDomainService<T>` (string).
- Base class provides: `GetAsync(id, shouldTrack, ct)`, `Create`, `Update`, `Delete`, and protected `GetAll(bool shouldTrack)`.
- **Name collision**: the Dapper base classes have the same names but live in `DataAccess/Dapper/Base`. Import `...DataAccess.EF.Base` for EF services.
- `Delete` removes the row physically. Use it only for technical entities without `ICanBeSoftDeleted`; business entities get a `SoftDelete` method and a filtered `GetAll` override, as below.

```csharp
public class CustomersDomainService(AppDbContext context) 
    : BaseLDomainService<Customer>(context), ICustomersDomainService
{
    public void SoftDelete(Customer entity)
    {
        entity.IsDeleted = true;
        Update(entity);
    }

    protected override IQueryable<Customer> GetAll(bool shouldTrack)
    {
        return base.GetAll(shouldTrack).Where(x => !x.IsDeleted);
    }
}
```

## Query Rules
- Read-only queries must use `shouldTrack = false` (`AsNoTracking`).
- Projections: Project directly to DTOs in query methods using `.Select(...)` rather than loading entire entity graphs into memory.
- Pagination is keyset-based (`performance.md`): order by a stable indexed key and read one extra row to detect the next page. Offset (`Skip`/`Take`) is allowed only for small bounded lists (admin dictionaries, fixed catalogs) — it degrades linearly with page depth and skips/duplicates rows under concurrent inserts.

```csharp
// Cursor page contract (Contracts): items + opaque next cursor. The solution's offset PageResult (Data/Page/Total)
// is for small bounded lists; do not add a cursor to it or reuse it for keyset pages.
public sealed record CursorPage<T>(IReadOnlyList<T> Items, Guid? NextCursor);

public async Task<CursorPage<OrderDto>> GetPageAsync(Guid? cursor, int limit, CancellationToken ct)
{
    var page = await context.Orders
        .Where(x => !x.IsDeleted)
        .OrderBy(x => x.Id)
        .Where(x => cursor == null || x.Id > cursor.Value)
        .Take(limit + 1)
        .Select(x => new OrderDto { Id = x.Id, ... })
        .ToListAsync(ct);

    return new CursorPage<OrderDto>(
        page.Take(limit).ToList(),
        page.Count > limit ? page[limit - 1].Id : null); // cursor = last returned item
}
```

- `CountAsync` over the full filter is only for contracts that explicitly require a total — it re-scans the same filter on every page.
- Pass `CancellationToken` to all async calls.

## Transactions
Use `ITransactionScope` for multiple entity updates. `BeginAsync` returns `IAsyncDisposable`, so it must be `await using` — a plain `using` does not compile, and `using (var t = transactionScope.BeginAsync(ct))` without `await` disposes the `Task` instead of the transaction:
```csharp
await using (await transactionScope.BeginAsync(ct))
{
    try
    {
        // domain service operations...
        await transactionScope.CommitAsync(ct); // commits and executes SaveChangesAsync
    }
    catch
    {
        await transactionScope.RollbackAsync(CancellationToken.None); // a cancelled ct must not skip the rollback
        throw;
    }
}
```
