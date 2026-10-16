# Security test catalog

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

Examples use xUnit, `ApiFactory` and `CreateClientAs` from test-infrastructure.md. Adapt routes, DTOs, use-case interfaces and seeded IDs to the repo. On xUnit v3, pass `TestContext.Current.CancellationToken` to async calls.

Contents: 1 authorization matrix · 2 mass assignment · 3 validation · 4 response allowlist · 5 endpoint inventory · 6 error leakage · 7 CORS and headers · 8 rate limits · 9 JWT options · 10 staging smoke tests

## 1. Authorization matrix
```csharp
public static class AuthorizationMatrix
{
    public static readonly Guid OrderX = Guid.Parse("11111111-1111-1111-1111-111111111111"); // seeded, owned by "owner"

    // identity, method, path, JSON body (null = none), expected status: one row per matrix cell
    public static TheoryData<string, string, string, string?, int> Rows => new()
    {
        { "anon",         "GET",    $"/api/v1/orders/{OrderX}", null, 401 },
        { "no-perm",      "GET",    $"/api/v1/orders/{OrderX}", null, 403 },
        { "owner",        "GET",    $"/api/v1/orders/{OrderX}", null, 200 },
        { "reader",       "GET",    $"/api/v1/orders/{OrderX}", null, 404 },
        { "other-user",   "GET",    $"/api/v1/orders/{OrderX}", null, 404 },
        { "other-tenant", "GET",    $"/api/v1/orders/{OrderX}", null, 404 },
        { "reader",       "PUT",    $"/api/v1/orders/{OrderX}", TestBodies.ValidUpdate, 403 },
        { "other-user",   "PUT",    $"/api/v1/orders/{OrderX}", TestBodies.ValidUpdate, 404 },
        { "other-tenant", "DELETE", $"/api/v1/orders/{OrderX}", null, 404 },
        { "daemon",       "POST",   "/api/v1/orders",           TestBodies.ValidCreate, 403 },
    };
}

public static class TestBodies
{
    public const string ValidLine = """{"productId":"22222222-2222-2222-2222-222222222222","quantity":1}""";
    public const string ValidCreate = $$"""{"customerReference":"REF-1","lines":[{{ValidLine}}]}""";
    public const string ValidUpdate = """{"notes":"updated"}""";

    public static string Create(string reference, string lines, string extra = "") =>
        $$"""{"customerReference":"{{reference}}","lines":[{{lines}}]{{extra}}}""";
}

public sealed class AuthorizationMatrixTests(ApiFactory factory) : IClassFixture<ApiFactory>
{
    [Theory]
    [MemberData(nameof(AuthorizationMatrix.Rows), MemberType = typeof(AuthorizationMatrix))]
    public async Task Caller_gets_expected_status(string identity, string method, string path, string? body, int expected)
    {
        var client = identity == "anon" ? factory.CreateClient() : factory.CreateClientAs(identity);
        using var request = new HttpRequestMessage(new HttpMethod(method), path);
        if (body is not null) request.Content = new StringContent(body, Encoding.UTF8, "application/json");

        using var response = await client.SendAsync(request);

        Assert.Equal(expected, (int)response.StatusCode);
    }
}
```
- Rows that succeed and change state (owner PUT or DELETE) go in separate tests with their own seeded resource, so no row depends on execution order.
- API slice mode proves the 401/403 cells; the 404 (BOLA) cells need full-stack mode with seeded data.
- Keep `other-tenant` rows only if the project is confirmed multi-tenant.

## 2. Mass assignment
```csharp
[Fact]
public async Task Create_does_not_accept_server_controlled_fields()
{
    var owner = factory.CreateClientAs("owner");
    var body = TestBodies.Create("REF-2", TestBodies.ValidLine,
        ""","ownerObjectId":"00000000-0000-0000-0000-0000000000b1","tenantId":"bbbbbbbb-0000-0000-0000-000000000002","status":"Approved","total":0.01""");

    using var response = await owner.PostAsync("/api/v1/orders", new StringContent(body, Encoding.UTF8, "application/json"));

    // Team convention "ignore unknown members". With UnmappedMemberHandling.Disallow, assert 400 instead.
    Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    var location = response.Headers.Location!;
    Assert.Equal(HttpStatusCode.OK, (await owner.GetAsync(location)).StatusCode);
    Assert.Equal(HttpStatusCode.NotFound,
        (await factory.CreateClientAs("other-tenant").GetAsync(location)).StatusCode);   // tenant came from claims
    // Also assert status and total in the response are server-computed, not "Approved" / 0.01.
}
```

