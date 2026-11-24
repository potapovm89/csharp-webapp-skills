---
name: csharp-code-style
description: >-
  Use this skill when writing, reviewing, or refactoring C# / .NET 10 code. Covers naming conventions, code layout, mandatory XML documentation, engineering principles (SOLID, KISS, DRY, YAGNI), and library licensing restrictions.
  Triggers (RU): код-стайл, именование, XML-документация, лицензии библиотек, маппинг, AutoMapper, время, деньги, культура.
---

# C# Code Style & Guidelines

All C# code in the solution follows Microsoft C# Coding Conventions with strict project-specific rules. Ensure that an [.editorconfig](./references/.editorconfig) settings file is included in the project.

## Core Rules

1. **Naming Conventions**:
   - Solution name: kebab-case (e.g. `order-service`).
   - Projects: `Itl.<Project>.<Role>` in PascalCase (e.g. `Itl.Orders.Stories`, `Itl.Orders.WebHost`).
   - Constants: `UPPER_CASE_WITH_UNDERSCORES`.
   - Classes, records, structs, methods, properties: `PascalCase`.
   - Private and protected fields: `_camelCase` starting with an underscore.
   - Test methods: `[TestedMethod]_[Condition]_[ExpectedResult]`.
   - See [naming-conventions.md](./references/naming-conventions.md) for full details.

2. **Mandatory XML Documentation**:
   - Language: XML documentation is written in **Russian**; log messages and code identifiers are in **English**.
   - Every public element of interfaces, controllers, commands/queries, domain entities, and DTOs MUST have XML documentation (`/// <summary>...`).
   - Every XML description **MUST end with a period (`.`)**.
   - For interface implementations, use `/// <inheritdoc />`.

   ```csharp
   /// <summary>
   /// Таблица с информацией о покупателях.
   /// </summary>
   [Table("customers")]
   public class Customer : NumericKeyEntity, ICanBeSoftDeleted
   {
       /// <summary>
       /// Имя покупателя.
       /// </summary>
       [Column("name")]
       public required string Name { get; set; }

       /// <inheritdoc />
       [Column("is_deleted")]
       public bool IsDeleted { get; set; }
   }
   ```

3. **Software Principles**:
   - Follow SOLID, KISS, DRY, and YAGNI. See [principles.md](./references/principles.md).

4. **Library Licensing Constraints (CRITICAL)**:
   - Prohibited **for libraries** (anything linked into our process or build output): commercial licenses, GPL/AGPL, and source-available licenses (SSPL, BUSL, Elastic).
   - **Standalone services** (separate process/container, used unmodified, accessed only over a protocol) under these licenses are allowed after approval — e.g. Grafana, MongoDB Server, Garage. See "Libraries vs. Standalone Services" in libraries.md.
   - Version caps due to license changes:
     - `FluentAssertions`: **≤ 7.2.0**
     - `MediatR`: **≤ 12.5.0**
     - `PDFsharp` / `MigraDoc`: **≤ 6.2.x** (6.5.0+ is AGPL)
   - HTML → PDF: use `PeachPDF`; QuestPDF, iText and other commercial/AGPL PDF libraries are prohibited.
   - **`AutoMapper` is prohibited in any version.** Do not add it, do not suggest it, and remove it when you touch code that still uses it.
   - See [libraries.md](./references/libraries.md).

5. **Object Mapping**:
   - Map manually with static extension methods `To<Target>()` in `Itl.<Project>.Common/Mappers/`, one static class per area (`<Area>MappingExtensions`).
   - Assign every target property explicitly. Properties that must stay unset (identifiers, foreign keys filled after save) are documented in the method's `<remarks>`.
   - Collections: `source.Items?.Select(x => x.ToItem()).ToList() ?? []` — never return `null` collections.
   - Never inject a mapper service into handlers, stories or hosts; call the extension method directly.
   - Every mapping method is covered by a unit test (see `testing`).
   - Do not replace AutoMapper with another runtime/reflection mapping library (Mapster, ExpressMapper, etc.) without explicit approval.
   - **All** mappings live in `Common/Mappers`, including proto ↔ Contracts DTO (used by WebHost and RpcHost), entity ↔ DTO (used by Stories) and broker message ↔ proto/DTO. Host- or Stories-local `Mappers/` folders and `internal static class XxxMappers` are prohibited: the same conversion then gets written twice and drifts. Exception: the database projection inside a DataAccess `Select(...)` (it must stay an expression EF can translate).
   - Class names: `<Area>MappingExtensions` (`ChannelMappingExtensions`, `MessageMappingExtensions`), `public static`, with XML docs.

   ```csharp
   public static class PaymentMappingExtensions
   {
       /// <summary>
       /// Преобразует запрос на оплату в заказ на оплату.
       /// </summary>
       public static OrderOnPayment ToOrderOnPayment(this PaymentRequestDto source)
       {
           return new OrderOnPayment
           {
               OrderId = source.OrderId,
               Purchases = source.Purchases?.Select(x => x.ToPurchaseOnPayment()).ToList() ?? [],
               FinalPrice = source.FinalPrice
           };
       }
   }
   ```

6. **Time, Money & Culture**:
   - Current time only through an injected `TimeProvider` (`timeProvider.GetUtcNow()`); instants stored in UTC (`timestamptz`), dates as `DateOnly`. `DateTime.UtcNow`, `DateTime.Now`, `DateTimeOffset.UtcNow` are prohibited in Stories, DataAccess, hosts and validators (`.Must(x => x > timeProvider.GetUtcNow())`, not `DateTime.UtcNow`). Register `builder.Services.AddSingleton(TimeProvider.System)` in every host; tests use `FakeTimeProvider`.
   - Money is `decimal` with an explicit currency and explicit rounding; `double` for money is prohibited.
   - Machine-readable formats use `CultureInfo.InvariantCulture`; identifiers are compared with `StringComparison.Ordinal`.
   - See [time-money-culture.md](./references/time-money-culture.md).

7. **Nullable Reference Types & Warnings**:
   - `<Nullable>enable</Nullable>` for every project; warnings are fixed, not suppressed. A full build (`dotnet build --no-incremental`) has 0 warnings; `TreatWarningsAsErrors` is recommended once a solution is clean.
   - See [nullability.md](./references/nullability.md).
