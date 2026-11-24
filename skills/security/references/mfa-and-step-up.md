# MFA and Sensitive-Operation Assurance

Use the existing IdP (Keycloak) for authenticator enrollment, challenges, recovery and revocation; do not implement another OTP/password store in an application. The threat model identifies actions requiring MFA/recent step-up (for example privilege changes or high-impact financial actions) and sets acceptable assurance/freshness. Avoid imposing a universal numeric freshness limit without a business decision.

On the server, validate the trusted token/session first, then enforce the IdP's configured assurance claims (`acr`/`amr` or a documented equivalent) and authentication freshness (`auth_time` where supplied by that IdP). A role, the mere presence of any `amr`, or a new access token's `iat` does not prove MFA or recent user authentication. Document exact claim values/issuer mapping; do not guess that a provider's generic “level 2” means MFA. Client-credentials tokens describe a service, not a human MFA event.

If assurance is missing/stale, return a defined step-up-required error/challenge and keep the unsafe operation unexecuted. Bind the resumed request to the authenticated actor, intended operation and original idempotency semantics; do not trust a client-provided “MFA passed” flag. For federated/downstream calls, verify how human delegation/assurance survives each hop.

Test real IdP login/callback and tokens for no MFA, approved MFA, stale step-up, wrong issuer/audience, expired/replayed credentials and absent/unknown assurance claims. Include enrollment reset, account recovery and alternative login paths: they must not silently bypass required assurance. Revoke/terminate sessions after relevant credential/security changes according to the project's policy.

Record which controls the application implements and which require a configured IdP test stand. A unit test with fabricated claims only proves the application's policy predicate, not that Keycloak actually emits those claims or prevents recovery bypass.

Reference: [OWASP ASVS authentication requirements](https://github.com/OWASP/ASVS/blob/master/5.0/en/0x15-V6-Authentication.md), reviewed 2026-11-14.
