# Test infrastructure

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Factory
```csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.TestHost;

public sealed class ApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        // appsettings.Testing.json: fictional AzureAd values (incl. Scopes/AppPermissions), test CORS origins,
        // Swagger:Enabled=false, and no KeyVault:Uri.
        builder.UseEnvironment("Testing");

        builder.ConfigureTestServices(services =>
        {
            services.AddAuthentication(o =>
                {
                    o.DefaultAuthenticateScheme = TestAuthHandler.Scheme;
                    o.DefaultChallengeScheme = TestAuthHandler.Scheme;
                })
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(TestAuthHandler.Scheme, _ => { });

            // API slice mode: replace Application use cases with fakes here.
            // Full-stack mode: point the DbContext at the Testcontainers connection string instead.
        });
    }
}

public static class FactoryExtensions
{
    public static HttpClient CreateClientAs(this WebApplicationFactory<Program> factory, string identityKey)
    {
        var client = factory.CreateClient();
        client.DefaultRequestHeaders.Add(TestAuthHandler.Header, identityKey);
        return client;
    }
}
```
- .NET 10 generates `public partial class Program` for top-level statements; on older SDKs add `public partial class Program;` to the API project.
- Program.cs reads some settings before `Build()`. Values supplied through the factory's `ConfigureAppConfiguration` arrive too late for that code; use `appsettings.Testing.json` or `builder.UseSetting(key, value)`.
- Keep policies scheme-free; a policy that names a scheme bypasses the test default.

## Test authentication handler (test project only)
```csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

public sealed class TestAuthHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder) : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    public const string Scheme = "Test";
    public const string Header = "X-Test-Identity";

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(Header, out var key)
            || !TestIdentities.All.TryGetValue(key.ToString(), out var claims))
            return Task.FromResult(AuthenticateResult.NoResult());   // → normal 401 challenge

        var identity = new ClaimsIdentity(claims, Scheme, nameType: "name", roleType: "roles");
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(identity), Scheme);
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}

public static class TestIdentities
{
    public const string TenantA = "aaaaaaaa-0000-0000-0000-000000000001";
    public const string TenantB = "bbbbbbbb-0000-0000-0000-000000000002";

    public static readonly IReadOnlyDictionary<string, Claim[]> All = new Dictionary<string, Claim[]>
    {
        ["owner"]        = User("a1", TenantA, "Orders.Read Orders.Write"),
        ["reader"]       = User("a2", TenantA, "Orders.Read"),
        ["other-user"]   = User("a3", TenantA, "Orders.Read Orders.Write"),
        ["other-tenant"] = User("b1", TenantB, "Orders.Read Orders.Write"),
        ["no-perm"]      = User("a4", TenantA, scopes: null),
        ["daemon"]       = App("d1", TenantA, "Orders.Read.All"),
        ["admin"]        = User("a5", TenantA, "Orders.Read Orders.Write", "Api.Admin"),
    };

    private static Claim[] User(string seed, string tenantId, string? scopes, params string[] roles)
    {
        var claims = new List<Claim> { new("oid", Oid(seed)), new("tid", tenantId), new("name", $"Test user {seed}") };
        if (scopes is not null) claims.Add(new Claim("scp", scopes));
        claims.AddRange(roles.Select(r => new Claim("roles", r)));
        return [.. claims];
    }

    private static Claim[] App(string seed, string tenantId, params string[] appPermissions)
    {
        var claims = new List<Claim> { new("oid", Oid(seed)), new("tid", tenantId), new("idtyp", "app") };
        claims.AddRange(appPermissions.Select(p => new Claim("roles", p)));
        return [.. claims];
    }

    public static string Oid(string seed) => $"00000000-0000-0000-0000-{seed.PadLeft(12, '0')}";
}
```
The header exists only because this handler is registered in the test project; production never reads it.

## SQL Server with Testcontainers (full-stack mode)
```csharp
using Testcontainers.MsSql;

public sealed class SqlServerFixture : IAsyncLifetime
{
    // Pin the image tag the team approves (builder API differs slightly between Testcontainers versions).
    public MsSqlContainer Sql { get; } = new MsSqlBuilder().Build();

    public async ValueTask InitializeAsync() => await Sql.StartAsync();   // xUnit v3 signatures
    public async ValueTask DisposeAsync() => await Sql.DisposeAsync();     // xUnit v2: return Task instead
}
```
Apply migrations once per fixture and seed `owner`, `other-user` and `other-tenant` data with fictional IDs (`TestIdentities.Oid(...)`).
