# Secret Protection & Leak Prevention

## Strictly Prohibited in Git Repositories
Never commit real production, stage, or development secrets:
- Real usernames and passwords.
- Real connection strings with credentials.
- Private keys (RSA, JWT), client secrets (OAuth, Keycloak), SSL certificates.
- Cloud API keys and cloud storage secrets (AWS/S3 AccessKey/SecretKey).
- External API tokens.

## Permitted in Git
- Fake local dummy secrets for `Local` or `Aspire` test containers (e.g. `secretName`, `secretPassword`, `admin`, `password`).

## How Real Secrets are Delivered
- Secrets are passed at deployment time via **Environment Variables** (`SECTION__PROPERTY`, e.g. `FileStorageOptions__SecretKey`). This is the default channel.
- Secret files (e.g. populated `connectionStrings.Stage.json`) are injected by CI/CD deployment pipelines, never committed to git.
- CI jobs receive only their environment/job permissions; prefer scoped short-lived workload credentials where supported and prevent untrusted MR code from accessing protected secrets (`delivery-and-supply-chain`). Keep signing keys/trust policy and backup decryption/recovery keys protected and rotatable; a recovery drill uses isolated identities, not production credentials (`data-recovery`).

### Mounted Secret Files
Deployment may also mount secrets as read-only files (Kubernetes `Secret` volumes backed by tmpfs) — for certificates, key material and configuration files that do not map cleanly to environment variables:
- The mount is read-only; the file path reaches the application as an explicit configuration option (e.g. `FileStorageOptions__KeyFilePath`), never a hardcoded path.
- The file is loaded through the configuration layer (a file-based `IConfiguration` source wired in `CreateWebAppBuilder`); business code still reads validated `Options`, not files.
- Multi-line PEM/PKCS#8/JWT signing keys belong here — not in environment variables.

### Named Secret Stores
When the platform provides a secret store (HashiCorp Vault, Azure Key Vault, Secret Store CSI driver), secrets are injected into the standard configuration layer by the platform integration (CSI mount, Vault Agent sidecar, or a configuration provider):
- Business code and `Options` classes never reference a store SDK. A secret reaches the application only through `IConfiguration`/`Options`, so `Local`/`Aspire` runs keep working without the store.
- Introducing or replacing a store integration is an architecture decision recorded in `docs/adr/`, coordinated with `runtime-deployment` (mounts) and this skill.
- Secrets are never cached into source-controlled files anywhere in the flow.

## Rotation
- Every secret has an owner and a documented rotation interval (at minimum: broker credentials, external API tokens, Keycloak client secrets, TLS/signing keys); the table lives in `docs/settings/` next to the options that consume them.
- Rotation is a runbook operation for the authorized operator (`docs/runbooks/secret-rotation.md`): rotate in the target service, then update the deployment secret. This skill prepares the runbook; it does not grant production access.
- Rotation must not require an image rebuild or code change — values arrive through the configuration layer. A restart (or validated reload) is part of the runbook; `ValidateOnStart` fails fast if a rotated value is invalid.
- Where the target supports dual credentials, rotate over an overlap window instead of a hard cutover.

## Strict Rules
- Never log secrets or exception messages containing sensitive data.
- Never pass secrets in URL query strings.
- In `Options` classes, never expose secrets in `ToString()` or public serialization models.
- If a secret is ever accidentally committed: **immediately rotate the secret** in the target service. Never attempt to rewrite git history (force push/rebase); the secret is considered permanently compromised.
