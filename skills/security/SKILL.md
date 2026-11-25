---
name: security
description: >-
  Use this skill when adding or reviewing endpoints, authentication and authorization (JWT, Keycloak roles, policies, resource ownership), CORS, rate limiting, security headers, input handling (injection, file uploads, HTML templates, SSRF), personal data (152-FZ / GDPR), service-to-service security, or dependency vulnerability checks.
  Triggers (RU): безопасность, авторизация, аутентификация, Keycloak, JWT, CORS, rate limiting, инъекции, SSRF, персональные данные, уязвимости.
---

# Security

Security rules for every host and library in the solution. They apply to new code and to any existing code you touch; a violation found in passing is fixed or reported to the user.

## Changing Service-to-Service Authorization

1. Before editing, read [the service-to-service rules](./references/authn-authz.md#service-to-service). Check the applicable project policy, accepted ADR, host registration and deployment configuration. Identify who authenticates the caller, which identities can call the changed method, and how tenant delegation is verified.
2. Use the approved enforcement boundary. Application JWT is the reference default. Platform mTLS/workload authorization or a project-specific exception must satisfy the same caller, method and delegation checks. An exception applies only to the services and environments recorded in the decision and only where the governing policy permits it.
3. If the decision or the effective enforcement is unclear, report the mismatch and pause the affected auth change for a decision. Keep the existing controls in force and continue independent work.

**Example** — A project ADR selects pre-shared service tokens for a gRPC gateway with per-method authorization. For a new method, verify the allowlist, credential source and tenant delegation under that profile.

## Non-negotiable Rules

1. **Deny by default.** Every endpoint requires an authenticated user unless it is explicitly marked `[AllowAnonymous]` with a reason in a comment. Health endpoints are the usual exception.
2. **Authorization is checked on the server for every resource.** A role check is not enough: the Story verifies that the current principal owns the object (store, order, customer). Identifiers that define ownership come from the authenticated principal's claims, never from a client-controlled header, query or body field.
3. **401 vs 403.** Not authenticated → 401. Authenticated but not allowed → 403. Not found and "exists but belongs to someone else" both → 404 when existence itself is sensitive.
4. **Never trust input.** Validate with FluentValidation in Stories; parameterize every SQL; HTML-encode user data inserted into HTML (including PDF templates); never build file paths, S3 keys or URLs from raw user input.
5. **No secrets and no personal data in logs, URLs, exceptions or error responses.** See `configuration-secrets` and `observability`.
6. **Least privilege everywhere.** Separate DB users for migrations and for the application; per-service broker credentials; tokens with the narrowest scopes.
7. **Known-vulnerable dependencies fail the build.** High and critical NuGet advisories (NU1903, NU1904) are errors; moderate ones are tracked. `Directory.Build.props` must contain:
   ```xml
   <NuGetAudit>true</NuGetAudit>
   <NuGetAuditMode>all</NuGetAuditMode>
   <NuGetAuditLevel>moderate</NuGetAuditLevel>
   <WarningsAsErrors>$(WarningsAsErrors);NU1903;NU1904</WarningsAsErrors>
   ```
8. **Passwords are not stored when Keycloak can do it.** Local password hashes only with PBKDF2 (600 000+ iterations for SHA-256, 16-byte salt), parameters stored with the hash, rehash on sign-in, constant-time comparison.
9. **Personal data is minimized, protected and deletable.** Store only what the feature needs, mask it in logs, define retention, and keep it in the storage location required by law (152-FZ: personal data of Russian citizens is stored in Russia).
10. **Internal services verify caller identity and permissions.** Enforce the approved service-to-service profile described above. Document and verify caller and method restrictions, protection against direct bypass, tenant/actor delegation and credential rotation. Network location alone establishes no service identity.

## Where Each Rule Is Wired (check the code, not just the skill)
| Rule | Must exist in code |
|---|---|
| 1 | `AddAuthorizationBuilder().SetFallbackPolicy(...RequireAuthenticatedUser())` in application-authenticated hosts; equivalent enforced platform caller policy for explicitly documented platform-authenticated internal endpoints |
| Rate limits | `AddRateLimiter` + `UseRateLimiter` in WebHost |
| 7 | the `Directory.Build.props` block above |
| 10 | Application JWT: `AddAuthentication().AddJwtBearer(...)` + `UseAuthentication/UseAuthorization`, allowed service clients and client token handler. Platform enforcement: protected workload identity, caller/method rules and deployment verification. Approved exception: ADR, effective enforcement and verification of the equivalent controls |

If any row is missing — including in code inherited from the template — add it or report it to the user before finishing the task.

## Subtopic References
- [authn-authz.md](./references/authn-authz.md) — JWT/Keycloak validation, policies, fallback policy, resource ownership, 401/403 mapping, service-to-service.
- [web-hardening.md](./references/web-hardening.md) — security constraints on the pipeline order, CORS, rate limiting, security headers, forwarded headers, request limits, error responses.
- [passwords.md](./references/passwords.md) — when to store passwords at all, PBKDF2 parameters, self-describing hash format and rehash on sign-in, constant-time verification, password policy.
- [input-and-data.md](./references/input-and-data.md) — injection, HTML/PDF templates, file uploads and S3 keys, SSRF, deserialization, personal data, dependency and secret scanning.
- [threat-modeling.md](./references/threat-modeling.md) — assets/trust boundaries, abuse cases and scoped ASVS verification.
- [mfa-and-step-up.md](./references/mfa-and-step-up.md) — MFA assurance, freshness, sensitive operations and recovery/bypass checks.
- [audit-and-verification.md](./references/audit-and-verification.md) — durable security audit, retention, real authentication tests and vulnerability response.

For CI runner permissions, scanner gates, SBOM/provenance and artifact authenticity, use [delivery-and-supply-chain](../delivery-and-supply-chain/SKILL.md); this skill remains the source of application vulnerability/auth/data-protection rules. Backup/drill copies and recovered keys follow the same data-access/classification rules and [data-recovery](../data-recovery/SKILL.md). Neither skill grants production access or permission to disclose data to external tools.

## Review Checklist (security part of every MR)
- [ ] New endpoints: authentication required or `[AllowAnonymous]` justified; policy/role set; ownership checked in the Story.
- [ ] Ownership identifiers (store id, customer id, tenant id) taken from claims, not from client input.
- [ ] Input validated; SQL parameterized; user data HTML-encoded in templates; no user-controlled URLs or paths.
- [ ] No secrets or personal data in logs, exceptions, URLs or responses.
- [ ] Rate limit applied to public and expensive endpoints (login, registration, file generation).
- [ ] No new high/critical NuGet advisories; licenses checked (`csharp-code-style`).
- [ ] Personal data: purpose, retention and masking documented in `docs/`.
- [ ] Changed trust boundaries/sensitive flows have an updated threat/abuse model and applicable verification evidence; MFA/step-up and mandatory audit requirements are explicit rather than inferred from a role or login screen.

## Completion Evidence

- For each changed internal authorization boundary, link the applicable policy or ADR and verify the real configured boundary in an isolated test. For gRPC, a permitted caller succeeds, absent or invalid credentials return `Unauthenticated`, and a valid caller without method permission receives `PermissionDenied`. Use the equivalent HTTP statuses for HTTP.
- If the method accepts tenant or actor delegation, verify that changing a caller-supplied identity cannot grant access to another tenant. For platform enforcement, include a deployment test that verifies direct access without that enforcement is blocked.
- Record the test command and outcomes. A handler-only test may verify a rule but cannot establish that the deployed endpoint enforces it.
