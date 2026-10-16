# Layer dependency test and the guardrail test set

> **Placeholders:** type names (`Order`, `IGetOrderUseCase`, `AppDbContext`) and folder names are illustrations. Use one real public type from each layer, as recorded in project-facts.

## Guardrail tests and where their code comes from
| Test | What it proves | Code |
|---|---|---|
| Endpoint inventory | Every endpoint names a policy, or is an allowlisted anonymous endpoint | api-security-testing/references/security-test-catalog.md, section 5 |
| JWT options guard | Token validation has not been weakened | same file, section 9 |
| Error leakage | 500 responses are generic ProblemDetails | same file, section 6 |
| CORS | Only configured origins are allowed | same file, section 7 |
| AutoMapper configuration | Mapping configuration is valid | api-input-validation/references/automapper-safety.md, section 4 |
| Forbidden-pattern scan | No dangerous patterns without a reviewed exception | forbidden-patterns-test.md |
| Layer dependencies | Layers reference only what Clean Architecture allows | below |

Place them in a `Guardrails` folder of the API test project, or in a dedicated test project if the team prefers (ask). The examples assume `ImplicitUsings` is enabled, as in the default templates. All of these files are guardrail files.

## Layer dependency test (no extra packages)
```csharp
using System.Reflection;
using Xunit;

namespace Contoso.Api.Tests.Guardrails;

public sealed class LayerDependencyTests
{
    // One public type from each layer's assembly (from project-facts).
    private static readonly Assembly Domain = typeof(Order).Assembly;
    private static readonly Assembly Application = typeof(IGetOrderUseCase).Assembly;
    private static readonly Assembly Infrastructure = typeof(AppDbContext).Assembly;

    [Fact]
    public void Domain_depends_on_no_other_layer_or_framework() =>
        AssertNoReference(Domain,
            Name(Application), Name(Infrastructure),
            "Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore", "Microsoft.Data.SqlClient", "AutoMapper", "Serilog");

    [Fact]
    public void Application_does_not_depend_on_infrastructure_or_web() =>
        AssertNoReference(Application,
            Name(Infrastructure), "Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore", "Microsoft.Data.SqlClient");

    private static string Name(Assembly assembly) => assembly.GetName().Name!;

    private static void AssertNoReference(Assembly assembly, params string[] forbiddenPrefixes)
    {
        var offending = assembly.GetReferencedAssemblies()
            .Select(reference => reference.Name ?? string.Empty)
            .Where(name => forbiddenPrefixes.Any(prefix => name.StartsWith(prefix, StringComparison.Ordinal)))
            .ToList();

        Assert.True(offending.Count == 0, $"{Name(assembly)} must not reference: {string.Join(", ", offending)}");
    }
}
```
- `GetReferencedAssemblies` lists only assemblies the compiled code actually uses. It catches real usage, not project references nobody uses.
- Adjust the forbidden lists to the repo's confirmed rules (clean-architecture-rules). Relaxing a rule is a guardrail change.
- An architecture-testing package can express the same rules; choosing one is a team decision.

## Done when
- [ ] All guardrail tests exist and run in `dotnet test`
- [ ] They pass on the current code: violations fixed, or allowlisted with approval
- [ ] Their folder is listed in `CODEOWNERS`
