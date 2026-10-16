# Security review procedure (PR, branch or whole repo)

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

Use for "review", "audit", "check this PR/diff/branch", "is this secure". Stay read-only: report findings; fix only after the user approves a fix plan.

## 1. Scope
- Diff review: `git diff --stat <base>...HEAD`, then `git diff <base>...HEAD`. Review changed lines plus the code they call and the code that calls them.
- Repo audit: build the endpoint inventory first (step 2), then review by risk.
- Record what you could not see (other repos, Azure configuration, pipelines) as limits of the review.

## 2. Endpoint inventory
Search the code (regex) for:
- Controllers/actions: `\[(HttpGet|HttpPost|HttpPut|HttpPatch|HttpDelete)` and `class \w+Controller`
- Minimal APIs: `\.Map(Get|Post|Put|Patch|Delete|Group|Methods)\(`
- Authorization metadata: `\[Authorize|\[AllowAnonymous|RequireAuthorization|AllowAnonymous\(\)`

Build a table: route · method · policy · anonymous? · resource IDs · request DTO · response DTO.

## 3. Search recipes
Every hit needs judgement, not an automatic verdict.

| Concern | Pattern (regex) | Verify |
|---|---|---|
| Anonymous access | `AllowAnonymous` | Justified, allowlisted, not on a controller or base class |
| Bare authorize | `\[Authorize\]` | Default policy is the strict baseline; a named policy fits better |
| Weakened token validation | `Validate(Issuer\|Audience\|Lifetime)\s*=\s*false`, `RequireHttpsMetadata\s*=\s*false`, `SignatureValidator`, `IssuerValidator\s*=`, `ClockSkew` | Critical unless test-only |
| Identity from client input | `FromHeader`, `Request\.Headers\[`, action parameters named `userId`/`ownerId`/`tenantId` | Must come from claims |
| Entity binding or exposure | `\[FromBody\]` (check the types), `ReverseMap\(\)` | DTOs only |
| Mapping onto tracked entities | `_mapper\.Map\(\w+,\s*\w+\)` | Protected members ignored; configuration validated |
| Unbounded lists | parameters named `pageSize`, `take`, `limit`, `top` | `[Range]` caps |
| Error leakage | `ex\.Message`, `\.StackTrace`, `UseDeveloperExceptionPage`, `IncludeErrorDetails` | Generic ProblemDetails |
| CORS | `AllowAnyOrigin`, `SetIsOriginAllowed`, `AllowCredentials` | Explicit origins, no credentials |
| SQL injection | `FromSqlRaw`, `ExecuteSqlRaw`, `SqlQueryRaw` | No concatenation or `$"..."` inside them |
| Tenant filter bypass | `IgnoreQueryFilters` | Justified and tenant re-applied |
| Unsafe deserialization | `TypeNameHandling`, `BinaryFormatter` | Never driven by input |
| Secrets | `Password=`, `AccountKey=`, `SharedAccessSignature`, `ClientSecret`, `-----BEGIN` | Remove, rotate, move to Key Vault |
| Sensitive logging | `EnableSensitiveDataLogging`, `ShowPII`, `LogCompleteSecurityArtifact`, `Log\w*\(\$"` | Development only; message templates |
| SSRF | `GetAsync\(`, `PostAsync\(`, `new Uri\(` fed by request data | Allowlist; no user URLs |
| TLS disabled | `ServerCertificateCustomValidationCallback`, `DangerousAcceptAnyServerCertificateValidator` | Never outside tests |
| Weak crypto/randomness | `MD5`, `SHA1`, `new Random\(` in security code | SHA-256+, `RandomNumberGenerator` |
| Swagger exposure | `UseSwagger`, `MapSwagger`, `UseSwaggerUI` | Non-production only |
| Sync over async | `\.Result\b`, `\.Wait\(\)`, `GetAwaiter\(\)\.GetResult\(\)` | Availability risk |

## 4. Dependencies and configuration
- `dotnet list package --vulnerable --include-transitive` and `dotnet list package --deprecated` (needs feed access).
- Known issues for this stack: AutoMapper before 15.1.1, or 16.0.0-16.1.0, is affected by CVE-2026-32933 (process-killing DoS through deep recursion). Swashbuckle on .NET 10 should be v10+.
- Check every `appsettings*.json`, `launchSettings.json`, pipeline YAML and test fixture for secrets. A secret found in git history must be rotated, not just deleted.

## 5. Judge each endpoint
Apply the threat check and Security Gate (G1-G15) from SKILL.md to every endpoint and data-access change in scope.

## 6. Report
Severity rubric:
- **Critical**: unauthenticated remote exploitation, cross-tenant data access, secret exposure, authentication bypass.
- **High**: authenticated access to other users' data, privilege escalation, process-wide denial of service.
- **Medium**: information leakage, missing limits, weak configuration with compensating controls.
- **Low**: defence-in-depth gaps and hygiene.

Order findings by severity, using this template for each:

    ### [Severity] Title
    - Where: path/File.cs:line (endpoint if relevant)
    - Risk: what an attacker can do; OWASP API reference
    - Evidence: the code or configuration, quoted briefly
    - Fix: the concrete change, and the layer that owns it
    - Proof: the test or check that will show it is fixed

Finish with: endpoints reviewed, what was out of scope, and good practices worth keeping.
