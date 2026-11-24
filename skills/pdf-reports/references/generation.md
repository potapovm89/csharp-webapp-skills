# Generation (Templates, PeachPDF, PDFsharp)

## Template Project and Renderer
- Templates are Razor components in a dedicated Razor class library `Itl.<Project>.Documents` (`Microsoft.NET.Sdk.Razor`, solution folder `Libs/`). It references only `Contracts`: every template takes a typed model (a Contracts DTO) as its single `[Parameter]` — no entities, no services injected besides `IStringLocalizer`, no data access from markup (`solution-architecture`, dependency rules).
- `IDocumentRenderer` lives in `Common` (it references `Documents`) and is registered as a singleton. It renders the component to HTML with `HtmlRenderer` and converts the HTML with PeachPDF; Stories call only `IDocumentRenderer`.

```csharp
public async Task<string> RenderHtmlAsync<TTemplate, TModel>(TModel model, CultureInfo culture)
    where TTemplate : IComponent
{
    await using var renderer = new HtmlRenderer(serviceProvider, loggerFactory);
    return await renderer.Dispatcher.InvokeAsync(async () =>
    {
        CultureInfo.CurrentCulture = CultureInfo.CurrentUICulture = culture;   // the document culture, scoped to this render
        var parameters = ParameterView.FromDictionary(new Dictionary<string, object?> { ["Model"] = model });
        var output = await renderer.RenderComponentAsync<TTemplate>(parameters);
        return output.ToHtmlString();
    });
}
```

- Everything user-supplied is output with Razor `@value`, which HTML-encodes. `MarkupString` is allowed only for rich text stored as sanitized HTML upstream (`security`, input-and-data.md) — the template only places it.
- Logo/branding assets: embedded resources or fetched from S3 by a fixed config key — never by a user-controllable URL (`s3-storage`, presigned rules).

## Layout
- Paged layout is CSS (`@page { size: A4; margin: 20mm }`, repeated table headers via `thead { display: table-header-group }`, page numbers via margin boxes). PeachPDF implements a subset of CSS: every layout feature a template relies on (paging, page breaks, repeated headers, page numbers, fonts) is proven by a template fixture before it is used. If the engine lacks a feature, simplify the layout or switch that document to MigraDoc — do not add a second HTML engine.

## Fonts, Locale, Encoding
- The container image must contain the fonts the templates reference and ICU (`runtime-deployment`, containers.md — fonts/ICU). A missing font renders tofu boxes instead of failing — add a generation smoke test for every supported language.
- Cyrillic/CJK: declare `charset="utf-8"` and a font stack covering the document's languages; test with the longest locale (`blazor-frontend`, localization.md truncation rule applies to PDF layout too).
- Dates, numbers and money use the document's culture (RU documents in ru-RU); the model carries invariant machine values; currency amounts carry an explicit currency and rounding (`csharp-code-style`, time-money-culture.md). Templates never call `ToString()` without a culture-aware format.

## Determinism and Correctness
- Inject `TimeProvider` for any "generated at" stamp; tests use `FakeTimeProvider` — a document with `DateTime.UtcNow` inside is a determinism bug (`testing`, test-data-and-determinism.md).
- Do not assert byte equality: assert the page count, the presence of key strings (extracted text), and totals. A snapshot of raw PDF bytes breaks on every library/font update.
- Keep a fixture per template: one minimal model, one boundary model (empty list, max-length name, decimal rounding edge, a value with `<script>` that must appear as text) — table-driven like validator tests (`testing`, unit-testing.md).
- Totals are computed in the model (decimal, explicit rounding) and rendered — the template never does arithmetic.

## Performance
- The renderer service is a singleton; do not cache documents.
- Generation is CPU-bound: the job dispatching generation uses bounded parallelism (a `SemaphoreSlim` or `MaxDegreeOfParallelism`), not one Task per report (`worker-host-jobs`).
- PeachPDF builds the whole document in memory. Large documents (hundreds of pages / thousands of rows) get an explicit size limit per report type, measured with the load test of the affected journey (`reliability-and-performance`); beyond the limit, split the report into parts or offer CSV instead.

## PDFsharp / MigraDoc (programmatic)
- Version cap ≤ 6.2.x is enforced by the package policy tests (`testing`, architecture-and-contract-tests.md).
- Use for machine-drawn output: barcode labels, tickets with exact geometry, forms with overlay positions. Tables and wrapped text → MigraDoc; raw canvas → PdfSharp.
- Coordinates are in points; money columns align on `decimal` totals computed in the model — same rules as HTML templates.
