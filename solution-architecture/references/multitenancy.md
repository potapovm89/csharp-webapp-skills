# Multitenancy Reference

Apply when one deployment serves several isolated clients (tenants: stores, companies, partners).

## Choosing the Isolation Model (record it in an ADR)
| Model | Isolation | Cost | Use when |
| ----- | --------- | ---- | -------- |
| Shared tables + `tenant_id` column | Logical | Lowest | Many small tenants, same schema (default) |
| Schema per tenant | Medium | Medium | Few tenants, per-tenant backup/restore needed |
| Database per tenant | Strong | Highest | Regulatory isolation, very large tenants |

## Tenant Resolution
- The tenant id comes **only from the authenticated principal** (a claim such as `store_id` / `tenant_id`), never from a header, query or body the client controls.
- Resolve it once at the edge into a scoped `ITenantContext` (`TenantId`); Stories and domain services read it from there.
- Background jobs and consumers set the tenant context explicitly per item/message (the tenant id travels in the message), and clear it after.

## Shared Tables Rules
- Every tenant-owned table has `tenant_id NOT NULL`, indexed, and included in unique indexes (`(tenant_id, order_id)`).
- EF Core: a global query filter `HasQueryFilter(x => x.TenantId == _tenant.TenantId)` on every tenant-owned entity, plus setting `TenantId` in `SaveChanges` for new entities. `IgnoreQueryFilters()` is allowed only in admin/cross-tenant code and is reviewed.
- Dapper: every query includes `WHERE tenant_id = @TenantId`; cover it with tests.
- Optional defense in depth: PostgreSQL Row-Level Security with `SET app.tenant_id` per connection.
- Cache keys, S3 object keys, broker routing keys and log scopes include the tenant id.

## Testing
- Every endpoint/use-case has a test that a tenant cannot read or modify another tenant's data (returns 404, not 403, to avoid revealing existence).
- Architecture or integration tests check that every entity with `TenantId` has a query filter.

## Operations
- Metrics and logs carry the tenant id (low-cardinality metrics only if the tenant count is small; otherwise only logs/traces).
- Rate limits and quotas are per tenant so one tenant cannot exhaust shared capacity.
