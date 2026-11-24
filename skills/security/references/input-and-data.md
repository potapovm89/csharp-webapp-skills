# Input Handling & Data Protection Reference

## Injection
- **SQL**: parameters only (Dapper `CommandDefinition` parameters, EF LINQ or `FromSql` with interpolation). `FromSqlRaw` / string-concatenated SQL with user input is prohibited.
- **HTML / PDF templates**: every user-supplied value inserted into HTML is encoded with `WebUtility.HtmlEncode` (or a template engine with auto-encoding). Raw insertion lets a user inject markup and links into generated documents.
- **Logs**: structured logging only (`{OrderId}`), never interpolation — it prevents log forging and keeps values out of the message template.
- **Process/shell**: do not start processes with user input in arguments.

## Files, Paths and S3 Keys
- Object keys are built from validated identifiers (`$"{storeId}_{orderId}.pdf"`), never from a user-provided file name or path.
- If a user file name must be kept, store it as metadata and sanitize it for `Content-Disposition`.
- Uploads: size limit per endpoint, allowed content types checked by content (magic bytes), not only by extension; files never land in a web-served directory.
- Download endpoints check ownership before streaming the object.

## SSRF
- Never call a URL that comes from user input. External integrations use base addresses from options (`api-clients`).
- If user-provided URLs are a business requirement (webhooks), validate against an allowlist of schemes and hosts, resolve and reject private/link-local/loopback addresses, and disable redirects.
- HTML-to-PDF rendering does not load remote resources for templates that contain user data; embed images as local files or `data:` URIs.

## Deserialization
- System.Text.Json by default. If Newtonsoft.Json is used, `TypeNameHandling` stays `None`.
- Never deserialize to `object`/polymorphic types from untrusted input without an explicit allowlist of derived types.

## Personal Data (152-FZ / GDPR)
- Classify fields: personal data (name, phone, email, address, IP, device ids), sensitive (passport, payment data — avoid storing; use the payment provider's tokens).
- Minimize: collect and keep only what the feature needs; document purpose and retention in `docs/` next to the table/contract.
- Protect: TLS in transit; encrypt sensitive columns at rest where required; restrict DB access by role.
- Never log personal data or send it to external services that are not approved processors (including AI tools).
- Support deletion/anonymization: soft delete is not erasure — provide an anonymization path for erasure requests.
- Storage location: personal data of Russian citizens is collected and stored in databases located in Russia.

## Dependencies and Secrets in CI
In `Directory.Build.props`:

```xml
<PropertyGroup>
  <NuGetAudit>true</NuGetAudit>
  <NuGetAuditMode>all</NuGetAuditMode>      <!-- include transitive packages -->
  <NuGetAuditLevel>moderate</NuGetAuditLevel>
  <WarningsAsErrors>$(WarningsAsErrors);NU1903;NU1904</WarningsAsErrors>
</PropertyGroup>
```

- A high/critical advisory that cannot be fixed (no patched version under an allowed license) needs an explicitly approved, finding-scoped exception with affected package/version or digest, reason, compensating control, owner, approver, expiry and remediation issue. Only then use `<NuGetAuditSuppress Include="<advisory url>" />` linked to that record; an expired exception blocks. No blanket audit disablement. `delivery-and-supply-chain` defines the executable gate and evidence checks.
- CI runs `dotnet list package --vulnerable --include-transitive` and a secret scanner (e.g. gitleaks) on every MR.
- Evaluate the scanner's documented findings, freshness and failures, not its exit code alone. Apply [pipeline-gates.md](../../delivery-and-supply-chain/references/pipeline-gates.md) to affected MR/candidate jobs; release image contents, SAST/IaC and optional test-environment DAST have explicit applicability and policy.
