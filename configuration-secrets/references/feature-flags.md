# Feature Flags Reference

## Library
`Microsoft.FeatureManagement.AspNetCore` (MIT). Commercial flag SaaS SDKs only after approval; self-hosted Unleash/Flagsmith servers count as standalone services (see csharp-code-style, libraries.md).

## Configuration
Flags live in the `FeatureManagement` section and follow the normal environment hierarchy:

```json
{
  "FeatureManagement": {
    "NewInvoiceLayout": false,
    "PaymentRetryV2": {
      "EnabledFor": [
        { "Name": "Percentage", "Parameters": { "Value": 10 } }
      ]
    }
  }
}
```

```csharp
builder.Services.AddFeatureManagement();
```

Flag names are constants in one static class (`FeatureFlags.NEW_INVOICE_LAYOUT = "NewInvoiceLayout"`), never string literals at call sites.

## Usage
- Check flags in Stories (business behavior) or at the controller edge (`[FeatureGate(FeatureFlags.X)]` hides an endpoint with 404). Never in DataAccess or Domain.
- Inject `IVariantFeatureManager` / `IFeatureManager`; for per-request consistency use `IFeatureManagerSnapshot`.
- Background jobs and consumers read the flag per execution/message, so a change applies without restart.
- Both branches of a flag are covered by tests.

## Lifecycle
- Every flag has an owner and a removal task created together with the flag. Release flags are removed within 1–2 releases after full rollout; dead flags are technical debt.
- A flag never guards a database schema change: schema follows expand/contract (database-migrations), the flag only switches code paths.
- Flags are not a permission system; access rules belong to authorization policies (security).
- Kill switches (ops flags for disabling an expensive or failing integration) are documented in the runbook of that integration.