## 3. Validation
```csharp
public static TheoryData<string, string> InvalidCreates => new()
{
    { "reference missing",     $$"""{"lines":[{{TestBodies.ValidLine}}]}""" },
    { "reference too long",    TestBodies.Create(new string('a', 65), TestBodies.ValidLine) },
    { "reference bad chars",   TestBodies.Create("<script>", TestBodies.ValidLine) },
    { "no lines",              TestBodies.Create("REF-3", "") },
    { "too many lines",        TestBodies.Create("REF-3", string.Join(",", Enumerable.Repeat(TestBodies.ValidLine, 51))) },
    { "empty product id",      TestBodies.Create("REF-3", """{"productId":"00000000-0000-0000-0000-000000000000","quantity":1}""") },
    { "quantity zero",         TestBodies.Create("REF-3", """{"productId":"22222222-2222-2222-2222-222222222222","quantity":0}""") },
    { "undefined enum",        TestBodies.Create("REF-3", TestBodies.ValidLine, ""","delivery":999""") },
    { "malformed json",        """{"customerReference":""" },
};

[Theory]
[MemberData(nameof(InvalidCreates))]
public async Task Invalid_create_returns_400_without_internals(string caseName, string body)
{
    var client = factory.CreateClientAs("owner");
    using var response = await client.PostAsync("/api/v1/orders", new StringContent(body, Encoding.UTF8, "application/json"));
    var text = await response.Content.ReadAsStringAsync();

    Assert.True(response.StatusCode == HttpStatusCode.BadRequest, caseName);
    Assert.Equal("application/problem+json", response.Content.Headers.ContentType?.MediaType);
    Assert.DoesNotContain("System.", text);       // no type names from serializer exceptions
    Assert.DoesNotContain("Exception", text);
}
```
Add query cases the same way: `?pageSize=100000`, `?page=0`, `?sort=999` → 400.

## 4. Response field allowlist
```csharp
[Fact]
public async Task Order_response_exposes_only_allowlisted_fields()
{
    var client = factory.CreateClientAs("owner");
    using var response = await client.GetAsync($"/api/v1/orders/{AuthorizationMatrix.OrderX}");
    response.EnsureSuccessStatusCode();

    using var json = JsonDocument.Parse(await response.Content.ReadAsStringAsync());
    var actual = json.RootElement.EnumerateObject().Select(p => p.Name).Order().ToArray();

    string[] allowed = ["createdAt", "customerReference", "delivery", "id", "lines", "status", "total"];
    Assert.Equal(allowed.Order(), actual);   // a new field fails until someone reviews its exposure
}
```

## 5. Endpoint inventory
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Routing;
using Microsoft.Extensions.DependencyInjection;

[Fact]
public void Every_endpoint_names_a_policy_or_is_allowlisted_anonymous()
{
    // Keys look like "GET api/v{version:apiVersion}/orders/{id:guid}". MapHealthChecks has no method: "ANY".
    string[] anonymousAllowlist = ["ANY health/live"];

    var endpoints = factory.Services.GetRequiredService<EndpointDataSource>().Endpoints.OfType<RouteEndpoint>();
    var problems = new List<string>();

    foreach (var endpoint in endpoints)
    {
        var methods = endpoint.Metadata.GetMetadata<IHttpMethodMetadata>()?.HttpMethods ?? new[] { "ANY" };
        var route = endpoint.RoutePattern.RawText?.TrimStart('/') ?? "?";
        var anonymous = endpoint.Metadata.GetMetadata<IAllowAnonymous>() is not null;
        var hasNamedPolicy = endpoint.Metadata.GetOrderedMetadata<IAuthorizeData>()
            .Any(a => !string.IsNullOrWhiteSpace(a.Policy));

        foreach (var key in methods.Select(m => $"{m} {route}"))
        {
            if (anonymous && !anonymousAllowlist.Contains(key)) problems.Add($"{key}: anonymous, not allowlisted");
            else if (!anonymous && !hasNamedPolicy) problems.Add($"{key}: no named policy");
        }
    }

    Assert.Empty(problems);
}
```
A bare `[Authorize]` fails this test on purpose: every endpoint names its policy.

## 6. Error leakage
```csharp
public sealed class ErrorLeakageTests
{
    [Fact]
    public async Task Unhandled_exception_returns_generic_problem_details()
    {
        await using var baseFactory = new ApiFactory();
        var factory = baseFactory.WithWebHostBuilder(b => b.ConfigureTestServices(s =>
            s.AddScoped<IGetOrderUseCase, ThrowingGetOrderUseCase>()));

        using var response = await factory.CreateClientAs("owner").GetAsync($"/api/v1/orders/{Guid.NewGuid()}");
        var body = await response.Content.ReadAsStringAsync();

        Assert.Equal(HttpStatusCode.InternalServerError, response.StatusCode);
        Assert.Equal("application/problem+json", response.Content.Headers.ContentType?.MediaType);
        Assert.Contains("traceId", body);
        Assert.DoesNotContain("hunter2", body);                    // exception message
        Assert.DoesNotContain("InvalidOperationException", body);  // type name
        Assert.DoesNotContain(" at ", body);                       // stack frames
    }

