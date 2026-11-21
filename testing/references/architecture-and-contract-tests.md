# Architecture & Contract Tests Reference

## Architecture Tests (`tests/Itl.<Project>.ArchitectureTests`)
Layer rules from `solution-architecture` are enforced by tests, not by review alone. Reflection over referenced assemblies checks assembly-level dependencies, as below. Type/method-level restrictions require an IL/type dependency check (e.g. NetArchTest.Rules, MIT) or a Roslyn analyzer; assembly references alone cannot distinguish DI registration from business use.

```csharp
[Theory]
[MemberData(nameof(ForbiddenDependencies))]
public void Assembly_DoesNotReference_ForbiddenLayer(Assembly assembly, string forbiddenPrefix)
{
    // Act
    var references = assembly.GetReferencedAssemblies()
        .Select(reference => reference.Name!)
        .Where(name => name.StartsWith(forbiddenPrefix, StringComparison.Ordinal))
        .ToList();

    // Assert
    references.Should().BeEmpty($"{assembly.GetName().Name} must not depend on {forbiddenPrefix}");
}
```

Minimum rule set:
- Domain references no other solution project (and no protobuf/broker/web packages).
- Contracts reference nothing from the solution.
- DataAccess does not reference Stories or Hosts; Stories do not reference Hosts.
- Hosts do not reference each other.
- Hosts use `DataAccess` only in composition/DI code: maintain an explicit allowlist of registration types/methods and inspect type/method dependencies outside it. Include a negative fixture with DataAccess use in a controller to prove the check catches it.
- The chosen Stories mechanism is the only one in use; mapping types follow `Common/Mappers`, except EF-translatable DataAccess projections.
- No prohibited package or version is resolved, including transitive dependencies. Read NuGet package IDs and versions from each restored `obj/project.assets.json` (all target frameworks/runtime targets), a restore-generated package report or committed lock files verified by locked restore. Use a NuGet/SemVer-aware comparison and check package license metadata. `AssemblyName.Version` is not the NuGet package version and must not enforce package caps. The policy is in `csharp-code-style`, libraries.md.
- No direct system clock reads (`DateTime.Now/UtcNow/Today`, `DateTimeOffset.Now/UtcNow`) in Stories, DataAccess, Common and hosts — detected in IL, with a negative fixture (`csharp-code-style`, time-money-culture.md).
- Every project has a committed `packages.lock.json` (a missing lock file fails the test).
- Exactly one `.slnx` in the root and no `.sln`; every `.csproj` is SDK-style (`Sdk` attribute or element), there is no `packages.config`, and every `.csproj` in the repository is included in the `.slnx`. Verify with a fixture: a stray `.sln` must fail the test.
- Optional naming rules: handlers end with `Handler`, validators with `Validator`, options with `Options`.

### Implementation notes
- Type/method-level checks need IL, not reflection over signatures: walk `MethodBody.GetILAsByteArray()`, resolve tokens with `Module.ResolveMember` and include nested compiler-generated types (async state machines, lambda closures). A constructor-parameter check misses `GetRequiredService<T>()`, `typeof(T)` and lambdas.
- Each rule has fixtures that **must** fail (a controller resolving DataAccess by `typeof`, in an async method and in a lambda; a lambda reading `DateTime.UtcNow`; lock files with a capped package as direct and as transitive dependency and as pre-release). A rule without a failing fixture proves nothing.
- Copying architecture tests from another repository (including a template) is not compliance: check the copy against this minimum set item by item and against the current skill text.

The compiler drops unused assembly references: reflection checks actual compiled usage. Validate `.csproj`/resolved project references separately when a rule forbids the reference itself. Package checks inspect the restored graph even when no package type is used. Verify them with a disallowed direct package, a disallowed transitive package and a package whose assembly version differs from its NuGet version.

Source: [Microsoft: NuGet package version versus assembly version](https://learn.microsoft.com/en-us/dotnet/standard/library-guidance/versioning).

## Contract Tests
Contracts break consumers silently; a test must fail when a contract changes unintentionally.

- **REST**: fetch each configured OpenAPI document in an integration test — `/openapi/user-api.json` and `/openapi/admin-api.json` with the built-in stack (`web-host-api`, swagger.md); `/swagger/{name}/swagger.json` only for a documented Swashbuckle legacy host. Compare with `docs/contracts/api/openapi.user-api.v1.json` and `openapi.admin-api.v1.json`; use the exact configured document names if versioned documents differ. A snapshot detects a difference, not whether it is compatible: run a semantic OpenAPI compatibility check against the main-branch baseline or explicitly review changes against `web-host-api`, api-evolution.md. Updating a snapshot alone does not approve a breaking change. Test that a removed endpoint or newly required request field is rejected by the compatibility gate.
- **gRPC**: run `buf breaking` (Apache-2.0, standalone tool) against the main branch in CI, or keep the compiled descriptor snapshot and compare field numbers and types.
- **Broker messages**: serialize a sample of every published message type and compare it with a committed JSON snapshot; deserialize the previous version's sample with the current consumer to prove backward compatibility.
- Snapshots are compared as normalized JSON, not as raw text.

## Where They Run
- Architecture tests are fast and run with unit tests on every build.
- Contract snapshot tests run with integration tests; `buf breaking` runs in the CI pipeline.
