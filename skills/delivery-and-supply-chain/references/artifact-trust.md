# Artifact Identity, Evidence & Promotion

## Release Manifest
Store a machine-readable manifest as access-controlled CI/release evidence, not as a secret-filled file in git. Use the project's existing format; at minimum bind:

| Field | Meaning |
|---|---|
| Release and source | Release id, full source commit, repository, reviewed ref/MR |
| Build | Pipeline/run and trusted builder identity; effective toolchain; target OS/architecture |
| Artifacts | Registry/repository plus digest for every host, frontend asset bundle and migration runner |
| Dependencies | Lock-file hashes and SBOM references/hashes for actual shipped contents |
| Gates | Check/policy versions, outcomes, timestamps and links to reports/approved exceptions |
| Trust | Provenance/attestation, signature reference and expected verifier policy |
| Operations | Compatible schema/config revision, prior verified release, rollout/rollback criteria |

Do not insert credentials, access tokens, private endpoints that must remain confidential, personal data or raw production payloads. Evidence retention must outlast artifact support/rollback needs and the agreed incident-investigation window; protect against replacement/deletion and record access.

## SBOM and Provenance
- Produce a machine-readable SBOM (SPDX or CycloneDX with a supported format version) covering the shipped .NET/npm components and final image OS/runtime components, including transitive packages and license identifiers. Build-only test tools are separate from runtime inventory. Check that packages/components can be associated with the exact artifact; a solution-wide package list alone is not the final image's inventory.
- Preserve document relationships when inventories are split by application/image. Verify an added runtime dependency appears, a development-only dependency is classified correctly and the SBOM hash/reference resolves for the promoted digest.
- Provenance records who built the artifact, from what revision/inputs and through which build process. It is distinct from SBOM (contents), scan results (known findings) and a signature (authenticated statement). None proves that the application is free of vulnerabilities.
- Choose the required assurance level in the project's delivery policy. Do not label an arbitrary CI JSON file “SLSA compliant”; a level claim needs the corresponding builder/verification guarantees. [SLSA build tracks](https://slsa.dev/spec/v1.2/tracks) describes this distinction.

## Authenticity Policy
For deployed artifacts, define an enforced authenticity mechanism: e.g. signed digest-bound artifacts/attestations, or a documented equivalent trusted registry/promotion control appropriate to the threat model. A registry tag alone is not such a mechanism. For high-risk releases prefer independently verifiable, digest-bound signatures/provenance; adding a signer/service remains an architectural/licensing decision.

If signatures are used, verify the expected key or workload identity, issuer, authorized repository/ref/workflow and artifact digest, plus validity/revocation or transparency evidence required by that signing scheme. Verification must use trust roots/policy from a protected source, not a key/policy supplied by the untrusted artifact or MR. Specify key/identity rotation and compromise response. A correctly signed image from an unauthorized builder/ref still fails policy.

## Promotion and Rollback
1. Resolve the recorded artifact digest. Validate evidence hashes/binding, current policy, non-expired exceptions and scan freshness. Missing/invalid required evidence stops promotion.
2. Promote the exact tested digest (no rebuild). If registry transfer, multi-architecture index conversion or repackaging changes identity, establish and verify the new candidate's evidence; never reuse old reports for different bytes/platform contents.
3. Inject environment configuration/secrets at runtime; record non-secret configuration revision. Run migrations once before the new hosts (`database-migrations`, `runtime-deployment`). A migration failure blocks rollout.
4. Check rollout timeout, smoke/business behavior, and the release observation window/SLIs (`reliability-and-performance`). Readiness is not the whole release check. A passing sample shorter than the SLO window is not proof that the SLO was met.
5. If rollback is appropriate and authorized, use the prior verified digest/config combination only while schema/contracts remain compatible. Re-verify that it is still permitted by security policy; an emergency exception is explicit and time-bound. Never run destructive `Down` or restore production data automatically after a rollout failure.

Exercise rejection of a modified digest, mismatched SBOM/attestation, valid signature from the wrong signer/ref, missing required evidence and an expired exception using isolated fixtures. Logs should identify the failed policy without exposing signing material. Repository changes do not grant this agent permission to execute production operations.
