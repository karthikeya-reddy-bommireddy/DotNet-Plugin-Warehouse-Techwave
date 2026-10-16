# Token pitfalls and claims cheat sheet

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Claims you will use
| Claim | Meaning | Use it for |
|---|---|---|
| `aud` | Intended audience | Validated by the middleware; must be this API |
| `iss` | Issuer (tenant-specific) | Validated by the middleware |
| `tid` | Tenant ID | Tenant isolation |
| `oid` | Object ID of the user or service principal | Stable caller identity (with `tid`) |
| `scp` | Delegated scopes, space-separated | Scope policies; absent in app-only tokens |
| `roles` | App roles of a user, or application permissions of an app | Role and app-permission policies |
| `azp` (v2) / `appid` (v1) | Client application ID | Auditing; optional client allowlist |
| `idtyp` | `app` for app-only tokens (optional claim) | Detecting daemon callers |
| `name`, `preferred_username`, `email`, `upn` | Display data | Display only, never keys or authorization |

Read `oid`/`tid` with `User.GetObjectId()` / `User.GetTenantId()`: inbound claim mapping can rename them to long URI claim types, and these helpers handle both forms.

## Pitfalls
| Pitfall | Why it hurts | Do instead |
|---|---|---|
| Only `[Authorize]` on endpoints | Any app able to get a token for the API passes | Require a scope or app permission per endpoint |
| `TenantId: "common"` in a single-tenant API | Accepts tokens from other tenants | Pin the tenant GUID |
| Multi-tenant without a tenant allowlist | Any Entra tenant can call | Policy on `tid` against a configured allowlist |
| Turning off audience/issuer validation to "fix" a 401 | Accepts tokens minted for other APIs | Fix the client's requested scope or the `Audience` setting |
| Keying users by email/UPN | Mutable; collisions across tenants | `oid` + `tid` |
| Treating app-only tokens like user tokens | Daemons act "as" users | Separate policies; reject app-only on user endpoints |
| Larger `ClockSkew` for flaky tests | Expired tokens accepted | Fix clocks or the test |
| Logging tokens or the `Authorization` header | Token theft from logs | Log `oid`, never the token |
| `ShowPII = true` outside Development | PII and token fragments in logs | Development only |
| Pasting production tokens into third-party decoders | Token leakage | Decode locally; never real users' production tokens |
| Client secret in appsettings | Credential leak | Certificate in Key Vault or managed identity |

## Debugging a 401 (non-production)
1. Read the `WWW-Authenticate` response header (`error`, `error_description`).
2. Decode the token locally and compare `aud`, `iss`/`tid`, `exp`, `scp`/`roles` with configuration.
3. Temporarily pass `subscribeToJwtBearerMiddlewareDiagnosticsEvents: true` to `AddMicrosoftIdentityWebApi` in Development only.
4. Fix the client or the configuration. Never loosen validation.

## Testing
Integration tests replace Entra with a test authentication handler (api-security-testing). Keep one test asserting the JWT bearer options still validate issuer, audience and lifetime and require signed tokens and HTTPS metadata.
