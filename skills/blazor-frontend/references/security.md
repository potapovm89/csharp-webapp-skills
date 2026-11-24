# Hosted WebAssembly Security

## Authentication Profile
- Record one profile in the project's ADR. **BFF:** the host performs OIDC authorization-code/PKCE login, validates issuer/audience/lifetime/signature, and creates an HttpOnly, Secure session cookie. Tokens remain server-side; if retained, use a protected server-side ticket/token store with the required replica/rotation behavior. Never serialize tokens in auth-state JSON, rendered HTML or JS storage.
- **Direct SPA OIDC:** use the supported OIDC client/library and exact redirect origins, no client secret in WASM, and documented in-memory token lifecycle. Do not assume library-default local/session storage meets the token policy. A BFF is preferred for sensitive hosted applications; changing an existing profile needs an explicit decision.
- UI auth state contains only necessary display/permission claims and is refreshed after login/logout/session expiry. `AuthorizeRouteView`/`AuthorizeView` are presentation controls. Every API, file download and mutation enforces server authorization and ownership (`security`).

## Cookies and CSRF
- Cookie-authenticated unsafe requests validate an antiforgery token, including logout and multipart uploads. Obtain a request token from the same-origin host; keep it in memory and send it in the configured header. Do not retry a rejected mutation without resolving its outcome.
- Choose SameSite with OIDC redirects/correlation in mind; a blanket Strict policy on all OIDC cookies can break sign-in. Require Secure outside explicit local test/dev environments. Persist/protect data-protection keys for deployed replicas and define cookie/session revocation/expiry.
- API authentication failures return 401/403 rather than an HTML sign-in redirect. Login/return URLs are local/allowlisted and logout is POST with CSRF protection. Auth-state/token responses use `Cache-Control: no-store`.

## Host and Markup
- Serve the static application shell without sensitive data; explicit anonymous shell/fallback access does not make API endpoints anonymous. Disable WASM debugging in production and configure static web assets for actual Local/Aspire environments.
- Default Razor text rendering encodes data. `MarkupString`/raw HTML requires an approved sanitizer and explicit use case. Validate href/src schemes; do not turn user strings into executable code or arbitrary outbound URLs.
- Define CSP for the selected .NET/browser runtime; permit only required sources and WASM execution capability, avoid broad script `unsafe-inline`/`unsafe-eval`. Test the published build under the policy, including error reporting and login callbacks. A CSP header is not proof the UI still works.
- No secrets in WASM config/resources/source maps. Public configuration contains only client-safe values. Use `security` for uploads/SSRF and `configuration-secrets` for server keys.

References, reviewed 2026-11-14: [Microsoft Blazor security](https://learn.microsoft.com/en-us/aspnet/core/blazor/security/?view=aspnetcore-10.0), [OIDC-hosted application guidance](https://learn.microsoft.com/en-us/aspnet/core/blazor/security/blazor-web-app-with-oidc). Match the guidance to hosted standalone WASM versus Blazor Web App; do not copy prerender/circuit APIs into the wrong hosting model.