    private sealed class ThrowingGetOrderUseCase : IGetOrderUseCase   // match the real interface
    {
        public Task<OrderResponse?> ExecuteAsync(Guid id, CancellationToken ct) =>
            throw new InvalidOperationException("Login failed; password=hunter2; Server=prod-sql");
    }
}
```

## 7. CORS and security headers
```csharp
[Fact]
public async Task Cors_allows_only_configured_origins()
{
    var client = factory.CreateClientAs("owner");

    using var preflight = new HttpRequestMessage(HttpMethod.Options, "/api/v1/orders");
    preflight.Headers.Add("Origin", "https://web.test.example");            // from appsettings.Testing.json
    preflight.Headers.Add("Access-Control-Request-Method", "POST");
    preflight.Headers.Add("Access-Control-Request-Headers", "authorization,content-type");
    using var allowed = await client.SendAsync(preflight);
    Assert.Equal("https://web.test.example", allowed.Headers.GetValues("Access-Control-Allow-Origin").Single());
    Assert.False(allowed.Headers.Contains("Access-Control-Allow-Credentials"));

    using var foreign = new HttpRequestMessage(HttpMethod.Get, "/api/v1/orders");
    foreign.Headers.Add("Origin", "https://evil.example");
    using var denied = await client.SendAsync(foreign);
    Assert.False(denied.Headers.Contains("Access-Control-Allow-Origin"));
}

[Fact]
public async Task Responses_carry_security_headers()
{
    using var response = await factory.CreateClientAs("owner").GetAsync("/api/v1/orders");

    Assert.Equal("nosniff", response.Headers.GetValues("X-Content-Type-Options").Single());
    Assert.Equal("DENY", response.Headers.GetValues("X-Frame-Options").Single());
    Assert.Contains("frame-ancestors 'none'", response.Headers.GetValues("Content-Security-Policy").Single());
    Assert.True(response.Headers.CacheControl?.NoStore);
}

[Fact]
public async Task Swagger_is_not_served_when_disabled()
{
    // As an authenticated caller: anonymous callers get 401 for every path because of the fallback policy.
    using var response = await factory.CreateClientAs("owner").GetAsync("/swagger/v1/swagger.json");
    Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
}
```
If HTTPS redirection interferes with tests, create clients with `BaseAddress = new Uri("https://localhost")`.

## 8. Rate limits
```csharp
[Fact]
public async Task Writes_are_rate_limited_per_user()
{
    await using var factory = new ApiFactory();   // fresh limiter state; never share with other tests
    var client = factory.CreateClientAs("owner");
    HttpResponseMessage? limited = null;

    for (var i = 0; i < 40 && limited is null; i++)   // Writes policy allows 30 per minute
    {
        var response = await client.PostAsync("/api/v1/orders",
            new StringContent(TestBodies.ValidCreate, Encoding.UTF8, "application/json"));
        if (response.StatusCode == HttpStatusCode.TooManyRequests) limited = response;
        else response.Dispose();
    }

    Assert.NotNull(limited);
    Assert.True(limited.Headers.Contains("Retry-After"));
    Assert.Equal("application/problem+json", limited.Content.Headers.ContentType?.MediaType);
}
```
If limits come from configuration, set small values in `appsettings.Testing.json` and loop fewer times.

## 9. JWT options guard
```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.Extensions.Options;

[Fact]
public void Jwt_bearer_validation_is_not_weakened()
{
    var options = factory.Services.GetRequiredService<IOptionsMonitor<JwtBearerOptions>>()
        .Get(JwtBearerDefaults.AuthenticationScheme);
    var p = options.TokenValidationParameters;

    Assert.True(options.RequireHttpsMetadata);
    Assert.True(p.ValidateIssuer);
    Assert.True(p.ValidateAudience);
    Assert.True(p.ValidateLifetime);
    Assert.True(p.RequireSignedTokens);
    Assert.True(p.RequireExpirationTime);
    Assert.Null(p.SignatureValidator);              // nobody replaced signature validation
    Assert.True(p.ClockSkew <= TimeSpan.FromMinutes(5));
}
```
Do not assert `IssuerValidator`/`AudienceValidator` are null: Microsoft.Identity.Web sets its own validators.

AutoMapper configuration test: api-input-validation/references/automapper-safety.md, section 4.

## 10. Staging smoke tests (pipeline step against the staging slot, before swap)
```powershell
$base = $env:API_BASE_URL
$r = Invoke-WebRequest "$base/api/v1/orders" -SkipHttpErrorCheck
if ($r.StatusCode -ne 401) { throw "Expected 401 without a token, got $($r.StatusCode)" }
if (-not $r.Headers['Strict-Transport-Security']) { throw "HSTS missing (check forwarded headers)" }
if ($r.Headers['Server'] -or $r.Headers['X-Powered-By']) { throw "Server banner headers present" }

$s = Invoke-WebRequest "$base/swagger/v1/swagger.json" -SkipHttpErrorCheck
if ($s.StatusCode -eq 200) { throw "Swagger is exposed" }

$h = Invoke-WebRequest "$base/health/live" -SkipHttpErrorCheck
if ($h.StatusCode -ne 200) { throw "Liveness check failed" }
```
Runs in a `pwsh` step (PowerShell 7) in Azure Pipelines. Add the CORS check from section 7 with the real web origin.
