# Threat Model and Verification Scope

Before adding a sensitive flow, new public/internal entry point or cross-service trust boundary, update `docs/security/threat-model.md`; reuse the existing model for routine changes. Record assets (data/money/credentials/audit), actors/service identities, data flows and trust boundaries, entry points/dependencies, deployment assumptions and data classification. Reference C4/contracts rather than copying them.

For each material threat or abuse case, link: affected journey → attacker capability/precondition → control and enforcement point → negative test → residual risk/owner. Include cross-tenant IDOR, privilege escalation, replay/double execution, stolen credentials, SSRF through integrations, unsafe uploads, overload and compromised internal callers where applicable. Distinguish application guarantees from infrastructure assumptions that must be verified at deployment.

Use an explicit scoped ASVS checklist, e.g. the applicable OWASP ASVS 5 requirements and chosen assurance level. Record requirement/version, applicability with reason, implementation/evidence and owner; “not tested” is not “not applicable.” Do not invent a compliance level or silently downgrade scope to make a release pass. Product/operations choose business risk acceptance; security-relevant exceptions have approver, expiry and remediation.

An internal RpcHost may rely on platform mTLS/service-identity authorization instead of duplicating JWT logic, provided the workload has no bypass path, the allowed callers/methods are enforced, identity/delegation is protected end to end, and rotation/revocation are tested. A private IP or namespace alone establishes none of those properties. App-only tests cannot prove mesh/NetworkPolicy deployment controls.

Source, reviewed 2026-11-14: [OWASP ASVS 5](https://asvs.owasp.org/). This is a verification framework; the project must select applicable controls and provide evidence, not merely cite it.
