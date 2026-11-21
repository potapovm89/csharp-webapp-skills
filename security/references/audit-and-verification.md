# Security Audit and Authentication Verification

## Mandatory Audit
Classify audit events separately from optional diagnostic logs. For sensitive actions define actor/service identity, action, resource identifier/type, time from `TimeProvider`, outcome/reason code, correlation/operation id and tenant scope. Derive actor/tenant from trusted authentication/delegation; never accept an arbitrary request `UserName` as the authoritative actor. Avoid tokens, passwords, raw payloads and unnecessary personal data.

Required audit of a committed change is stored atomically with that change or durably queued in its outbox; a catch-and-log HTTP/gRPC call after commit is not reliable audit. Define behavior when required recording cannot be made durable (normally do not commit the protected change). Failed/denied attempts have an explicit capture, rate-limit and outage policy so audit cannot become a DoS primitive.

Restrict audit readers/writers, tenant access, mutation/deletion and retention; document clock integrity, tamper-evidence/export requirements, backup/recovery and sanctioned expiry/erasure. Use append-oriented storage and explicit correction events rather than silently rewriting history. An operational Mongo audit demo is not automatically a legally sufficient security journal.

## Real Authentication Tests
Keep mocked auth for ordinary business integration tests, and add a separate suite using the actual authentication middleware and cryptographically signed tokens or real cookies:
- Valid token/session accepted; missing, malformed, wrong-signature, wrong-issuer/audience and expired tokens rejected.
- Authenticated wrong role/service/tenant denied; required ownership and service allowlists enforced at the protected entry point.
- API cookie failures remain 401/403; unsafe cookie requests require CSRF, including logout; session expiry/revocation and return URL validation are checked.
- JWKS/IdP key rotation and metadata/TLS failures are checked in an IdP test stand. An in-memory trusted test signing key proves validation logic, not discovery, Keycloak role mappings or deployed mTLS.
- Sensitive flows test assurance/freshness (`mfa-and-step-up.md`) and durable audit failure/replay. Verify no secrets leak in denied/error responses or test reports.

## Vulnerability Response
Maintain finding → affected package/artifact → exposure → owner → remediation deadline → fix/verification evidence. Set response deadlines by severity/exploitability and service risk in project policy; do not invent uniform time limits. Re-evaluate supported artifacts when advisories change. High/critical dependency gates and time-bound exceptions follow `delivery-and-supply-chain`; a suppression is not a fix.

Keep the real-authentication suite separate from mocked-auth business tests and document its boundary in `docs/security/`. Tests must state their boundary and unavailable evidence; a green mocked-auth suite is not proof of secure authentication.
