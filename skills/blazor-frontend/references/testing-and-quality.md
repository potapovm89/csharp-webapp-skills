# Blazor Verification and UI Quality

## Component Tests
Use a pinned compatible bUnit version with the existing test framework. Exercise observable behavior rather than private fields or generated markup details:
- A controlled asynchronous service proves loading → success, empty and error states. An empty initialized list is not a loading flag.
- Invalid form input is shown next to labelled fields; server Problem Details maps back to the form, preserving input. Submission is disabled in flight and ambiguous retries keep the original operation key.
- Navigation/disposal cancels the request; cancellation is not shown as a server error. For changing route parameters, late responses cannot replace newer state.
- Auth rendering covers anonymous, permitted and denied users. These tests do not prove server authorization; signed-token/cookie/API integration tests live under `security`.
- Fix culture and time; verify resource fallback and key date/number output. Resolve dependencies through a test provider and await render-state changes, not arbitrary sleeps.

Avoid trivial tests that only repeat simple DTO mapping or localization plumbing. Prioritize asynchronous screens, the HTTP error/cancellation boundary and authorization/CSRF behavior.

## Browser Acceptance
Use the project's Playwright toolchain in an isolated test stand: sign-in/out, denied page/API, main form submission, validation/conflict recovery and session expiry. Include a real configured test IdP for OIDC callback/claims/MFA cases; mock auth alone is not that evidence. Reuse worker-scoped test identities without sharing mutable browser state across tests.

Check keyboard-only use, visible focus/dialog focus return, labelled controls, headings/table headers and an accessibility smoke scan. Record the applicable WCAG 2.2 level; automated scanning alone does not establish conformance. No production browser tests or real payment/email endpoints.

## Rendering and Network Budgets
- Bound list sizes and use pagination/`Virtualize` for large data sets where appropriate; preserve accessible semantics and stable item keys. Do not issue a network request on every render or keystroke without cancellation/debounce.
- Dispose event handlers, timers, JS object references and cancellation sources. Avoid blocking `.Result`/`.Wait()` and `async void` handlers.
- Measure published WASM download/startup and critical screen latency on the target device/network. AOT, trimming and lazy loading are decisions driven by measurements and compatibility, not compulsory optimizations.
- Run client/host build, component tests and applicable browser checks; record unavailable checks honestly. Server readiness or successful compilation does not prove the UI flow.

Sources: [Microsoft component testing](https://learn.microsoft.com/en-us/aspnet/core/blazor/test?view=aspnetcore-10.0), [WCAG 2.2](https://www.w3.org/TR/WCAG22/).
