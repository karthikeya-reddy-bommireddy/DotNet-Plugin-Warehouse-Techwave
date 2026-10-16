# Policies, handlers and the current-user adapter

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## 1. Registration (API layer)
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.Identity.Web;

public static class AppRoles
{
    public const string Admin = "Api.Admin";   // value defined in the app registration's app roles
}

public static class AuthorizationSetup
{
    public static IServiceCollection AddApiAuthorization(
        this IServiceCollection services, IConfiguration configuration)
    {
        string[] scopes = Split(configuration["AzureAd:Scopes"]);
        string[] appPermissions = Split(configuration["AzureAd:AppPermissions"]);
        if (scopes.Length == 0 && appPermissions.Length == 0)
            throw new InvalidOperationException(
                "Configure AzureAd:Scopes and/or AzureAd:AppPermissions; the API must not accept tokens without permissions.");

        // Baseline: a token for this API is not enough; it must carry one of this API's permissions.
        var baseline = new AuthorizationPolicyBuilder()
            .RequireAuthenticatedUser()
            .RequireScopeOrAppPermission(scopes, appPermissions)
            .Build();

        services.AddAuthorizationBuilder()
            .SetFallbackPolicy(baseline)        // endpoints without authorization metadata
            .SetDefaultPolicy(baseline)         // bare [Authorize]
            .AddPolicy(Policies.OrdersRead, p => p
                .RequireAuthenticatedUser()
                .RequireScopeOrAppPermission(["Orders.Read"], ["Orders.Read.All"]))
            .AddPolicy(Policies.OrdersWrite, p => p
                .RequireAuthenticatedUser()
                .RequireScope("Orders.Write"))                  // delegated only: no daemon writes
            .AddPolicy(Policies.Admin, p => p
                .RequireAuthenticatedUser()
                .RequireScope("Orders.Write")
                .RequireRole(AppRoles.Admin));

        services.AddSingleton<IAuthorizationHandler, OwnedResourceAuthorizationHandler>();
        services.AddHttpContextAccessor();
        services.AddScoped<ICurrentUser, HttpContextCurrentUser>();
        return services;

        static string[] Split(string? value) =>
            (value ?? string.Empty).Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);
    }
}
```
`RequireScope` and `RequireScopeOrAppPermission` come from Microsoft.Identity.Web. A named policy replaces the fallback for its endpoint, so each named policy repeats the baseline requirements.

## 2. Current-user adapter
The interface belongs to the Application layer (it may already exist under another name; reuse it). The API layer implements it:
```csharp
using System.Security.Claims;
using Microsoft.Identity.Web;

// Application layer (port), shown for reference
public interface ICurrentUser
{
    string ObjectId { get; }
    string TenantId { get; }
    bool IsAppOnly { get; }
    bool IsInRole(string role);
}

// API layer (adapter)
internal sealed class HttpContextCurrentUser(IHttpContextAccessor accessor) : ICurrentUser
{
    private ClaimsPrincipal User =>
        accessor.HttpContext?.User ?? throw new InvalidOperationException("No current HTTP request.");

    public string ObjectId => User.GetObjectId()
        ?? throw new UnauthorizedAccessException("Token has no object ID.");

    public string TenantId => User.GetTenantId()
        ?? throw new UnauthorizedAccessException("Token has no tenant ID.");

    public bool IsAppOnly =>
        !User.HasClaim(c => c.Type is ClaimConstants.Scp or ClaimConstants.Scope);

    public bool IsInRole(string role) => User.IsInRole(role);
}
```
Missing `oid`/`tid` throws (fail closed); the global exception handler maps `UnauthorizedAccessException` to 403.

## 3. Resource-based authorization
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Authorization.Infrastructure;
using Microsoft.Identity.Web;

public static class Operations
{
    public static readonly OperationAuthorizationRequirement Read   = new() { Name = nameof(Read) };
    public static readonly OperationAuthorizationRequirement Update = new() { Name = nameof(Update) };
    public static readonly OperationAuthorizationRequirement Delete = new() { Name = nameof(Delete) };
}

// Minimal ownership view returned by an Application query (not the entity)
public sealed record OwnedResource(string OwnerObjectId, string TenantId);

internal sealed class OwnedResourceAuthorizationHandler
    : AuthorizationHandler<OperationAuthorizationRequirement, OwnedResource>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OperationAuthorizationRequirement requirement,
        OwnedResource resource)
    {
        var tenantId = context.User.GetTenantId();
        var objectId = context.User.GetObjectId();

        var sameTenant = tenantId is not null
            && string.Equals(tenantId, resource.TenantId, StringComparison.OrdinalIgnoreCase);
        var isOwner = objectId is not null
            && string.Equals(objectId, resource.OwnerObjectId, StringComparison.OrdinalIgnoreCase);
        var isAdmin = context.User.IsInRole(AppRoles.Admin);

        // Example rule: owners may do everything; tenant admins may read and update, not delete.
        if (sameTenant && (isOwner || (isAdmin && requirement != Operations.Delete)))
            context.Succeed(requirement);

        return Task.CompletedTask;   // not succeeding is a denial
    }
}
```
Controller usage (use cases injected through the controller's constructor):
```csharp
[HttpPut("{id:guid}")]
[Authorize(Policy = Policies.OrdersWrite)]
public async Task<IActionResult> Update(
    [FromRoute] Guid id, [FromBody] UpdateOrderRequest request,
    [FromServices] IAuthorizationService authorization, CancellationToken ct)
{
    var ownership = await getOrderOwnership.ExecuteAsync(id, ct);   // OwnedResource?
    if (ownership is null) return NotFound();

    var result = await authorization.AuthorizeAsync(User, ownership, Operations.Update);
    if (!result.Succeeded) return NotFound();                        // do not reveal existence

    await updateOrder.ExecuteAsync(id, request, ct);
    return NoContent();
}
```
The Application command should still verify ownership when it writes (defence in depth): the resource can change between check and write.

## 4. Minimal APIs
```csharp
var orders = app.MapGroup("/api/v1/orders").RequireAuthorization(Policies.OrdersRead);
orders.MapPost("/", CreateOrder).RequireAuthorization(Policies.OrdersWrite);
```
