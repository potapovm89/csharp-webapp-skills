---
name: blazor-frontend
description: >-
  Use this skill when developing hosted Blazor WebAssembly UI: authentication/BFF, components and forms, cancellation, API errors, localization, accessibility, performance and component/E2E tests. Triggers (RU): Blazor, WebAssembly, BlazorHost, Razor-компонент, bUnit, resx, авторизация Blazor.
---

# Blazor Frontend (WebAssembly & Host)

This skill covers the frontend UI development with Blazor WebAssembly and its companion hosting server `BlazorHost`.

## Two-Project Architecture
- `Itl.<Project>.Blazor` (in `Front/`): Client-side WebAssembly app (Pages, Components, Layout, CSS isolation, client services).
- `Itl.<Project>.BlazorHost` (in `Hosts/`): ASP.NET Core host serving WASM static assets, fallback routing (`MapFallbackToFile("index.html")`), and dedicated API controllers.

## Key Rules
1. **Separation of Markup and Logic**: Keep `.razor` for markup; put C# logic in code-behind `.razor.cs` partial classes.
2. **CSS Isolation**: Put component styles in `<ComponentName>.razor.css`.
3. **No Direct HTTP in Pages**: Components invoke domain client services (`Services/<Area>`) which use `IHttpRepository`.
4. **No Business Rules in UI**: Client performs only presentation logic; data mutations and rules are executed by backend controllers and Stories via CQRS.
5. **Authentication**: use the project's explicit BFF or OIDC-PKCE profile. Client role checks are UX, server policies/ownership are authoritative. Read [security.md](./references/security.md) before changing auth, cookies or HTML rendering.
6. **State and failure**: each data screen handles loading, empty, success, validation and failure states. Cancel obsolete requests and dispose subscriptions; pass tokens through services and JSON reads. Retry an ambiguous write only with the same operation's idempotency key/payload.
7. **Localization/accessibility**: visible text goes through resx/IStringLocalizer; dates/numbers use the selected culture. Use semantic controls, labels, keyboard/focus behavior and a stated WCAG target; do not rely on color alone.
8. **Acceptance**: behavior tests with bUnit for state transitions/forms/auth rendering and Playwright for critical browser journeys. Read [testing-and-quality.md](./references/testing-and-quality.md) when changing UI behavior; load/render budgets are defined by `reliability-and-performance`, not invented per component.

## Subtopic References
- [components.md](./references/components.md) — Component layout, parameters, and form handling.
- [localization.md](./references/localization.md) — Localization and `.resx` handling.
- [security.md](./references/security.md) — BFF/PKCE boundaries, cookie/CSRF, server authorization, CSP and safe markup.
- [testing-and-quality.md](./references/testing-and-quality.md) — bUnit, browser journeys, accessibility and bounded rendering/network work.
