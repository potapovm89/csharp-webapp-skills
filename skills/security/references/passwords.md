# Passwords Reference

## Prefer Not to Store Passwords
User sign-in goes through Keycloak (OIDC); services keep no passwords. Store password hashes in a service only when the requirements explicitly demand local accounts, and record that decision in an ADR.

## Hashing (when local passwords are required)
Use one of:
- **ASP.NET Core Identity `PasswordHasher<TUser>`** (`Microsoft.Extensions.Identity.Core`, MIT) — PBKDF2 with a self-describing format, built-in `PasswordVerificationResult.SuccessRehashNeeded`.
- **`Rfc2898DeriveBytes.Pbkdf2`** with the parameters below, when Identity is not used.

| Parameter | Minimum |
| --------- | ------- |
| Algorithm | PBKDF2-HMAC-SHA256 (or SHA-512) |
| Iterations | 600 000 for SHA-256, 210 000 for SHA-512 (OWASP) |
| Salt | 16 random bytes from `RandomNumberGenerator`, unique per password |
| Hash length | 32 bytes |

Prohibited: plain or reversible storage, unsalted hashes, fast hashes (MD5, SHA-x without key stretching), the obsolete `Rfc2898DeriveBytes` constructors (SYSLIB0060), homemade schemes.

## Format and Upgrades
- Store the parameters with the hash, e.g. `pbkdf2-sha256$600000$<base64 hash>` (salt in its own column or in the same string). Never hardcode the parameters only in code: you will not be able to raise them later.
- After a successful sign-in, if the stored hash uses outdated parameters (`NeedsRehash`), compute a new hash with the current parameters and save it in the same transaction. This upgrades users gradually without forcing a reset.
- Keep the verification code for the previous format until no such hashes remain.

## Verification
- Compare hashes with `CryptographicOperations.FixedTimeEquals`, never with `==`, `SequenceEqual` or string comparison.
- Run the hash computation even when the login is unknown (against a dummy hash) and return the same error for "unknown login" and "wrong password", so response time and message do not reveal which accounts exist.
- Rate-limit sign-in, registration and password reset endpoints (see web-hardening.md); lock out or slow down after repeated failures per account and per IP.

## Password Policy
- Minimum length **15 characters** when a password is the only authentication factor; **8 characters** is allowed only when it is always part of MFA. Permit a maximum length of at least 64 characters, spaces and Unicode; count Unicode code points rather than UTF-16 code units. Do not silently truncate passwords or impose composition rules or periodic forced changes. Source: [NIST SP 800-63B-4, Password Verifiers](https://pages.nist.gov/800-63-4/sp800-63b.html#passwordver).
- At password creation/change, reject common, expected and compromised passwords against a maintained blocklist. A local list is acceptable; do not send plaintext passwords to third-party services.
- Never log passwords, hashes or reset tokens; reset tokens are single-use, short-lived and stored hashed.

## Tests
- Hash → verify round trip; wrong password fails; unknown user fails.
- A hash in the previous format still verifies and is flagged for rehash; an integration test shows that sign-in replaces it with the current format.
- Malformed stored values fail verification instead of throwing.
- Password-only sign-in rejects 14 characters and accepts a valid 15-character password; the 8-character policy applies only with enforced MFA. Test Unicode length, the blocklist and inputs of at least 64 characters without truncation.
