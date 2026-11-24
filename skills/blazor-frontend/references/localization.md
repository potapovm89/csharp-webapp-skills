# Blazor Localization Reference

## Resource Files
- Store localized strings in `.resx` resource files next to the feature (`Features/Orders/Resources/Orders.resx`, `Orders.ru.resx`) or shared in `Resources/SharedResource.resx`; the generated designer file is committed.
- Access via `IStringLocalizer<SharedResource>` / `IStringLocalizer<Orders>` injected into the component or code-behind.
- The invariant resource is the fallback: every new key ships in the default culture in the same MR that uses it; a key added to `.ru.resx` without the default resource is a bug.
- Key naming: dot-namespaced PascalCase (`Orders.DeleteConfirmationTitle`); a composite sentence is one key with placeholders (`{0}`), never concatenated fragments.
- UI strings are never hardcoded in markup. Strings that come from the backend (domain errors, catalog data) are localized on the backend — they are not duplicated in `.resx`.

## Culture Selection (WebAssembly)
- Register `builder.Services.AddLocalization();` in the client `Program.cs`.
- The selected culture is persisted client-side and applied at startup before the first render; a `?culture=` query parameter wins over the stored value. Switching the language updates the stored culture and reloads the app (the standard Blazor culture-controller pattern from the aspnetcore-10.0 docs).
- `CultureInfo.DefaultThreadCurrentCulture` / `DefaultThreadCurrentUICulture` are set once at startup; the layout exposes a culture switcher, and server-authoritative data carries its own culture contract (`csharp-code-style`, time-money-culture.md).

## Formatting & Plurals
- Dates, numbers and money are formatted with the current culture (`@order.CreatedAt.ToString("D", CultureInfo.CurrentCulture)`, `@bind-Format`); hardcoded separators in string interpolation are prohibited.
- `.resx` has no plural rules. Model plural forms as separate keys (`Orders.ItemsOne`, `Orders.ItemsFew`, `Orders.ItemsMany` — Russian requires three forms) selected by a small count helper; `"{0} item(s)"` approximations are prohibited.

## Acceptance
- Switching the language renders every visible screen without mixed languages; a bUnit test renders key screens under a non-default culture — a missing key renders the key name itself, which fails the assertion.
- Check the longest locale for truncation (buttons, table headers, badges): Russian/German strings are typically 20–30% longer than the invariant source.
- Date/number formatting is verified with a pinned culture in tests (`testing`, test-data-and-determinism.md).
