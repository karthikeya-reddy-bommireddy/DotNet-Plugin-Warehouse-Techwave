# Security event catalog

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

Stable event IDs make alerting and investigations possible. Log identifiers and categories only.

| EventId | Name | Level | When | Properties |
|---|---|---|---|---|
| 4001 | TokenRejected | Warning | JWT validation failed | ExceptionType, Path |
| 4002 | AuthenticationChallenged | Information | 401 issued | Method, Path |
| 4003 | AuthorizationDenied | Warning | Policy failed (403) | UserObjectId, ClientAppId, FailedRequirements, Method, Path |
| 4004 | ObjectAccessDenied | Warning | Resource-based check denied (served as 404) | UserObjectId, ResourceType, ResourceId |
| 4005 | ValidationRejected | Information | 400 from model validation | Path, ErrorCount (never the values) |
| 4006 | RateLimited | Warning | 429 issued | PartitionKind (user/ip), Path |
| 4007 | PrivilegedAction | Information | Admin or high-impact operation succeeded | Action, UserObjectId, TargetId |
| 4008 | SuspiciousInput | Warning | Unknown members, oversized payload, blocked outbound URL | Path, Reason |
| 4009 | UnhandledException | Error | 500 from the global handler | ExceptionType, TraceId |

## Helpers (source-generated)
```csharp
using Microsoft.Extensions.Logging;

internal static partial class SecurityLog
{
    [LoggerMessage(EventId = 4001, Level = LogLevel.Warning,
        Message = "Token rejected ({ExceptionType}) on {Path}")]
    public static partial void TokenRejected(ILogger logger, string exceptionType, string path);

    [LoggerMessage(EventId = 4002, Level = LogLevel.Information,
        Message = "Authentication required for {Method} {Path}")]
    public static partial void AuthenticationChallenged(ILogger logger, string method, string path);

    [LoggerMessage(EventId = 4003, Level = LogLevel.Warning,
        Message = "Authorization denied for {UserObjectId} via {ClientAppId}: {FailedRequirements} on {Method} {Path}")]
    public static partial void AuthorizationDenied(ILogger logger, string? userObjectId, string? clientAppId,
        string failedRequirements, string method, string path);

    [LoggerMessage(EventId = 4004, Level = LogLevel.Warning,
        Message = "Object access denied for {UserObjectId} on {ResourceType} {ResourceId}")]
    public static partial void ObjectAccessDenied(ILogger logger, string? userObjectId, string resourceType, string resourceId);

    [LoggerMessage(EventId = 4007, Level = LogLevel.Information,
        Message = "Privileged action {Action} by {UserObjectId} on {TargetId}")]
    public static partial void PrivilegedAction(ILogger logger, string action, string? userObjectId, string targetId);
}
```
Serilog receives these through Microsoft.Extensions.Logging; the event ID is kept as the `EventId` property.

## Logging authorization outcomes centrally
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Authorization.Policy;
using Microsoft.Identity.Web;

internal sealed class AuditingAuthorizationResultHandler(
    ILogger<AuditingAuthorizationResultHandler> logger) : IAuthorizationMiddlewareResultHandler
{
    private readonly AuthorizationMiddlewareResultHandler _default = new();

    public async Task HandleAsync(RequestDelegate next, HttpContext context,
        AuthorizationPolicy policy, PolicyAuthorizationResult authorizeResult)
    {
        if (authorizeResult.Forbidden)
        {
            var failed = authorizeResult.AuthorizationFailure?.FailedRequirements
                .Select(r => r.GetType().Name).Distinct() ?? Enumerable.Empty<string>();
            SecurityLog.AuthorizationDenied(logger,
                context.User.GetObjectId(),
                context.User.FindFirst("azp")?.Value ?? context.User.FindFirst("appid")?.Value,
                string.Join(",", failed), context.Request.Method, context.Request.Path);
        }
        else if (authorizeResult.Challenged)
        {
            SecurityLog.AuthenticationChallenged(logger, context.Request.Method, context.Request.Path);
        }

        await _default.HandleAsync(next, context, policy, authorizeResult);
    }
}

// Registration (API layer):
// builder.Services.AddSingleton<IAuthorizationMiddlewareResultHandler, AuditingAuthorizationResultHandler>();
```
Log `ObjectAccessDenied` where a resource-based check denies access; owner-scoped queries cannot tell "missing" from "not yours", so they do not log it.
