# Test Data & Determinism Reference

A test that passes on one machine and fails on another (or at midnight, or in parallel) is a bug in the test.

## Time
- Production code takes `TimeProvider` (registered as `TimeProvider.System`) instead of `DateTime.UtcNow` / `DateTimeOffset.UtcNow`.
- Tests use `FakeTimeProvider` (`Microsoft.Extensions.TimeProvider.Testing`, MIT): set the start time explicitly, move it with `Advance(...)`.
- Integration tests replace the registration in `ConfigureTestServices`: `services.AddSingleton<TimeProvider>(fakeTime)`.
- Never `Task.Delay` / `Thread.Sleep` to wait for a time-based effect; advance the fake clock. For truly asynchronous effects (broker consumption) poll with a timeout helper.

## Culture
- Pin the culture once per test assembly with a module initializer, so messages and number formats do not depend on the machine locale:

```csharp
internal static class TestCultureInitializer
{
    [ModuleInitializer]
    internal static void Initialize()
    {
        CultureInfo.DefaultThreadCurrentCulture
            = CultureInfo.DefaultThreadCurrentUICulture
            = CultureInfo.InvariantCulture;
        CultureInfo.CurrentCulture = CultureInfo.InvariantCulture;
        CultureInfo.CurrentUICulture = CultureInfo.InvariantCulture;
        ValidatorOptions.Global.LanguageManager.Culture = new CultureInfo("en");
    }
}
```

- Tests that check localized output set the culture explicitly inside the test.

## Test Data Builders
- One builder per aggregate in `tests/.../Builders/` with valid defaults; the test overrides only the fields its scenario is about:

```csharp
var payment = new PaymentBuilder()
    .WithStoreId(storeId)
    .WithStatus(PaymentStatus.Pending)
    .Build();
```

- Unique values (ids, order numbers, e-mails) are generated per test (`Guid.NewGuid()`, a counter) so parallel tests never collide on unique indexes.
- Do not copy the same literal into Arrange, request JSON and Assert; keep it in one variable.
- Random data (Bogus, MIT) only with a fixed seed, and never for the values the assertion depends on.

## Database State
- Each test creates the data it needs; no shared seed that tests mutate.
- Reset between tests with Respawn (Apache-2.0) against the Testcontainers database, or isolate by a fresh database per test class (`GetConnectionStringForDatabase`).
- Never rely on test execution order; xUnit runs classes in parallel unless they share a collection. Ordering extensions are prohibited.
- Tests that use global resources (advisory lock keys, Quartz scheduler, a broker topic) go into a sequential collection or use unique names per test.

## Asynchronous Assertions
- Broker and background effects: poll the observable side effect (DB row, DLQ message) with a timeout (e.g. 10 s) and a short interval; fail with a clear message on timeout.
- Assert "nothing happened" by first producing a marker that must be processed after the message under test, then checking the absence.

## Flaky Tests
A test that fails intermittently without a code change is a defect, not weather:
1. **Quarantine with coverage preserved**: mark the test (`[Trait("Category", "Flaky")]` or the project's equivalent), assign an owner and keep running it in a scheduled job. If it is the only evidence for a required release criterion, keep promotion blocked until an equivalent check passes for that exact artifact digest or an approved, applicable, time-bound exception is in force. A scheduled run against another artifact cannot satisfy the gate. Verify that a missing or expired exception still blocks promotion.
2. **Fix the cause in the sprint.** Typical roots: real clock usage (`FakeTimeProvider`), a parallel collision on shared data (unique values per test, sequential collection), an async race (poll helper instead of sleep), order dependence, culture leakage.
3. **Never "fix" flakiness by retry-until-green in CI**: retries hide the race. A retry attribute may exist only for documented, bounded external flakiness (e.g. real container startup), never for the assertion under test.
4. Deletion or rewrite of a flaky test goes through the same review as adding a test; the fix commit references the task and names the root cause.
