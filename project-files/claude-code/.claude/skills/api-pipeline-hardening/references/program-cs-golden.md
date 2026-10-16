# Golden Program.cs (API layer composition root)

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

Adapt to the repo; keep the order. Every number in the limits below is a placeholder: use the confirmed value from project-facts or ask the user, and bind it from configuration.

## Program.cs
```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Identity.Web;
using Serilog;

Log.Logger = new LoggerConfiguration().WriteTo.Console().CreateBootstrapLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);

    builder.AddApiConfiguration();                       // Key Vault + options validation (secrets-config-outbound.md)

    builder.Services.AddSerilog((services, lc) => lc     // api-secure-logging
        .ReadFrom.Configuration(builder.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext());

    builder.WebHost.ConfigureKestrel(o =>                // ignored under in-process IIS: use IIS limits there
    {
        o.AddServerHeader = false;
        o.Limits.MaxRequestBodySize = 1 * 1024 * 1024;   // raise per endpoint with [RequestSizeLimit]
    });

    builder.Services.AddProblemDetails();                // adds traceId to every ProblemDetails
    builder.Services.AddExceptionHandler<GlobalExceptionHandler>();

    builder.Services
        .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));
    builder.Services.AddApiAuthorization(builder.Configuration);   // api-authorization

    builder.Services.AddApiCors(builder.Configuration);
    builder.Services.AddApiRateLimiting();
    builder.Services.AddRequestTimeouts(o => o.DefaultPolicy = new() { Timeout = TimeSpan.FromSeconds(30) });

    builder.Services
        .AddControllers()
        .AddJsonOptions(o =>                             // api-input-validation
        {
            o.JsonSerializerOptions.MaxDepth = 32;
            o.AllowInputFormatterExceptionMessages = false;
        });

    // Versioning: api-endpoint-design/references/http-patterns.md
    // Swagger (non-Production only): api-endpoint-design/references/openapi-swashbuckle.md
    builder.Services.AddHealthChecks();

    var app = builder.Build();

    app.UseForwardedHeaders();                           // configure trusted proxies for your hosting first
    app.UseExceptionHandler();
    if (!app.Environment.IsDevelopment())
    {
        app.UseHsts();
    }
    app.UseHttpsRedirection();
    app.UseSecurityHeaders();
    app.UseSerilogRequestLogging();
    app.UseCors(CorsPolicies.Clients);
    app.UseAuthentication();
    app.UseRateLimiter();
    app.UseAuthorization();
    app.UseRequestTimeouts();

    app.MapControllers();
    app.MapHealthChecks("/health/live", new HealthCheckOptions { Predicate = _ => false })
        .AllowAnonymous()
        .DisableRateLimiting();

    app.Run();
}
catch (Exception ex) when (ex is not HostAbortedException)
{
    Log.Fatal(ex, "API host terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```
With a fallback policy, even unknown paths answer 401 to anonymous callers; that is expected.

## CORS
```csharp
public static class CorsPolicies { public const string Clients = "clients"; }

public static class CorsSetup
{
    public static IServiceCollection AddApiCors(this IServiceCollection services, IConfiguration configuration)
    {
        var origins = configuration.GetSection("Cors:AllowedOrigins").Get<string[]>() ?? [];
        if (!origins.All(IsExplicitOrigin))
            throw new InvalidOperationException("Cors:AllowedOrigins must list explicit https origins.");

        return services.AddCors(o => o.AddPolicy(CorsPolicies.Clients, p => p
            .WithOrigins(origins)
            .WithMethods("GET", "POST", "PUT", "PATCH", "DELETE")
            .WithHeaders("Authorization", "Content-Type", "If-Match", "Idempotency-Key")
            .WithExposedHeaders("ETag", "Location", "Retry-After")
            .SetPreflightMaxAge(TimeSpan.FromHours(1))));

        static bool IsExplicitOrigin(string origin) =>
            Uri.TryCreate(origin, UriKind.Absolute, out var uri)
            && (uri.Scheme == Uri.UriSchemeHttps || uri.IsLoopback)   // loopback only for local dev configs
            && uri.AbsolutePath == "/";
    }
}
```

