# Forbidden-pattern scan (xUnit test)

> **Placeholders:** the namespace, the allowlist path and the test-folder naming are illustrations. Confirm them at setup and record them in project-facts.

A test that scans production code (`*.cs`) and configuration (`appsettings*.json`) for dangerous patterns. It runs in `dotnet test`, so every IDE and CI enforce it the same way, and it needs no extra packages. Test folders are not scanned.

## The test
```csharp
using System.Text.Json;
using System.Text.RegularExpressions;
using Xunit;

namespace Contoso.Api.Tests.Guardrails;

public sealed class ForbiddenPatternTests
{
    // Repo-relative allowlist path, confirmed at setup.
    private const string AllowlistPath = "eng/security/guardrails-allowlist.json";

    private static readonly string[] SkippedDirectories = ["bin", "obj", ".git", ".vs", ".idea", "node_modules", "TestResults"];
    private static readonly JsonSerializerOptions JsonOptions = new(JsonSerializerDefaults.Web);
    private static readonly string RepoRoot = FindRepoRoot();

    [Flags]
    private enum Scope { Code = 1, Config = 2, Both = Code | Config }

    private sealed record Rule(string Id, Scope Scope, string Pattern, string Why)
    {
        public Regex Regex { get; } = new(Pattern, RegexOptions.CultureInvariant, TimeSpan.FromSeconds(1));
    }

    private sealed record AllowlistEntry(string Rule, string File, string Contains, string Reason, string ApprovedBy, DateOnly ReviewBy);
    private sealed record AllowlistFile(AllowlistEntry[] Entries);

    private static readonly Rule[] Rules =
    [
        new("SQL-RAW", Scope.Code, @"\b(?:FromSqlRaw|ExecuteSqlRaw(?:Async)?|SqlQueryRaw)\b",
            "Raw SQL needs a reviewed exception; prefer LINQ or FromSql/ExecuteSql with interpolation."),
        new("QUERY-FILTER-BYPASS", Scope.Code, @"\bIgnoreQueryFilters\s*\(",
            "Bypasses query filters; needs a reviewed exception that re-applies owner/tenant checks."),
        new("CORS-ANY-ORIGIN", Scope.Code, @"\bAllowAnyOrigin\s*\(|\bSetIsOriginAllowed\s*\(\s*_?\w*\s*=>\s*true\b",
            "CORS must list explicit origins."),
        new("JWT-VALIDATION-OFF", Scope.Code,
            @"\b(?:ValidateIssuer|ValidateAudience|ValidateLifetime|RequireSignedTokens|RequireExpirationTime|RequireHttpsMetadata)\s*=\s*false\b",
            "Token validation must not be weakened."),
        new("JWT-CUSTOM-VALIDATOR", Scope.Code, @"\b(?:SignatureValidator|IssuerValidator|AudienceValidator|LifetimeValidator)\s*=(?!=)",
            "Custom token validators need security review."),
        new("TLS-VALIDATION-OFF", Scope.Code, @"\bDangerousAcceptAnyServerCertificateValidator\b|\bServerCertificateCustomValidationCallback\s*=(?!=)",
            "TLS certificate validation must not be disabled."),
        new("SQL-TRUST-CERT", Scope.Both, @"(?i)\bTrustServerCertificate\s*=\s*(?:true|yes)\b",
            "The SQL Server certificate must be validated outside local development."),
        new("EF-SENSITIVE-LOGGING", Scope.Code, @"\bEnableSensitiveDataLogging\s*\(",
            "Allowed only inside a Development-only branch, with a reviewed exception."),
        new("DEV-EXCEPTION-PAGE", Scope.Code, @"\bUseDeveloperExceptionPage\s*\(",
            "Allowed only in Development, with a reviewed exception."),
        new("IDENTITY-PII-LOGGING", Scope.Code, @"\b(?:ShowPII|LogCompleteSecurityArtifact)\s*=\s*true\b",
            "Allowed only in Development, with a reviewed exception."),
        new("UNSAFE-DESERIALIZATION", Scope.Code, @"\bBinaryFormatter\b|\bTypeNameHandling\s*=\s*TypeNameHandling\.(?!None\b)\w+",
            "Type-name-driven deserialization enables remote code execution."),
        new("LOG-INTERPOLATION", Scope.Code,
            @"\.Log(?:Trace|Debug|Information|Warning|Error|Critical)\s*\((?:[^;""]*,\s*)?\$""|\bLog\.(?:Verbose|Debug|Information|Warning|Error|Fatal)\s*\((?:[^;""]*,\s*)?\$""",
            "Use message templates, not interpolated strings."),
        new("DB-SCHEMA-AT-RUNTIME", Scope.Code, @"\.EnsureCreated(?:Async)?\s*\(|\.Database\.Migrate(?:Async)?\s*\(",
            "Schema changes go through reviewed migrations applied by the pipeline."),
        new("WEAK-HASH", Scope.Code, @"\b(?:MD5|SHA1)\.(?:Create|HashData)\s*\(|\bnew\s+(?:MD5|SHA1)CryptoServiceProvider\b",
            "Weak hash; use SHA-256 or better, or a reviewed exception for non-security use."),
        new("SECRET-IN-SOURCE", Scope.Both,
            @"(?i)""[^""]*\b(?:password|pwd)\s*=\s*[^;""'\s<{$]+|\b(?:AccountKey|SharedAccessKey)\s*=\s*[^;""'\s<{$]+|-----BEGIN [A-Z ]*PRIVATE KEY-----|""[A-Za-z]*(?:Secret|Password|ApiKey)""\s*:\s*""[^""<{$\s][^""]*""",
            "Secrets belong in Key Vault or App Service settings, never in the repo."),
    ];

    [Fact]
    public void Production_code_has_no_unapproved_forbidden_patterns()
    {
        var allowlist = LoadAllowlist();
        var violations = new List<string>();

        foreach (var path in Walk(RepoRoot))
        {
            var scope = ScopeOf(Path.GetFileName(path));
            if (scope is null) continue;

            var relative = Path.GetRelativePath(RepoRoot, path).Replace('\\', '/');
            var lines = File.ReadAllLines(path);

            foreach (var rule in Rules.Where(r => (r.Scope & scope.Value) != 0))
            {
                for (var i = 0; i < lines.Length; i++)
                {
                    if (!rule.Regex.IsMatch(lines[i])) continue;

                    var approved = allowlist.Any(e =>
                        e.Rule == rule.Id && e.File == relative && lines[i].Contains(e.Contains, StringComparison.Ordinal));
                    if (!approved)
                        violations.Add($"{relative}:{i + 1} [{rule.Id}] {rule.Why}");
                }
            }
        }

        Assert.True(violations.Count == 0,
            "Forbidden patterns found. Fix the code, or propose an allowlist entry as a guardrail change:\n"
            + string.Join("\n", violations));
    }

    [Fact]
    public void Allowlist_entries_are_complete_known_and_not_expired()
    {
        var today = DateOnly.FromDateTime(DateTime.UtcNow);
        var problems = LoadAllowlist()
            .Where(e => string.IsNullOrWhiteSpace(e.Reason) || string.IsNullOrWhiteSpace(e.ApprovedBy)
                     || string.IsNullOrWhiteSpace(e.Contains) || e.ReviewBy < today
                     || !Rules.Any(r => r.Id == e.Rule))
            .Select(e => $"{e.Rule} in {e.File}: missing reason/approver/text, unknown rule, or review date {e.ReviewBy:yyyy-MM-dd} has passed")
            .ToList();

        Assert.True(problems.Count == 0, string.Join("\n", problems));
    }

    [Fact]
    public void Allowlist_entries_still_match_the_code()
    {
        var stale = LoadAllowlist()
            .Where(e =>
            {
                var path = Path.Combine(RepoRoot, e.File);
                return !File.Exists(path) || !File.ReadAllText(path).Contains(e.Contains, StringComparison.Ordinal);
            })
            .Select(e => $"{e.Rule} in {e.File} no longer matches; remove the entry (guardrail change)")
            .ToList();

        Assert.True(stale.Count == 0, string.Join("\n", stale));
    }

    private static Scope? ScopeOf(string fileName)
    {
        if (fileName.EndsWith(".cs", StringComparison.OrdinalIgnoreCase)) return Scope.Code;
        if (fileName.StartsWith("appsettings", StringComparison.OrdinalIgnoreCase)
            && fileName.EndsWith(".json", StringComparison.OrdinalIgnoreCase)) return Scope.Config;
        return null;
    }

    private static IEnumerable<string> Walk(string directory)
    {
        foreach (var file in Directory.EnumerateFiles(directory))
            yield return file;

        foreach (var sub in Directory.EnumerateDirectories(directory))
        {
            var name = Path.GetFileName(sub);
            if (SkippedDirectories.Contains(name, StringComparer.OrdinalIgnoreCase) || IsTestDirectory(name))
                continue;
            foreach (var file in Walk(sub))
                yield return file;
        }
    }

    // Test code is out of scope for this scan. Confirm the repo's test folder naming at setup.
    private static bool IsTestDirectory(string name) =>
        name.EndsWith("Tests", StringComparison.OrdinalIgnoreCase)
        || name.Equals("test", StringComparison.OrdinalIgnoreCase);

    private static IReadOnlyList<AllowlistEntry> LoadAllowlist()
    {
        var path = Path.Combine(RepoRoot, AllowlistPath);
        if (!File.Exists(path)) return [];
        var file = JsonSerializer.Deserialize<AllowlistFile>(File.ReadAllText(path), JsonOptions);
        return file?.Entries ?? [];
    }

    private static string FindRepoRoot()
    {
        for (var dir = new DirectoryInfo(AppContext.BaseDirectory); dir is not null; dir = dir.Parent)
        {
            var git = Path.Combine(dir.FullName, ".git");
            if (Directory.Exists(git) || File.Exists(git) || File.Exists(Path.Combine(dir.FullName, AllowlistPath)))
                return dir.FullName;
        }
        throw new InvalidOperationException("Repository root not found from " + AppContext.BaseDirectory);
    }
}
```

## Allowlist file
Location confirmed at setup. Start with no entries:
```json
{ "entries": [] }
```
Each entry is a reviewed exception. The approver and review date come from the user:
```json
{
  "entries": [
    {
      "rule": "EF-SENSITIVE-LOGGING",
      "file": "src/<Infrastructure project>/DependencyInjection.cs",
      "contains": "EnableSensitiveDataLogging",
      "reason": "Called only inside the IsDevelopment() branch",
      "approvedBy": "<github-handle>",
      "reviewBy": "<yyyy-mm-dd>"
    }
  ]
}
```
- `file` is the repo-relative path with forward slashes; `contains` must appear on the matched line.
- Incomplete, expired, unknown-rule and stale entries fail the tests. Exceptions therefore expire, and none of them lingers after the code is gone.
- The allowlist is a guardrail file, protected by `CODEOWNERS`.

## Known limits (state them; do not hide them)
- **Line-based.** Code split across lines, or deliberately disguised, can slip past. Runtime tests, analyzers and code review remain the other layers.
- **Patterns, not meaning.** A match is a signal for review; a missing match is not proof that the code is safe.
- **New rules or changed patterns** are guardrail changes. Run the baseline scan first, so a new rule doesn't suddenly block everyone.
