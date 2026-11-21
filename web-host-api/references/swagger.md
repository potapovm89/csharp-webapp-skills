# OpenAPI Configuration Reference

Default stack: the built-in `Microsoft.AspNetCore.OpenApi` package (OpenAPI 3.1 documents by default on .NET 10, JSON Schema draft 2020-12). Swashbuckle is not required for generation; it may remain only as a UI provider or a legacy exception (see below). Configured via `AddSwaggerConfiguration` and `UseSwaggerConfiguration` in `WebHost/Extensions/`.

## Document Setup
- Multi-document setup: User API (`user-api`) and Admin API (`admin-api`) — one `AddOpenApi("<name>")` per document. Endpoints are routed to documents by group name: controllers declare `[ApiExplorerSettings(GroupName = "user-api")]`; an endpoint without a group lands in **every** document.
- `UseSwaggerConfiguration` maps the endpoints: `app.MapOpenApi()` serves `/openapi/user-api.json` and `/openapi/admin-api.json` (default route `/openapi/{documentName}.json`).
- Commit normalized snapshots as `docs/contracts/api/openapi.user-api.v1.json` and `openapi.admin-api.v1.json`; keep contract tests and frontend generation pointed at the same files. Generate them at build time with `Microsoft.Extensions.ApiDescription.Server` (`OpenApiDocumentsDirectory` + `--file-name openapi.user-api.v1`) instead of copying by hand.
- The OpenAPI endpoint is non-production only, like the UI (`if (!host.Environment.IsProduction())`), and must not be reachable through the ingress.

## Mandatory Configuration

```csharp
builder.Services.AddOpenApi("user-api", options =>
{
    // Inline enum schemas instead of separate components (replacement for UseInlineDefinitionsForEnums)
    options.CreateSchemaReferenceId = type =>
        type.Type.IsEnum ? null : OpenApiOptions.CreateDefaultSchemaReferenceId(type);

    // Map custom types (e.g. TimeSpan -> "00:00:00" string)
    options.AddSchemaTransformer((schema, context, ct) =>
    {
        if (context.JsonTypeInfo.Type == typeof(TimeSpan))
        {
            schema.Type = JsonSchemaType.String;
            schema.Format = "time-span";
        }
        return Task.CompletedTask;
    });

    options.AddDocumentTransformer<BearerSecuritySchemeTransformer>();
    options.AddDocumentTransformer<KeycloakOAuthSchemeTransformer>();
    options.AddOperationTransformer<AuthOperationTransformer>();
    options.AddOperationTransformer<ProblemDetailsOperationTransformer>();
});
```

- `<GenerateDocumentationFile>true</GenerateDocumentationFile>` is mandatory; XML comments from all referenced projects are included in the document (controller/action summaries become operation summaries/descriptions).
- Register the RFC 9457 error schema once and reference it from operations instead of copying it per operation:

```csharp
options.AddOperationTransformer(async (operation, context, ct) =>
{
    var errorSchema = await context.GetOrCreateSchemaAsync(typeof(ProblemDetails), null, ct);
    context.Document?.AddComponent("ProblemDetails", errorSchema);
    operation.Responses ??= new OpenApiResponses(); // collections are nullable in Microsoft.OpenApi 2.x
    operation.Responses["4XX"] = new OpenApiResponse
    {
        Description = "Error",
        Content = new Dictionary<string, OpenApiMediaType>
        {
            ["application/problem+json"] = new OpenApiMediaType
            {
                Schema = new OpenApiSchemaReference("ProblemDetails", context.Document)
            }
        }
    };
});
```

## Security Schemes (mandatory)
Both documents declare a Bearer (JWT) scheme; `user-api` additionally documents the Keycloak Authorization Code flow. Microsoft.OpenApi 2.x types live in the `Microsoft.OpenApi` namespace (`IOpenApiSecurityScheme`, `OpenApiSecuritySchemeReference`); the 1.x `Microsoft.OpenApi.Models` object model from .NET 9 examples does not compile against 2.x.

```csharp
internal sealed class BearerSecuritySchemeTransformer(IAuthenticationSchemeProvider authenticationSchemeProvider)
    : IOpenApiDocumentTransformer
{
    public async Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context, CancellationToken ct)
    {
        var schemes = await authenticationSchemeProvider.GetAllSchemesAsync();
        if (!schemes.Any(s => s.Name == "Bearer"))
        {
            return;
        }

        document.Components ??= new OpenApiComponents();
        document.Components.SecuritySchemes ??= new Dictionary<string, IOpenApiSecurityScheme>();
        document.Components.SecuritySchemes["Bearer"] = new OpenApiSecurityScheme
        {
            Type = SecuritySchemeType.Http,
            Scheme = "bearer",
            BearerFormat = "JWT",
            In = ParameterLocation.Header,
            Description = "Application JWT issued by Keycloak."
        };
    }
}
```

- Keycloak Authorization Code flow (`user-api`): an `OpenApiSecurityScheme` with `Type = SecuritySchemeType.OAuth2` and `Flows.AuthorizationCode` pointing at `<authority>/protocol/openid-connect/auth` and `/protocol/openid-connect/token`, with the scopes the API actually issues. Authority/token URLs come from validated options — never hardcoded.
- The UI signs in through a dedicated **public** Keycloak client (e.g. `<app>-swagger`): Standard Flow only, PKCE `S256` required (`pkce.code.challenge.method`), redirect URIs limited to `/swagger/oauth2-redirect.html` of known hosts, an audience mapper so tokens carry the API's `aud`. Swagger UI uses `OAuthClientId(...)` + `OAuthUsePkce()`; never `OAuthClientSecret(...)` — anything given to Swagger UI is in the browser. The API's own confidential client has no browser flow and no redirect URIs. Verify the flow against a real Keycloak with the imported realm (PKCE enforced, token without secret, accepted by the API).
- Security *requirements* are applied per operation by an `IOpenApiOperationTransformer` that reads endpoint metadata: skip operations carrying `[AllowAnonymous]`, add the `Bearer` (and OAuth2, where applicable) requirement to the rest. The document must honestly show which endpoints are actually open.

## UI
Document generation does not include a UI. Add one for non-production environments only, strictly disabled in Production:
- `Scalar.AspNetCore` (`MapScalarApiReference()`) — UI without a Swashbuckle dependency; or
- `Swashbuckle.AspNetCore.SwaggerUI` pointed at the built-in documents (`c.SwaggerEndpoint("/openapi/user-api.json", "User API")`) — allowed as a UI-only dependency.

## Swashbuckle Legacy
A solution may still generate documents with Swashbuckle. That is acceptable only if: the resolved Swashbuckle major supports Microsoft.OpenApi 2.x (older majors bring Microsoft.OpenApi 1.x and conflict at runtime); custom `IOperationFilter`s written against the 1.x object model are migrated (namespaces, security-requirement keys via `OpenApiSecuritySchemeReference`). Do not mix both generation stacks in one host. New documents use the built-in stack; a migration of the legacy one is a separate task.
