---
name: pdf-reports
description: >-
  Use this skill when generating PDF documents or reports server-side (HTML→PDF via PeachPDF from Razor component templates, programmatic PDF via PDFsharp/MigraDoc ≤ 6.2.x), building localized templates, generating in Stories (directly or dispatched by a WorkerHost job), storing reports in S3 and issuing presigned links, or defining report retention. Triggers (RU): PDF, отчёт, документ, PeachPDF, PDFsharp, печать, выгрузка.
---

# PDF Reports

Server-side document generation with a full delivery path: a Story builds the document model, a Razor component template renders HTML, PeachPDF converts it to PDF, the file goes to S3, the report row in the database becomes `Ready`, and the client receives a presigned link. Generation is business logic and lives in `Stories`. A WorkerHost job only claims due reports and dispatches the generating Story; it never renders or uploads itself (`solution-architecture`, `cqrs-stories`, `worker-host-jobs`).

## Library Policy (license caps are non-negotiable, `csharp-code-style`)
- **HTML → PDF: PeachPDF** (BSD-3) — the default for any document with layout, CSS or localized text.
- **HTML templates: Razor components rendered with `HtmlRenderer`** (`Microsoft.AspNetCore.Components.Web`, part of ASP.NET Core, MIT). Razor encodes every `@value` by default. String interpolation/concatenation of HTML, `Replace`-based templating and third-party template engines are prohibited without an ADR.
- **Programmatic PDF: PDFsharp / MigraDoc ≤ 6.2.x** (6.5.0+ is AGPL) — for machine-drawn documents (barcodes, stamps, exact geometry) where HTML adds nothing.
- QuestPDF, iText and other commercial/AGPL generators are prohibited. Adding a new PDF library requires the standard license check and an ADR.

## Pipeline

```
Controller → RequestReport Story: ownership check, report row (Pending), outbox "report-requested"   — one transaction
WorkerHost job (or the broker consumer of "report-requested") → GenerateReport Story:
  1. Claim: Pending → Generating (xmin conflict = someone else claimed it)
  2. Build the document model from domain data (mapped via Common/Mappers, never a raw entity)
  3. IDocumentRenderer: Razor template → HTML (user data encoded) → PeachPDF → PDF bytes
  4. Upload to S3 via ISingleBucketFileClient under the deterministic key (s3-storage)
  5. Transaction: Generating → Ready + storage_key (+ outbox "report-ready" for a realtime push)
Client polls the status or gets the push; a Story issues the link only for an owned, unexpired Ready report
```

- Upload the object before committing `Ready` and `storage_key`. If the commit in step 5 fails, the generation retry uses the same deterministic key.
- A report that takes longer than a couple of seconds is asynchronous: the API returns `202 Accepted` with the report id and a status URL (`web-host-api`), the client polls or gets a realtime push (`realtime-signalr`). A small interactive document may be generated synchronously inside the `RequestReport` Story — through the same steps 2–5.
- Never stream a generated PDF straight into the HTTP response of a controller: it bypasses storage, retention and audit — an exception needs the user's explicit approval and an ADR.

## Retention and Cleanup

Read [the delivery workflow](./references/delivery.md#retention-and-cleanup) before changing cleanup or link issuance. A cleanup Story commits `CleanupPending` with the object identity before deleting from S3. A report in this state is unavailable for link issuance and remains discoverable by the cleanup job. After successful deletion or confirmed absence, finalize the row as deleted. A retry resumes from `CleanupPending` and uses the same object identity.

## Non-Negotiable Rules
1. **User data is encoded inside templates** (`security`, input-and-data.md): only Razor `@value` output; `MarkupString` is allowed only for HTML sanitized upstream. URLs, paths and S3 keys are never built from raw user input.
2. **Templates are localized** like UI text: strings via `IStringLocalizer`, dates/numbers/currency formatted with the document's culture, `CultureInfo.InvariantCulture` only for machine formats (`csharp-code-style`, time-money-culture.md).
3. **Storage is S3-only** (`s3-storage`): deterministic keys, presigned access, no public buckets. The DB row ([delivery.md](./references/delivery.md)) is the source of truth about what was issued to whom.
4. **Retention and personal data**: every report type has a documented retention (152-FZ/GDPR, `security`); the cleanup job is part of the delivery flow, not an afterthought.
5. **Reproducibility**: a regenerated report of the same business data is byte-different but content-identical — tests assert content, not bytes.

## Subtopic References
- [generation.md](./references/generation.md) — template project and renderer, fonts, page layout, determinism, performance, tests. **Read it before writing the first template.**
- [delivery.md](./references/delivery.md) — report entity, statuses, S3 keys, presigned issuance, retention and cleanup.

## Completion Evidence

- On an isolated database and private bucket, verify an authorized `Ready` report can be downloaded before expiry. Advance the test clock past expiry. Verify that link issuance is denied and that the row has committed `CleanupPending` before the first S3 delete.
- Inject a failure after S3 confirms deletion and before the DB finalization commits. Assert the object is absent, the row remains `CleanupPending`, and link issuance is still denied. Run cleanup again for the same report and object identity. Assert the retry accepts successful deletion or confirmed absence, the row is finalized as deleted, and another run makes no changes.
- Simulate a storage timeout or access denial. Verify that cleanup remains pending rather than reporting deletion. Record test commands and the observed row, object and link-issuance outcomes.
