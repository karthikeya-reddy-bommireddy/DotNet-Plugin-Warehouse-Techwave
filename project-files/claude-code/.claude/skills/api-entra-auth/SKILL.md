---
name: api-entra-auth
description: Microsoft Entra ID (Azure AD) authentication for the ASP.NET Core .NET 10 Web API with Microsoft.Identity.Web and JWT bearer tokens - configuration, token validation rules, delegated scopes vs application permissions (app roles), user vs app-only tokens, tenant pinning, which claims to trust (oid, tid), Swagger OAuth, and the misconfigurations behind 401 errors. Use whenever touching AddMicrosoftIdentityWebApi, JwtBearer options, the AzureAd config section, app registration settings, scopes or roles, or when debugging 401 responses, even if the request only says "login", "token" or "auth".
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: api-entra-auth/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# Entra ID authentication (Microsoft.Identity.Web)

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Rules (most important first)
1. **Authenticate only through Microsoft.Identity.Web**: `AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"))`. Never parse or validate JWTs by hand, and never authorize from an unvalidated token, an id_token or a custom header.
2. **Never weaken validation**: issuer, audience, lifetime and signature validation stay on; `RequireHttpsMetadata` stays true; no `SignatureValidator`, no permissive `IssuerValidator`/`AudienceValidator`, no large `ClockSkew`.
3. **Pin the tenant**: a single-tenant API sets `TenantId` to the tenant GUID (never `common` or `organizations`). A multi-tenant API allowlists tenants (`tid`) in a policy.
4. **A valid token is authentication, not authorization.** Every endpoint must also require a delegated scope (`scp`) and/or an application permission (app role in `roles`); see api-authorization. Consider "Assignment required" on the API's enterprise application so only assigned users and apps get tokens.
5. **Identify callers by `oid` + `tid`** via `User.GetObjectId()` / `User.GetTenantId()`. Never key data on `email`, `upn`, `preferred_username` or `name`.
6. **User tokens vs app-only tokens**: app-only tokens carry `roles` but no `scp`. Endpoints acting for a user reject app-only tokens; daemon endpoints require an app permission.
7. **No secrets**: a pure API needs no client secret. If it calls downstream APIs on behalf of users, use a certificate from Key Vault or a managed-identity federated credential.
8. **`IdentityModelEventSource.ShowPII` / `LogCompleteSecurityArtifact` only in Development**, never committed as enabled for other environments.

## Configuration (no secrets)
```json
"AzureAd": {
  "Instance": "https://login.microsoftonline.com/",
  "TenantId": "<tenant-guid>",
  "ClientId": "<api-app-registration-client-id>",
  "Scopes": "<delegated scopes, space-separated>",
  "AppPermissions": "<application permissions, space-separated>"
}
```
`Scopes` and `AppPermissions` also feed the strict fallback policy in api-authorization. Microsoft.Identity.Web accepts the client ID and `api://<client-id>` as audiences; add `Audience` only for a custom App ID URI.

## Status codes
- Missing, expired or invalid token → 401 with `WWW-Authenticate: Bearer ...` (from the middleware).
- Valid token without the required scope, app permission or role → 403 (from policies).

## References
- [entra-setup.md](references/entra-setup.md): Program.cs wiring, rejected-token logging hook, app registration checklist.
- [token-pitfalls.md](references/token-pitfalls.md): claims cheat sheet, pitfalls table, 401 debugging steps, how tests replace Entra.
