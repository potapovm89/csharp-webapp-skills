# API Evolution & Compatibility (REST)

Clients (frontends, other services, partner stores) update on their own schedule. Every change to a public endpoint is either **backward compatible** or shipped as a **new version** with a deprecation period.

## Compatible Changes (no new version)
- Add a new endpoint.
- Add an optional request field (missing value keeps the old behavior).
- Add a response field. Clients must ignore unknown fields; System.Text.Json does by default.
- Add a new enum value **only** if clients are documented to tolerate unknown values; otherwise it is breaking.
- Relax validation (accept more inputs).

## Breaking Changes (new version required)
- Remove or rename an endpoint, field, query parameter or header.
- Change a field type, format, unit, meaning or nullability; make an optional field required.
- Tighten validation; change status codes or error format for existing cases.
- Change authentication or authorization requirements for an existing endpoint.

## Versioning
- URL segment: `api/v1/...`, `api/v2/...`. The version is part of the route attribute on the controller; one controller per major version when the shape differs.
- v1 and v2 run side by side and share Stories; only contracts (DTOs) and mapping differ. DTOs of different versions live in separate folders: `Contracts/<Area>/V2/`.
- Deprecated endpoints return the `Deprecation` and `Sunset` headers (RFC 8594 / RFC 9745) and are marked `[Obsolete]` so Swagger shows them as deprecated.
- Remove an old version only after its sunset date and after metrics show no traffic.
- Every version change is documented in `docs/contracts/api/<controller>.md` with a changelog entry.

## Idempotent Writes
- `PUT` and `DELETE` are idempotent by definition; implement them that way.
- `POST` that creates a resource identified by a business key (store + order) is naturally idempotent: a retry finds the existing row through the unique constraint.
- Otherwise accept an `Idempotency-Key` header (see `postgresql-data-access`, concurrency.md): store key + request hash + response, return the stored response on retry, reject the same key with a different body with 422.

## Error Contract
- [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457.html) everywhere (supersedes RFC 7807); `type` is a stable URI per error kind, `title` a stable short text, `detail` human-readable. Updating this reference does not change the existing wire contract.
- Clients branch on `status` and `type`, never on `detail` text — so changing a message is not breaking, changing a `type` is.

## Pagination Contract
- Cursor (keyset) pagination for large or growing collections: `?limit=50&after=<opaque cursor>`, response `{ items, nextCursor }`.
- Offset pagination (`page`, `pageSize`) only for small bounded lists; cap `pageSize` in the validator.
