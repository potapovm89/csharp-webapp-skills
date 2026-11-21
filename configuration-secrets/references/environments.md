# Environments Reference

## Standard Environments (`ASPNETCORE_ENVIRONMENT`)
1. **Local**: Local developer workstation without Docker (installed local tools).
2. **Aspire**: Local developer machine orchestrated by AspireHost.
3. **Development**: Integration test and development server environment.
4. **Stage**: QA testing environment. Can be used for debugging only.
5. **Production**: Production runtime. Direct access, manual testing, or database inspection is strictly prohibited!

## Configuration Loading Order
1. `appSettings.json`
2. `appSettings.{Environment}.json`
3. `Configs/connectionStrings.{Environment}.json` (linked from `shared/configs/`)
4. `Configs/serilog.{Environment}.json` (linked from `shared/configs/`)
5. **Environment Variables** (Highest priority — overrides all file values).
