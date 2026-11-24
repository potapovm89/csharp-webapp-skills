# Allowed and Restricted Libraries

## Permitted Open-Source Licenses
Libraries with explicit licenses:
- MIT
- Apache-2.0
- PostgreSQL License (e.g. Npgsql)
- BSD (2-/3-clause)
- MPL-2.0
- LGPL (only when consumed as an unmodified NuGet package via dynamic linking)

## Strictly Prohibited
- Any library requiring a commercial or paid license (including "free for small companies" dual licenses).
- Copyleft licenses that extend obligations to our code: GPL, AGPL.
- Source-available / non-OSI licenses with service restrictions: SSPL, BUSL (Business Source License), Elastic License, Commons Clause.
- Libraries with no license file at all.
- **AutoMapper — in any version.** Versions 15+ are commercial; 14.x and earlier have a known high-severity vulnerability (GHSA-rvv3-g6hj-g44x) whose fix exists only in commercial versions. Use manual mapping extension methods instead (see `SKILL.md`, "Object Mapping").

If a library's license is unclear, stop and ask before adding it.

## Libraries vs. Standalone Services
License rules depend on **how** a component is used, not only on its license.

- **Library** — anything that ends up inside our process or our build output: NuGet packages, project references, copied or vendored source code, JS/CSS bundled into the frontend. The lists above apply strictly: GPL, AGPL, SSPL, BUSL and similar are **prohibited**, because they extend obligations to our own code.
- **Standalone service** — a separate process or container that our code talks to only over a network protocol or CLI (S3 API, SQL, HTTP, AMQP). Copyleft and source-available services are **allowed after approval** if all of the following hold:
  - the service is used **unmodified** (official image or binary; configuration only, no patched source and no custom builds with code changes);
  - our code reaches it only through its protocol, never by linking its code into our process;
  - for SSPL / BUSL / Elastic / RSAL: we use it as an internal backend and do **not** offer the product itself as a service to third parties;
  - it is recorded in `docs/arch/` as an external dependency.
- **Client SDKs are libraries.** Check the client's license separately from the server's (e.g. `MongoDB.Driver` — Apache-2.0, `StackExchange.Redis` — MIT, `AWSSDK.S3` — Apache-2.0).
- **Local-only containers** (Aspire resources, Testcontainers) that never ship to deployed environments are allowed under the same "unmodified, protocol-only" conditions without separate approval.
- Before adding a **new** copyleft or source-available service to deployed infrastructure, stop and ask the user. Prefer a permissive alternative when one is equivalent.

| Service | Server license | Usage |
| ------- | -------------- | ----- |
| RustFS | Apache-2.0 | Default local / test S3 (Aspire, Testcontainers); access via `AWSSDK.S3` |
| MinIO | AGPL-3.0 | Not used for new setups: images are no longer published openly |
| Grafana | AGPL-3.0 | Allowed as an unmodified monitoring service |
| MongoDB Server | SSPL | Allowed as an internal database; never offer MongoDB itself as a service |
| Redis ≥ 7.4 | RSALv2 / SSPL / AGPL (8.0+) | Allowed as an internal cache; prefer KeyDB or Valkey (BSD-3-Clause) for new setups |
| Garage | AGPL-3.0 | Allowed as unmodified S3-compatible storage; access via `AWSSDK.S3` |

## Version-Restricted Libraries (Licensing Restrictions)
These libraries changed to commercial/restricted licenses in later versions. You MUST NOT exceed these maximum versions:

| Library | Max Permitted Version | Reason |
| ------- | --------------------- | ------ |
| **FluentAssertions** | **7.2.0** | Version 8.0+ requires commercial license (Xceed). |
| **MediatR** | **12.5.0** | Later versions switched license / author restrictions. |
| **PDFsharp / MigraDoc** (empira) | **6.2.x** | 6.2.x and earlier are MIT; 6.5.0 is published under AGPL-3.0-only (6.3/6.4 declare no license). Also check transitive references: wrappers such as `HtmlRenderer.PdfSharp` pull PDFsharp in. |

## Central Package Management
All versions live in `Directory.Packages.props`; `.csproj` files contain `<PackageReference Include="..." />` without versions.

```xml
<PropertyGroup>
  <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
</PropertyGroup>
```

- **Transitive pinning is mandatory.** Without it a package can pull an older or newer version of a dependency than the one listed centrally, which causes version conflicts (MSB3277, e.g. two versions of `Microsoft.EntityFrameworkCore.Relational`) and lets a transitive dependency silently exceed a license cap.
- To cap a dependency that is only transitive (e.g. PDFsharp under a wrapper), add a `PackageVersion` entry for it; with pinning on, the capped version is used, and a package that requires a newer one fails restore (NU1109) instead of upgrading silently.
- Keep EF Core packages on a compatible release line with the Npgsql provider. For Aspire and OpenTelemetry core/contrib/integration packages, pin a tested compatible combination; they do not all publish matching version numbers. Check each integration's dependency ranges and release notes rather than forcing a single number across the family.
- Enforce caps against the restored NuGet graph, including transitive dependencies (see `testing`, architecture-and-contract-tests.md), not assembly versions.
- For each cap, record its official license/advisory source, review date and responsible maintainer in the project's dependency policy/ADR. A permitted old version is not automatically supported or secure: if a required fix is unavailable under the cap, plan a replacement or obtain an explicit exception with a reason, owner and expiry. Do not silently raise the cap or suppress the advisory.

## PDF Generation
- HTML → PDF: **PeachPDF** (BSD-3-Clause, pure .NET, no System.Drawing or ImageSharp). It uses system fonts, so container images must install fonts (e.g. `fonts-dejavu-core`, `fonts-liberation`).
- Prohibited for licensing reasons: QuestPDF (paid above a revenue threshold), iText (AGPL/commercial), IronPDF, Aspose, Syncfusion (commercial), SixLabors.ImageSharp 3.x+ (Six Labors Split License).
- Avoid unmaintained HTML→PDF wrappers: `HtmlRendererCore.PdfSharp`, `PdfSharpCore` (vulnerable ImageSharp 1.x / System.Drawing.Common 5.x), `DinkToPdf` / wkhtmltopdf (archived).
- `System.Drawing.Common` works only on Windows since .NET 6: do not use libraries that depend on it in services that run in Linux containers.
