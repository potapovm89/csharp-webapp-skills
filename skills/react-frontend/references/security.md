# Frontend Security Reference

## Authentication (OIDC + PKCE)
- Public client in Keycloak: Standard Flow (Authorization Code) with PKCE `S256`, no client secret in the SPA, exact redirect URIs and web origins (no wildcards in non-local environments).
- `oidc-client-ts` with `userStore: new WebStorageStateStore({ store: new InMemoryWebStorage() })` — tokens stay in memory; refresh through silent renew / refresh token rotation. A page reload re-authenticates silently via the Keycloak session.
- Never put access or refresh tokens in `localStorage`, `sessionStorage`, cookies set by JS, URLs or logs.
- The audience/scopes requested match what the API validates (`security` skill: `ValidateAudience`).
- UI checks of roles only hide controls; every permission is enforced by the API.
- **BFF option** (preferred for apps handling payments/personal data): an ASP.NET Core backend-for-frontend performs the code flow, keeps tokens server-side and gives the browser an `HttpOnly; Secure; SameSite=Strict` session cookie; the SPA calls the BFF with anti-forgery protection.

## XSS
- Rely on React escaping; `dangerouslySetInnerHTML` is prohibited unless the HTML is sanitized with DOMPurify (Apache-2.0/MPL) and the reason is documented.
- Never build URLs for `href`/`src` from user input without validating the scheme (`https:` / relative only; block `javascript:`).
- Do not use `eval`, `new Function`, or inject scripts dynamically.
- Third-party scripts (analytics, widgets) only after approval, with Subresource Integrity when loaded from a CDN.

## Content Security Policy
Serve the SPA with headers (from the host or ingress):

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:;
  connect-src 'self' https://<api-host> https://<keycloak-host>; frame-ancestors 'none'; base-uri 'self'; form-action 'self'
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
```

- No `'unsafe-inline'` / `'unsafe-eval'` for scripts; configure the build so it does not need them.
- `index.html` is served with `Cache-Control: no-cache`; hashed assets with long-term caching.

## Dependencies & Configuration
- `npm audit --audit-level=high` (or `pnpm audit`) fails the pipeline on high/critical vulnerabilities.
- Renovate/Dependabot keep dependencies current; lock file changes are reviewed.
- `.env*` files with anything but public values are never committed; the bundle contains no secrets — everything in it is public.
- Source maps are not served publicly in production (upload to the error tracker instead).
