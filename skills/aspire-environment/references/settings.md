# Aspire Host Configuration Reference

## Configuration Naming Alignment
The core invariant across all environments:
`Resource Name in AspireHost` == `ConnectionStrings:<Name> Key` == `GetConnectionString("<Name>") in App`.

Example:
- AspireHost: `postgres.CreateDatabase("postgresdbForAPI", out var apiDb)`
- Local / Dev / Stage: `shared/configs/connectionStrings.{env}.json` -> `"postgresdbForAPI": "..."`
- WebHost `Program.cs`: `builder.Configuration.GetConnectionString("postgresdbForAPI")`

## Configuration Files
- AspireHost settings: `AspireHost/appsettings.Aspire.json`.
- Kebab-case topic and queue validation is enforced by `[KebabCaseCollectionItemsValidation]` on options classes.
