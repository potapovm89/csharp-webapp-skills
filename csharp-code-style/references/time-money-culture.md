# Time, Money & Culture Reference

## Time
- Inject `TimeProvider` and call `GetUtcNow()`; `DateTime.Now` / `DateTime.UtcNow` / `DateTimeOffset.Now` are prohibited in business code (they make tests non-deterministic).
- Store and transfer instants in UTC: `DateTimeOffset` (or `DateTime` with `Kind = Utc`) → PostgreSQL `timestamptz`. Npgsql rejects non-UTC `DateTime` for `timestamptz` — do not enable the legacy timestamp behavior switch.
- Dates without time (birthday, business date) are `DateOnly` → `date`. Time of day is `TimeOnly`.
- A user's local time is converted at the edge (UI / report) with an explicit IANA time zone (`TimeZoneInfo.FindSystemTimeZoneById("Europe/Moscow")`), never with the server's local zone.
- Business deadlines ("end of day in the store's zone") are calculated with the zone from data, not from the server.
- Durations are `TimeSpan`; configuration values for durations are `TimeSpan` too (`"00:05:00"`), not "minutes" integers.
- Do not add fallback or test-only constructors such as `: this(..., TimeProvider.System)`: `TimeProvider` is registered in DI (`AddServiceDefaults`), and tests pass `FakeTimeProvider`. A second public constructor hides a missing registration and keeps the system clock reachable in production code.
- Console tools without DI pass `TimeProvider.System` explicitly from `Program` (composition root).
- Enforced by an architecture test (`testing`, architecture-and-contract-tests.md).

## Money
- Amounts are `decimal` (`numeric(18,2)` or the precision the currency needs); `double`/`float` for money are prohibited.
- An amount without a currency is ambiguous in multi-currency systems: keep `Currency` (ISO 4217 code, `char(3)`) next to the amount, or document that the system is single-currency.
- Rounding is explicit and in one place: `Math.Round(value, 2, MidpointRounding.ToEven)` (banking) or `AwayFromZero` — as the business requires; never rely on default formatting to round.
- Split amounts (installments, discounts across lines) so the parts sum exactly to the total; assign the remainder to the last part.
- Calculate on the server; the client's price is never trusted.

## Culture
- Machine-readable formats (JSON, CSV for integrations, file names, cache keys, SQL, logs, URLs) use `CultureInfo.InvariantCulture`: `value.ToString(CultureInfo.InvariantCulture)`, `decimal.Parse(text, CultureInfo.InvariantCulture)`.
- Human-readable output (UI, PDF, e-mail) uses an explicit culture from the user/request, not the server default.
- String comparison: `StringComparison.Ordinal` / `OrdinalIgnoreCase` for identifiers and keys; culture-aware comparison only for user-visible sorting.
- `ToUpper()`/`ToLower()` without culture are prohibited for identifiers; use `ToUpperInvariant()`.
- Containers often run with invariant globalization; do not assume a culture is installed — load the ICU package in the Dockerfile if culture-specific formatting is required.
- Tests pin the culture (see `testing`, test-data-and-determinism.md).
