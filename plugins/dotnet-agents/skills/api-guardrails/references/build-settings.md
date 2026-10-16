# Build settings

> **Placeholders:** values and options here are starting points. Merge them with the repo's existing files, and confirm team decisions with the user.

These settings make every local and CI build fail on known-vulnerable packages and on security analyzer findings. They are guardrail files.

## Directory.Build.props (merge into the existing file if there is one; never replace it)
```xml
<Project>
  <PropertyGroup>
    <!-- Fail restore/build on known vulnerabilities in direct and transitive packages -->
    <NuGetAudit>true</NuGetAudit>
    <NuGetAuditMode>all</NuGetAuditMode>
    <NuGetAuditLevel>low</NuGetAuditLevel>
    <WarningsAsErrors>$(WarningsAsErrors);NU1901;NU1902;NU1903;NU1904</WarningsAsErrors>

    <!-- Code analysis -->
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```
- NU1901-NU1904 are the audit warnings for low, moderate, high and critical advisories.
- `latest-recommended` can surface many warnings in existing code. Whether all warnings become errors (`TreatWarningsAsErrors`) is a team decision; the security rules below are errors either way.
- If restore uses only a private feed, the audit may have no vulnerability data. Adding nuget.org as an audit source in `nuget.config` fixes that; confirm with whoever owns the feeds.

## .editorconfig: security analyzers as errors
Comments go on their own lines; editorconfig does not support comments at the end of a line.
```ini
[*.cs]
# ADO.NET commands built from strings
dotnet_diagnostic.CA2100.severity = error
# BinaryFormatter
dotnet_diagnostic.CA2300.severity = error
# Json.NET TypeNameHandling other than None
dotnet_diagnostic.CA2326.severity = error
# Insecure DTD processing
dotnet_diagnostic.CA3075.severity = error
# Weak cryptographic algorithms
dotnet_diagnostic.CA5350.severity = error
# Broken cryptographic algorithms
dotnet_diagnostic.CA5351.severity = error
# Disabled certificate validation
dotnet_diagnostic.CA5359.severity = error
# Deprecated security protocols
dotnet_diagnostic.CA5364.severity = error
dotnet_diagnostic.CA5397.severity = error
```

## Supply-chain options (team decisions; ask)
- **Lock files**: `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>`, with CI restoring in locked mode (`dotnet restore --locked-mode`).
- **Package source mapping** in `nuget.config`, so each package can only come from an approved feed.
- **Central package management** (`Directory.Packages.props`), so versions are pinned in one place.

## After changing these files
Run `dotnet restore` and `dotnet build`. Packages that are already vulnerable now fail the build. List them in the baseline, and upgrade them with team approval; never lower the audit level to get green.
