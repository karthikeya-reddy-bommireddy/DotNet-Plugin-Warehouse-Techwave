# Entra ID setup for the API

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Program.cs
```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.Identity.Web;

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

// Log rejected tokens without leaking them (SecurityLog lives in api-secure-logging).
// Registered after AddMicrosoftIdentityWebApi so it runs after the library's own configuration.
builder.Services.Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, o =>
{
    o.Events ??= new JwtBearerEvents();
    var previous = o.Events.OnAuthenticationFailed;
    o.Events.OnAuthenticationFailed = async context =>
    {
        await previous(context);
        var logger = context.HttpContext.RequestServices
            .GetRequiredService<ILoggerFactory>().CreateLogger("Security");
        SecurityLog.TokenRejected(logger, context.Exception.GetType().Name, context.Request.Path);
    };
});

if (builder.Environment.IsDevelopment())
{
    Microsoft.IdentityModel.Logging.IdentityModelEventSource.ShowPII = true;   // never outside Development
}
```
Pipeline order: `UseAuthentication()` before `UseAuthorization()` (api-pipeline-hardening).

## Configuration per environment
- Separate app registrations for Dev, Test, Staging and Production.
- `TenantId`/`ClientId` are identifiers, not secrets, but keep per-environment values in App Service settings so builds stay environment-neutral.
- Credentials (only if the API calls downstream APIs): certificate in Key Vault or managed-identity federated credential; never a client secret in configuration.

## App registration checklist (agree with the Entra admin)
- [ ] "Expose an API": App ID URI set; delegated scopes with least privilege (e.g. `Orders.Read`, `Orders.Write`); admin consent required for sensitive scopes.
- [ ] App roles: roles for users (e.g. `Api.Admin`) and for applications (e.g. `Orders.Read.All`).
- [ ] Access token version 2 (`requestedAccessTokenVersion: 2` in the manifest).
- [ ] Enterprise application "Assignment required" where only assigned users/apps may obtain tokens.
- [ ] Optional access-token claim `idtyp` if you want an explicit app-only marker.
- [ ] No group claims for authorization (group overage above 200 groups); use app roles.
- [ ] Pre-authorized/known client applications for the web and app client registrations.
- [ ] Swagger UI: a separate SPA registration with redirect `https://<api-host>/swagger/oauth2-redirect.html`, only in non-production environments, with only the scopes needed for testing.
