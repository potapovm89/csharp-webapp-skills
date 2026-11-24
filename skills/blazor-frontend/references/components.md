# Blazor Components Reference

## Component File Structure
- `.razor`: Markup, directives (`@page`, `@inject`).
- `.razor.cs`: Code-behind `partial class` containing lifecycle methods (`OnInitializedAsync`), state, and event handlers.
- `.razor.css`: Component-scoped CSS isolation.

## Guidelines
- Pages use `@page "/kebab-case-route"`.
- Never call `HttpClient` directly from components: inject specialized client services (`Services/<Area>`) backed by `IHttpRepository`.
- Component parameters use `[Parameter]`; mandatory ones use `[EditorRequired]`.
- All business calculations and validation belong on the server/Stories, not in client markup.

## Parameters & Events
- `[Parameter]` properties are set by the framework: never mutate them inside the component; derive display state in `OnParametersSet`.
- A mandatory parameter is `[Parameter, EditorRequired]` and is checked in `OnInitialized` (`ArgumentNullException.ThrowIfNull`) — not suppressed with `!`.
- Child→parent communication is `EventCallback<T>` (dispatches to the render context automatically), not `Action<T>`.
- `CascadingValue` carries cross-cutting state only (current user, theme); entities are not cascaded through several levels — pass them explicitly per component.

## Data Screens
Every data screen implements the same five states: loading, empty, success, validation error, failure. Standard shape:

```csharp
private bool isLoading = true;
private string? loadError;
private IReadOnlyList<OrderDto> orders = [];

protected override async Task OnInitializedAsync()
{
    try
    {
        orders = await OrdersClient.GetOrdersAsync(ct); // domain client service, not HttpClient
        loadError = null;
    }
    catch (Exception ex) when (ex is not OperationCanceledException)
    {
        loadError = Localizer["Orders.LoadError"]; // localized; details are not shown to the user
        Logger.LogError(ex, "Failed to load orders");
    }
    finally
    {
        isLoading = false;
    }
}
```

- Cancel obsolete requests: own a `CancellationTokenSource` per load, cancel the previous one on reload, dispose it in `Dispose()`. Client-side cancellation is not a correctness mechanism: the server still executes an accepted write, so an ambiguous mutation is retried only with the same operation's idempotency key (SKILL.md rule 6).
- JS interop goes through `OnAfterRenderAsync(firstRender)` — never in `OnInitialized`.

## Forms (EditForm)
- `EditForm` + `Model` + `OnValidSubmit`/`OnInvalidSubmit`; `<DataAnnotationsValidator />` covers client-side shape checks. Business rules stay in the Story (`cqrs-stories`) — client validation is UX only and never the authorization boundary.
- Map server validation errors (`ProblemDetails`) onto form fields via a `ValidationMessageStore`:

```csharp
private EditContext? editContext;
private ValidationMessageStore? messageStore;

// ValidationProblemDetails (RFC 9457 + "errors"); plain ProblemDetails has no Errors property.
private void ApplyServerValidation(ValidationProblemDetails problem)
{
    if (editContext is null) return;
    messageStore ??= new ValidationMessageStore(editContext);
    messageStore.Clear();
    foreach (var (field, errors) in problem.Errors)
    {
        messageStore.Add(editContext.Field(field), errors);
    }
    editContext.NotifyValidationStateChanged();
}
```

- Disable the submit button while the request is pending and re-enable after the result: this is UX protection only — the duplicate-write guarantee lives in the Idempotency-Key contract (`web-host-api`, api-evolution.md).
- Never place identifiers that define ownership in hidden form fields: they come from claims on the server (`security`).
- Long forms: keep per-step state in a single model class owned by the code-behind; child components receive `[Parameter]` slices and raise `EventCallback<ModelChangedEventArgs>`.

## Disposal
- Components holding subscriptions, timers, `CancellationTokenSource` or JS interop handles implement `IDisposable`/`IAsyncDisposable` and release them explicitly.
- Injected services are disposed by the framework; do not dispose them manually.