## Security headers
```csharp
public static class SecurityHeaders
{
    public static IApplicationBuilder UseSecurityHeaders(this IApplicationBuilder app) =>
        app.Use(async (context, next) =>
        {
            var headers = context.Response.Headers;
            headers.XContentTypeOptions = "nosniff";
            headers.XFrameOptions = "DENY";
            headers["Referrer-Policy"] = "no-referrer";
            headers.CacheControl = "no-store";            // endpoints that may be cached override this
            if (!context.Request.Path.StartsWithSegments("/swagger"))
                headers.ContentSecurityPolicy = "default-src 'none'; frame-ancestors 'none'";
            await next(context);
        });
}
```
On Windows/IIS hosting, also remove `Server` and `X-Powered-By` through `web.config`.

## Rate limiting
```csharp
using System.Globalization;
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Identity.Web;

public static class RateLimitPolicies
{
    public const string Writes = "writes";
    public const string Expensive = "expensive";
}

public static class RateLimitingSetup
{
    public static IServiceCollection AddApiRateLimiting(this IServiceCollection services) =>
        services.AddRateLimiter(o =>
        {
            o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
            o.OnRejected = async (context, _) =>
            {
                // Not every limiter type supplies a retry hint; always send one.
                var retryAfter = context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var hint)
                    ? hint : TimeSpan.FromSeconds(60);
                context.HttpContext.Response.Headers.RetryAfter =
                    ((int)Math.Ceiling(retryAfter.TotalSeconds)).ToString(CultureInfo.InvariantCulture);

                await context.HttpContext.RequestServices.GetRequiredService<IProblemDetailsService>()
                    .TryWriteAsync(new ProblemDetailsContext
                    {
                        HttpContext = context.HttpContext,
                        ProblemDetails = new ProblemDetails { Status = 429, Title = "Too many requests" }
                    });
            };

            o.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(http =>
                RateLimitPartition.GetTokenBucketLimiter(PartitionKey(http), _ => new TokenBucketRateLimiterOptions
                {
                    TokenLimit = 200, TokensPerPeriod = 100,
                    ReplenishmentPeriod = TimeSpan.FromSeconds(10),
                    QueueLimit = 0, AutoReplenishment = true
                }));

            o.AddPolicy(RateLimitPolicies.Writes, http =>
                RateLimitPartition.GetSlidingWindowLimiter(PartitionKey(http), _ => new SlidingWindowRateLimiterOptions
                {
                    PermitLimit = 30, Window = TimeSpan.FromMinutes(1), SegmentsPerWindow = 6, QueueLimit = 0
                }));

            o.AddPolicy(RateLimitPolicies.Expensive, http =>
                RateLimitPartition.GetFixedWindowLimiter(PartitionKey(http), _ => new FixedWindowRateLimiterOptions
                {
                    PermitLimit = 5, Window = TimeSpan.FromMinutes(1), QueueLimit = 0
                }));
        });

    // oid is stable per user. Behind a proxy the IP is right only if forwarded headers are configured.
    private static string PartitionKey(HttpContext http) =>
        http.User.GetObjectId() is { } oid ? $"user:{oid}" : $"ip:{http.Connection.RemoteIpAddress}";
}
```
These limiters are per instance; with several App Service instances the effective limit multiplies. Strict global quotas belong in Azure API Management or Front Door (raise a cross-repo change request).

## Global exception handler
```csharp
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

internal sealed class GlobalExceptionHandler(
    IProblemDetailsService problemDetails,
    ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext context, Exception exception, CancellationToken ct)
    {
        var (status, title) = exception switch
        {
            // Map the Application layer's exception types here (names are project-specific):
            // NotFoundException => (StatusCodes.Status404NotFound, "Not found"),
            // ConflictException => (StatusCodes.Status409Conflict, "Conflict"),
            UnauthorizedAccessException => (StatusCodes.Status403Forbidden, "Forbidden"),
            BadHttpRequestException bad => (bad.StatusCode, "Bad request"),
            _ => (StatusCodes.Status500InternalServerError, "An unexpected error occurred")
        };

        // Log here: recent ASP.NET Core versions may not log exceptions a handler reports as handled.
        if (status >= 500)
            logger.LogError(exception, "Unhandled exception for {Method} {Path}", context.Request.Method, context.Request.Path);
        else
            logger.LogWarning("Request failed with {StatusCode} ({ExceptionType})", status, exception.GetType().Name);

        context.Response.StatusCode = status;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = context,
            ProblemDetails = new ProblemDetails { Status = status, Title = title }   // never exception text
        });
    }
}
```
