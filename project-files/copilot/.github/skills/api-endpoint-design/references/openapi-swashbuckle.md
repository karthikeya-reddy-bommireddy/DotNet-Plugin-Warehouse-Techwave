# Swagger / OpenAPI on .NET 10

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

The spec names Swashbuckle. On .NET 10 use **Swashbuckle.AspNetCore v10 or later**. v10 supports ASP.NET Core 10 and OpenAPI 3.1 (opt-in) and moved to Microsoft.OpenApi 2.x, which brings breaking changes: types live in the `Microsoft.OpenApi` namespace, filters use interface types, and `AddSecurityRequirement` takes a function. Since .NET 9 the Web API template no longer includes Swashbuckle, and .NET 10's built-in `Microsoft.AspNetCore.OpenApi` produces OpenAPI 3.1. Switching is a team decision; this file covers Swashbuckle.

## Security rules
1. Serve the document and UI only outside Production (config flag `Swagger:Enabled`, false by default). `UseSwagger`/`UseSwaggerUI` are middleware, not endpoints, so the fallback authorization policy does **not** protect them.
2. Swagger UI signs in with **OAuth2 authorization code + PKCE** through a separate public (SPA) app registration whose redirect URI is the UI's `oauth2-redirect.html`. No client secret anywhere.
3. Do not call `EnablePersistAuthorization()`; it keeps tokens in browser storage.
4. `[ApiExplorerSettings(IgnoreApi = true)]` hides an endpoint from docs; it does not secure it.
5. If API Management or clients need the document from Production, generate it at build time in the pipeline and publish it as an artifact instead of serving it.
6. The API's strict `Content-Security-Policy` header must skip `/swagger` (api-pipeline-hardening), or the UI cannot load.

## Setup (Swashbuckle v10)
```csharp
using Microsoft.OpenApi;

var swaggerEnabled = builder.Configuration.GetValue<bool>("Swagger:Enabled")
                     && !builder.Environment.IsProduction();

if (swaggerEnabled)
{
    var instance = builder.Configuration["AzureAd:Instance"]!.TrimEnd('/');
    var tenantId = builder.Configuration["AzureAd:TenantId"]!;
    var scope = builder.Configuration["Swagger:Scope"]!;   // e.g. api://<api-client-id>/Orders.Read

    builder.Services.AddSwaggerGen(o =>
    {
        o.AddSecurityDefinition("oauth2", new OpenApiSecurityScheme
        {
            Type = SecuritySchemeType.OAuth2,
            Flows = new OpenApiOAuthFlows
            {
                AuthorizationCode = new OpenApiOAuthFlow
                {
                    AuthorizationUrl = new Uri($"{instance}/{tenantId}/oauth2/v2.0/authorize"),
                    TokenUrl = new Uri($"{instance}/{tenantId}/oauth2/v2.0/token"),
                    Scopes = new Dictionary<string, string> { [scope] = "Call the API as the signed-in user" }
                }
            }
        });

        // v10: AddSecurityRequirement takes Func<OpenApiDocument, OpenApiSecurityRequirement>
        o.AddSecurityRequirement(document => new OpenApiSecurityRequirement
        {
            [new OpenApiSecuritySchemeReference("oauth2", document)] = [scope]
        });
    });
}

var app = builder.Build();

if (swaggerEnabled)
{
    app.UseSwagger();
    app.UseSwaggerUI(o =>
    {
        o.OAuthClientId(app.Configuration["Swagger:ClientId"]!);   // the public SPA client, not the API's
        o.OAuthUsePkce();
        o.OAuthScopes(app.Configuration["Swagger:Scope"]!);
    });
}
```
Check the Swashbuckle v10 migration guide for exact member names in the installed version before writing filters.

## Built-in alternative (team decision)
```csharp
builder.Services.AddOpenApi();          // Microsoft.AspNetCore.OpenApi, OpenAPI 3.1 on .NET 10
if (swaggerEnabled) app.MapOpenApi();   // an endpoint: the fallback policy applies unless you allow anonymous
```
A UI is a separate package. Document transformers replace Swashbuckle filters.
