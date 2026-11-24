# Containers Reference

## Dockerfile Template (per host)

```dockerfile
ARG SDK_IMAGE=mcr.microsoft.com/dotnet/sdk:10.0
ARG RUNTIME_IMAGE=mcr.microsoft.com/dotnet/aspnet:10.0
FROM ${SDK_IMAGE} AS build
WORKDIR /src
# Restore first for layer caching
COPY global.json Directory.Build.props Directory.Packages.props nuget.config ./
COPY src/ ./src/
RUN dotnet restore src/Itl.<Project>.WebHost/Itl.<Project>.WebHost.csproj --locked-mode
RUN dotnet publish src/Itl.<Project>.WebHost/Itl.<Project>.WebHost.csproj \
    -c Release -o /app/publish --no-restore /p:UseAppHost=false

FROM ${RUNTIME_IMAGE} AS final
WORKDIR /app
COPY --from=build /app/publish .
USER $APP_UID
EXPOSE 8080 8081
ENTRYPOINT ["dotnet", "Itl.<Project>.WebHost.dll"]
```

The default image tags above are illustrative local inputs. A release build supplies reviewed `repository:10.0@sha256:...` values for both image arguments, records them in provenance and copies the solution's actual build/configuration inputs. `src/` includes the committed lock files. If shared imported props/targets live outside this layout, copy them before restore; do not invent a digest or omit an input just to keep the example unchanged.

## Frontend (SPA) Image
A React/Vite SPA is a runnable application too. Either the backend host serves the built assets (one image), or the SPA gets its own image:

```dockerfile
ARG NODE_IMAGE=node:22-alpine
ARG WEB_IMAGE=nginxinc/nginx-unprivileged:1.27-alpine

FROM ${NODE_IMAGE} AS build
WORKDIR /app
COPY src/<App>.Frontend/package.json src/<App>.Frontend/package-lock.json ./
RUN npm ci
COPY src/<App>.Frontend/ ./
RUN npm run build

FROM ${WEB_IMAGE} AS final
COPY --from=build /app/dist /usr/share/nginx/html
COPY src/<App>.Frontend/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 8080
```

- The example uses npm and `package-lock.json`. Use the frozen install matching the app or workspace lockfile. For pnpm, copy the workspace build inputs, use the pinned package manager and run `pnpm install --frozen-lockfile`. Pin release base images to reviewed digests.
- Runtime configuration (API URL, OIDC authority) is served as `/config.json` mounted at deploy time, so one image works in every environment (`react-frontend`).
- nginx config: SPA fallback to `index.html`, long cache for hashed assets, `no-cache` for `index.html`, security headers/CSP (`react-frontend`, security.md). The unprivileged image listens on 8080 as non-root.

## Rules
- Run as non-root (`USER $APP_UID`, provided by the official images); the container listens on ports > 1024 (8080 by default in .NET 8+).
- No secrets, connection strings or environment-specific `appsettings.*.json` with real values in the image.
- Release builds pin base images by digest on the approved .NET release line; update them through reviewed patch changes and rebuild/scan a new candidate. The floating `10.0` tag alone does not fix the input bytes. Artifact checks/SBOM follow `delivery-and-supply-chain`.
- Install only what the host needs. PDF rendering needs fonts (e.g. `fonts-dejavu-core`); culture-specific formatting needs ICU (`DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false` + `libicu`). Document why a package is installed.
- `.dockerignore` excludes `bin/`, `obj/`, `tests/`, `.git/`, local secrets.
- One process per container; logs go to stdout in JSON (Serilog console sink with a JSON formatter in non-local environments).
- A release tag (commit SHA or semver) is descriptive; deployment selects the verified image digest and retains the tag-to-digest mapping. Never deploy `latest` or assume a tag cannot be overwritten. Promote the same candidate rather than rebuilding for each environment.
