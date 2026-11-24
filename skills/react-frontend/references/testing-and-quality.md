# Frontend Testing & Quality Reference

## Unit and Component Tests (Vitest + Testing Library)
- Test files next to the code: `PaymentForm.test.tsx`.
- Query by what the user sees: `getByRole("button", { name: /pay/i })`, `getByLabelText(...)`; `data-testid` only as a last resort; never CSS classes.
- Use `userEvent` (not `fireEvent`) for interactions; `await findBy...` for async UI.
- API mocking with MSW (MIT) handlers typed from the generated schema; no mocking of `fetch` by hand.
- Wrap components in a test provider (QueryClient with `retry: false`, i18n with a fixed language, router).
- Pin time with `vi.useFakeTimers()` / `vi.setSystemTime(...)` and the locale with an explicit `Intl` locale, so tests do not depend on the machine.
- Cover: form validation and server error mapping, loading/empty/error states, permission-dependent rendering, custom hooks with logic.

## End-to-End Tests (Playwright)
- Only critical user journeys (sign in, main business flow, payment); keep the suite small and stable.
- Run against the Aspire environment or a test stand, never against production.
- Locators by role/label; no fixed `waitForTimeout` — rely on auto-waiting and web-first assertions.
- Authenticate once per worker with a test user and reuse `storageState`.
- Accessibility smoke check with `@axe-core/playwright` (MPL-2.0, test-only dependency) on key pages.

## Accessibility
- Semantic HTML first (`button`, `label`, headings); every input has a label; images have `alt`.
- Keyboard navigation works for all interactive elements; focus is visible and managed in dialogs.
- `eslint-plugin-jsx-a11y` errors are not suppressed without a reason.

## Lint & Format
- ESLint flat config: `typescript-eslint` (recommended-type-checked), `eslint-plugin-react-hooks`, `eslint-plugin-jsx-a11y`, `eslint-config-prettier`.
- Rules worth enforcing: `no-explicit-any`, `react-hooks/exhaustive-deps`, `no-floating-promises`, `consistent-type-imports`.
- Prettier with a committed config; formatting is never discussed in review.
- Scripts: `lint`, `typecheck` (`tsc --noEmit`), `test`, `test:e2e`, `build` — all must pass before merge.
