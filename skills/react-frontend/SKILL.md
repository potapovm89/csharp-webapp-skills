---
name: react-frontend
description: >-
  Use this skill when creating or changing a React + TypeScript frontend (usually in Front/): project structure, API client generated from OpenAPI, server state with TanStack Query, forms with react-hook-form + zod, OIDC login with PKCE, localization, testing with Vitest / Testing Library / Playwright, linting, XSS and CSP, allowed UI libraries. Triggers (RU): фронтенд, React, TypeScript, компонент, форма, страница, SPA, авторизация во фронте, i18n.
---

# React Frontend (TypeScript)

Applies to React SPAs in the solution. Blazor projects use `blazor-frontend` instead. If the existing project already follows other established conventions, keep them consistent and point out deviations instead of rewriting.

**New app checklist** — when creating a React app from scratch, `package.json` must contain from the first commit: an OpenAPI client generator (`openapi-typescript` + `openapi-fetch` or `orval`), `@tanstack/react-query`, `react-hook-form` + `zod`, `oidc-client-ts` (or `react-oidc-context`), `i18next` + `react-i18next`, `msw`, `@playwright/test`, `@microsoft/signalr` when the app consumes realtime pushes (`realtime-signalr`), ESLint (typescript-eslint, react-hooks, jsx-a11y) and Prettier. A hand-rolled replacement (own PKCE, own fetch wrapper with hand-written DTO types, `useState` + `useEffect` data loading) is a deviation that needs the user's approval.

**Never in `localStorage`/`sessionStorage`**: access/refresh tokens, API keys, any other credential. Tokens stay in memory (`oidc-client-ts` with `InMemoryWebStorage`) or server-side (BFF).

## Change an Existing Screen or Form

1. Locate the app's `package.json` and the lockfile for its app or workspace root. Use the same package manager locally, in CI and in the Docker build. If `packageManager` is set, check that it agrees with the lockfile. Resolve any conflict or missing lockfile before installing dependencies.

   | Committed lockfile | Frozen install from its root |
   |---|---|
   | `package-lock.json` | `npm ci` |
   | `pnpm-lock.yaml` | `pnpm install --frozen-lockfile` |

2. If the change calls a new or changed API operation, trace the OpenAPI input through the generator script/config to the import used by that screen. Read [api-and-state.md](./references/api-and-state.md), regenerate using the project's script and use the generated types and client for the changed call. If the generator is missing, add it when in scope or report the missing prerequisite. For a UI-only change, preserve existing API wiring and record any deviation that affects the task.
3. Change the feature's query or mutation, form schema and UI as applicable. Map server validation errors to fields, invalidate affected queries after mutations and localize new visible text. Use the app's test and build scripts to verify the affected flow.

**Example from `it-letter`** — the React app has `package-lock.json` and a hand-written `src/api/client.ts` that calls `fetch`. Use `npm ci` for this lockfile. For a changed API call, check for a generator, its OpenAPI input and the actual import. The existence or name of `client.ts` alone cannot establish that the call uses generated code.

## Core Rules
1. **TypeScript strict**: `"strict": true`, `"noUncheckedIndexedAccess": true`; `any` is prohibited (use `unknown` + narrowing); no `@ts-ignore` without a comment explaining why.
2. **Build**: Vite with Node LTS pinned in `.nvmrc` / `engines`. Commit one lockfile for the app or workspace and use its matching frozen install in pipelines.
3. **API access only through a generated client** from the backend OpenAPI document (`openapi-typescript` + `openapi-fetch`, or `orval`). Hand-written DTO types for backend endpoints are prohibited; regenerate when the contract changes. See [api-and-state.md](./references/api-and-state.md).
4. **Server state** lives in TanStack Query (queries, mutations, invalidation); do not copy it into global stores. Global client state (if any) — React context or Zustand, kept small.
5. **Forms**: react-hook-form + zod schema; the schema is the single source for validation and form types. Server validation errors (ProblemDetails) are mapped back onto fields.
6. **Authentication**: OIDC Authorization Code + PKCE (`oidc-client-ts` / `react-oidc-context`) against Keycloak, or a BFF that keeps tokens server-side (preferred for high-risk apps). Tokens are never stored in `localStorage`. See [security.md](./references/security.md).
7. **Localization**: all user-visible text through `i18next` / `react-i18next`; no hardcoded strings in components; dates/numbers through `Intl` with the user's locale.
8. **Testing**: Vitest + Testing Library for components and hooks (query by role/label, not by CSS class), MSW for API mocking, Playwright for critical end-to-end flows. See [testing-and-quality.md](./references/testing-and-quality.md).
9. **Quality gates**: ESLint (typescript-eslint, react-hooks, jsx-a11y) and Prettier pass with zero warnings; `tsc --noEmit` passes.
10. **Licenses**: same rules as the backend (`csharp-code-style`, libraries.md). Commercial or "pro" UI packages (MUI X Pro/Premium, AG Grid Enterprise, Highcharts, Kendo, DevExtreme, Syncfusion) are prohibited without approval. Check `npm` package licenses before adding (`license-checker`).

## Project Structure

```
Front/<app>/
├── src/
│   ├── api/            # generated client (do not edit) + thin wrappers, query keys
│   ├── app/            # providers, router, layout, auth bootstrap
│   ├── features/<area>/# pages, components, hooks, schemas of one business area
│   ├── shared/         # reusable UI components, hooks, utils
│   ├── i18n/           # locales/<lang>/<namespace>.json
│   └── main.tsx
├── tests/e2e/          # Playwright
├── .env.example        # public config only (VITE_*), no secrets
└── vite.config.ts
```

- Feature folders own their code; `shared/` must not import from `features/`.
- One component per file, `PascalCase.tsx`; hooks `useXxx.ts`; no default exports except route modules if the router needs them.
- Runtime configuration (API URL, OIDC authority, client id) is loaded from `/config.json` served by the host, so one build works in every environment. `VITE_*` variables are public — never put secrets there.

## References
- [api-and-state.md](./references/api-and-state.md) — client generation, query keys, mutations and invalidation, ProblemDetails handling, forms.
- [security.md](./references/security.md) — OIDC PKCE, token storage, BFF option, XSS, CSP, dependency audit.
- [testing-and-quality.md](./references/testing-and-quality.md) — Vitest, Testing Library, MSW, Playwright, accessibility, ESLint/Prettier settings.

## Completion Evidence

- Record the lockfile, package manager and frozen install command. Check that applicable CI and Docker commands use the same choice. The frozen install must leave the lockfile unchanged.
- For a new or changed API call, record the generator command, OpenAPI input and generated import. Verify that generation is reproducible from that input and that the changed screen uses the generated contract for the request and response. An unrelated generated file does not satisfy this check.
- Run the project's build and relevant tests. Run the typecheck separately if the build does not include it. For a changed form that submits to an API, cover success and server validation failure. Record commands and results, including any checks that could not run.
