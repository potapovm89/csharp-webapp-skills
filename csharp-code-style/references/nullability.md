# Nullable Reference Types Reference

Every project has `<Nullable>enable</Nullable>` (set once in `Directory.Build.props`). A non-nullable reference type is a promise that the value is never `null`; the compiler checks it, and CS86xx warnings are real defects, not noise.

## Declaring Properties
| Kind | Pattern | Why |
| ---- | ------- | --- |
| Value that the caller must provide | `public required string Name { get; set; }` | The compiler forces every object initializer to set it. |
| String with a sensible empty default | `public string Address { get; set; } = string.Empty;` | Never `null`, no ceremony at call sites. |
| Optional value | `public string? MiddleName { get; set; }` | `null` is a valid state; callers must check. |
| Collection | `public ICollection<Order> Orders { get; set; } = [];` | Collections are never `null`. |
| EF navigation to a required parent | `public Customer Customer { get; set; } = null!;` | EF fills it when loaded; `null!` is allowed **only** for EF navigations. |
| Options class property | `[Required] public string Address { get; set; } = string.Empty;` | Configuration binding needs runtime validation: use DataAnnotations + `ValidateOnStart`. C# `required` alone does not reject missing configuration; helpers with a `new()` constraint can also reject required members (CS9040). |
| Test case data set in initializers | `public CreatePaymentCommand Request { get; set; } = null!;` | Allowed in test projects only. |

Changing `= string.Empty` / `= null!` / `= []` initializers does not change the EF model (nullability stays the same), so no migration is needed; changing `string` ↔ `string?` does.

## Handling Possible Nulls
- Results of lookups (`FirstOrDefault`, `Find`, `GetService`, `GetManifestResourceStream`, `Path.GetDirectoryName`, configuration reads) are nullable: handle `null` explicitly or throw a meaningful exception with `?? throw new InvalidOperationException("...")`.
- Resolve services with `GetRequiredService<T>()`; `GetService<T>()` only when the service is genuinely optional, followed by a null check.
- The null-forgiving operator `!` is allowed only where non-null is guaranteed by something the compiler cannot see (after a validator has run, EF navigation) — leave a comment when it is not obvious. Never use `!` just to silence a warning.
- Use `[NotNullWhen(true)]` / `[MaybeNullWhen(false)]` on `TryXxx(out ...)` methods instead of `!` at call sites.
- Generic helpers that pass values to non-nullable APIs constrain the type: `where T : notnull`.
- Event/callback signatures must match the delegate's nullability (e.g. `X509Certificate2? certificate` in `ServerCertificateCustomValidationCallback`).

## Checking
- `dotnet build --no-incremental` — incremental builds skip up-to-date projects and hide analyzer warnings.
- Do not add `#nullable disable`, `<NoWarn>CS8618</NoWarn>` or `<Nullable>disable</Nullable>` to make warnings go away. Generated code (EF migrations, protobuf) is already excluded from analysis.
