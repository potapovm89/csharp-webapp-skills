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

## Choose the Change and Its Checks

Identify the client and host projects, the affected Razor files, and the existing bUnit/browser test entry points. Follow each row that applies:

| Change | Read before editing | Verify |
|---|---|---|
| Component or data screen | [components.md](./references/components.md) and [testing-and-quality.md](./references/testing-and-quality.md) | Control the async service in a bUnit test. Assert the applicable loading, empty, success and failure states. When the route can change during loading, assert the old request is cancelled and a late response cannot replace the new state. |
| Form or submission | [Forms](./references/components.md#forms-editform) and [testing-and-quality.md](./references/testing-and-quality.md) | Exercise invalid input, a successful submit and an API field error. Verify the visible field messages, preserved input and single submission while a request is in flight. |
| Authentication, session or host security | [security.md](./references/security.md) and [Browser Acceptance](./references/testing-and-quality.md#browser-acceptance) | Read the BFF or direct OIDC profile from the project decision and host registration. Resolve conflicting evidence before changing the boundary. Verify the flow through the browser and the actual server boundary. With cookie-authenticated mutations, include antiforgery rejection. |
| Localized UI | [localization.md](./references/localization.md) | Render under a non-default culture and assert localized text and applicable date or number formatting. |

**Example** — for an `EditForm` that edits a notification template, use a controlled client service to return `ProblemDetails` for the title. Assert the message next to the title field, preserved input, and one service call even if the user clicks submit again while it is pending. Correct the input and verify a successful submission.

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

## Completion Evidence

- Run the project's applicable client and host build commands. Run the relevant bUnit project using its documented command, or `dotnet test` from that test project's directory. Run the existing Playwright suite for a changed critical browser journey or authentication flow.
- Record at least the changed behavior's passing case and its relevant failure case from the table. For authorization changes, verify the API rejects the unauthorized request even if the UI is bypassed. A bUnit rendering test establishes UI behavior only.
- Report the commands, visible outcomes and any checks blocked by missing test configuration or browser infrastructure.
