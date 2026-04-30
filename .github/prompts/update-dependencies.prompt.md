---
name: maintenance
description: Upgrade dependencies to their latest safe versions across all projects in the repository.
agent: agent
model: GPT-5.4
tools: [execute, read, edit, search, agent, todo]
---

Your task is to update NuGet package dependencies across all projects in this repository to their latest available versions, without introducing any breaking changes.

## Context

This is a .NET library (WorkflowCore) targeting `netstandard2.0`, `netstandard2.1`, `net6.0`, `net8.0`, and `net9.0`. It has many NuGet dependencies spread across multiple `.csproj` files under `src/` and `test/`. Dependencies must be updated safely: only patch and minor version upgrades are acceptable unless you have verified that a major version upgrade is non-breaking for this codebase.

## Steps

1. **Discover all dependencies**
   - Find every `<PackageReference>` element across all `.csproj` files in `src/` and `test/`.
   - Build a list of unique packages and their current pinned versions.

2. **Identify safe updates**
   - For each package, find the latest stable version available on NuGet.
   - Classify the upgrade:
     - **Patch** (e.g. `13.0.1` → `13.0.3`): always safe to apply.
     - **Minor** (e.g. `2.2.0` → `2.6.0`): apply unless the package's changelog documents a breaking change in the range.
     - **Major** (e.g. `5.x` → `6.x`): only apply after confirming that the public API surface consumed by this repository has not changed. If uncertain, skip the major upgrade and leave a `TODO` comment in the relevant `.csproj` file.
   - Do **not** update a package if the NuGet advisory database lists a known vulnerability fix that requires a major version bump you cannot verify as non-breaking — flag it in a comment instead.

3. **Apply updates**
   - Edit each `.csproj` file, replacing the `Version` attribute of each `<PackageReference>` with the chosen safe version.
   - Preserve any conditional `<ItemGroup>` blocks (e.g. `Condition=" '$(TargetFramework)' == 'net8.0' "`) — update the version inside them without changing the condition logic.
   - Keep floating version ranges (e.g. `Version="9.*"`) intact; only replace pinned versions.

4. **Build and verify**
   - Run `dotnet restore` to download the updated packages.
   - Run `dotnet build --no-restore` and confirm there are zero errors or warnings introduced by the updates.
   - If any compilation error or warning is caused by an updated package (e.g. an obsolete API, a changed method signature), either:
     - Revert that specific package to its previous version and document why in a comment, or
     - Fix the call-site if the fix is trivial (a rename, a parameter reorder) and does not alter the behavior of the library.

5. **Run the test suite**
   - Run `dotnet test test/WorkflowCore.UnitTests --no-build` and confirm all tests pass.
   - Run `dotnet test test/WorkflowCore.IntegrationTests --no-build` and confirm all tests pass.
   - If a test fails due to a dependency update, treat the failure the same way as a compilation error above (revert the package or apply a trivial fix).

6. **Summarize changes**
   - After all updates are applied and verified, produce a summary table listing:
     - Package name
     - Old version
     - New version
     - Upgrade type (Patch / Minor / Major)
     - Any packages skipped and the reason (breaking change risk, advisory, etc.)
