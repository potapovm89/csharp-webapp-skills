---
name: configuration-secrets
description: >-
  Use this skill when defining or reading configuration options (AddOptionsWithValidation), managing environments (Local, Aspire, Development, Stage, Production), or verifying secret protection and leak prevention rules.
  Triggers (RU): настройки, Options, appsettings, секреты, переменные окружения, среды, фича-флаги.
---

# Configuration & Secret Management

This skill covers the strongly typed options pattern, environment management, and confidential secret protection in the solution.

## Strongly Typed Options Pattern

All application settings are defined as typed classes with suffix `Options` in `Options/`:
- **Section Name** must exactly match the class name (`nameof(MyOptions)`).
- **Mandatory XML docs** on every property, ending with a period.
- **Validation**: Options use DataAnnotations (`[Required]`, `[Range]`, `[MinLength]`) and non-null defaults. C# `required` is not runtime configuration validation; do not use it for Options properties.
- **Time Spans** defined as `TimeSpan` (parsed from string format `"00:05:00"`), never bare integers.

```csharp
/// <summary>
/// Настройки подключения к RabbitMQ.
/// </summary>
public class RabbitMqOptions
{
    /// <summary>
    /// Название очереди.
    /// </summary>
    [Required]
    public string Queue { get; set; } = string.Empty;

    /// <summary>
    /// Название Exchange.
    /// </summary>
    [Required]
    public string Exchange { get; set; } = string.Empty;
}
```

## Registration with Validation
Options MUST be registered using `AddOptionsWithValidation<TOptions>` from `Common`:
```csharp
builder.Services.AddOptionsWithValidation<RabbitMqOptions>(
    builder.Configuration.GetRequiredSection(nameof(RabbitMqOptions)).Bind);
```
`GetRequiredSection` guarantees startup failure if the section is missing. Validation errors surface immediately at application boot (`ValidateOnStart`).

Test startup with a missing section, missing/blank required values and out-of-range values. `GetRequiredSection` does not validate the section's properties; the helper must apply DataAnnotations and `ValidateOnStart`.

## Consuming Options
- Inject `IOptions<TOptions>` into constructor.
- An option property that no code reads is removed together with its configuration keys; dead settings mislead operators.
- The options class itself is **not** registered in DI: `GetService<TOptions>()` returns `null`. In factories (`AddHttpClient((sp, client) => ...)`, `AddSingleton(sp => ...)`) use `sp.GetRequiredService<IOptions<TOptions>>().Value`.
- Options properties are non-nullable with defaults and validation (`[Required] public string Address { get; set; } = string.Empty;`), not `required` members — see `csharp-code-style`, nullability.md.
- Direct reading of `IConfiguration` in business code is prohibited.
- Connection strings are accessed only via `GetConnectionString("<name>")`.

## Subtopic References
- [environments.md](./references/environments.md) — Environments (`Local`, `Aspire`, `Development`, `Stage`) and settings hierarchy.
- [secret-protection.md](./references/secret-protection.md) — Zero-leak rules and environment variable overrides.
- [feature-flags.md](./references/feature-flags.md) — Microsoft.FeatureManagement, flag naming, where flags are checked, flag lifecycle and kill switches.
