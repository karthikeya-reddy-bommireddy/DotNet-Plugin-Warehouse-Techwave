# OWASP API Security Top 10 (2023) for this .NET 10 API

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Trust boundaries
| Source | Trust | Handle with |
|---|---|---|
| Route, query, headers, body, files | Untrusted | Validation, size limits, authorization of every ID |
| Token claims | Trusted only after Microsoft.Identity.Web validation | api-entra-auth |
| Web, app and daemon clients | Untrusted; can be bypassed with curl | Server-side checks only |
| Downstream API and database results | Untrusted input (API10) | Validation, size caps, timeouts |
| Configuration | Trusted; secrets only from a secret store | api-pipeline-hardening |

## Quick map: risk → control → skill
| Risk | Control in this repo | Skill |
|---|---|---|
| API1 Broken Object Level Authorization | Owner/tenant-scoped queries or resource-based authorization; 404 when not accessible | api-authorization, application-use-cases, efcore-data-access |
| API2 Broken Authentication | Microsoft.Identity.Web defaults; pinned tenant; no custom token parsing | api-entra-auth |
| API3 Broken Object Property Level Authorization | Request/response DTOs, no `ReverseMap` onto entities, unknown JSON members handled | api-input-validation |
| API4 Unrestricted Resource Consumption | Rate limiter, size limits, paging caps, timeouts, regex timeouts | api-pipeline-hardening |
| API5 Broken Function Level Authorization | Policy per action; admin endpoints separated; strict fallback and default policies | api-authorization |
| API6 Unrestricted Access to Sensitive Business Flows | Per-user/per-flow limits, idempotency keys, quotas | api-endpoint-design |
| API7 Server-Side Request Forgery | No user-supplied URLs; allowlisted typed HttpClients | api-pipeline-hardening |
| API8 Security Misconfiguration | HTTPS/HSTS, CORS allowlist, headers, Swagger off in production, ProblemDetails | api-pipeline-hardening |
| API9 Improper Inventory Management | Versioning, endpoint inventory test, OpenAPI matches reality | api-endpoint-design, api-security-testing |
| API10 Unsafe Consumption of APIs | Validate and bound downstream responses; never disable TLS validation | api-pipeline-hardening |

Also apply OWASP Top 10:2025 A03 Software Supply Chain Failures (rule 10) and A10 Mishandling of Exceptional Conditions (rule 6: fail closed). Vulnerable patterns, fixes and tests per risk: the sections below.

## Per risk
Each entry: how it shows up in ASP.NET Core, the fix, and the test that proves the fix. The 2023 edition is the current API list; for web-wide risks also see OWASP Top 10:2025 (bottom).

## API1 Broken Object Level Authorization (BOLA)
Looks like:
```csharp
[HttpGet("{id:guid}")]
public async Task<ActionResult<InvoiceResponse>> Get(Guid id, CancellationToken ct)
    => Ok(await invoices.GetByIdAsync(id, ct)); // any authenticated caller can read any invoice
```
Fix: the Application query filters by the caller (`ICurrentUser.ObjectId` / `TenantId`) and returns null when there is no access; the controller returns 404. When the rule needs more than a filter, use resource-based authorization (api-authorization). Apply the same check to IDs inside bodies (e.g. `lines[].productId` from another tenant) and to child routes (`/orders/{orderId}/lines/{lineId}`: the line belongs to the order, the order to the caller).
Test: user B requests user A's invoice → 404; tenant 2 requests a tenant 1 resource → 404; the same IDs on PUT/DELETE → 404 and nothing persisted.

## API2 Broken Authentication
Looks like: `ValidateAudience = false`, `ValidateIssuer = false`, a custom `IssuerValidator` that accepts any issuer, `TenantId: "common"` in a single-tenant API, manual `ReadJwtToken(...)` used for decisions, identity read from an `X-User-Id` header, `ShowPII = true` in production.
Fix: Microsoft.Identity.Web defaults (api-entra-auth); pinned tenant or tenant allowlist; never decode tokens by hand for authorization.
Test: JWT options test (api-security-testing); a request carrying only `X-User-Id` gets 401.

## API3 Broken Object Property Level Authorization
Looks like:
```csharp
[HttpPut("{id:guid}")]
public async Task<IActionResult> Update(Guid id, [FromBody] User user) // client can set Role, IsAdmin, TenantId
{
    db.Users.Update(user);
    await db.SaveChangesAsync();
    return NoContent();
}
```
Also: returning entities with `PasswordHash`, `InternalNotes`, `TenantId`; `CreateMap<User, UserDto>().ReverseMap()` letting a DTO write back into an entity.
Fix: request DTOs hold only client-settable fields; owner, tenant, role, status and audit fields are set server-side; response DTOs per audience; no `ReverseMap` onto entities; consider rejecting unknown JSON members.
Test: send `"role":"Admin","tenantId":"<other>"` → 400 or ignored, stored values unchanged; assert response JSON property names equal an allowlist.

## API4 Unrestricted Resource Consumption
Looks like: `pageSize` without maximum; `Include` chains loading whole graphs; search/export without rate limits; ~30 MB default body accepted everywhere; regex without timeout; `.Result` / synchronous I/O; no `CancellationToken`.
Fix: `[Range]` on paging; low global body limit with per-endpoint `[RequestSizeLimit]`; rate-limit policies; `MatchTimeoutInMilliseconds`; async with cancellation; request timeouts.
Test: `pageSize=100000` → 400; burst over the strict policy → 429 + `Retry-After`; oversized body → 413.

## API5 Broken Function Level Authorization
Looks like: admin actions in a user controller protected only by `[Authorize]`; `[AllowAnonymous]` on a base class or controller silently overriding action-level `[Authorize]`; one policy for GET and DELETE; "hidden" endpoints the UI never calls.
Fix: one named policy per capability; admin endpoints in their own controller with an admin policy at class level; no `[AllowAnonymous]` above action level; never rely on the UI hiding things.
Test: normal user calls each admin endpoint → 403; the endpoint inventory test fails for any endpoint without a policy.

## API6 Unrestricted Access to Sensitive Business Flows
Looks like: unlimited coupon redemption, sign-ups, bookings, OTP/verification attempts or bulk purchases by one caller or script.
Fix: identify sensitive flows with the business owner; per-user and per-flow rate limits; idempotency keys on create/pay operations; quotas; anomaly logging.
Test: repeat the flow past the limit → 429 or 409; replaying an idempotency key returns the original result without a second effect.

## API7 Server-Side Request Forgery
Looks like: `await http.GetAsync(request.CallbackUrl)`; "import from URL"; client-chosen webhook targets.
Fix: avoid user-supplied URLs. If unavoidable: https only, host allowlist, reject private/loopback/link-local/metadata addresses (169.254.169.254) after DNS resolution, no automatic redirects, short timeout, response size cap (api-pipeline-hardening).
Test: `http://169.254.169.254/`, `http://localhost`, `http://10.0.0.5`, `file:///` → 400 and no outbound call.

## API8 Security Misconfiguration
Looks like: Swagger in production; Developer Exception Page outside Development; `AllowAnyOrigin`; no HSTS; exception text in responses; `EnableSensitiveDataLogging` in production; `AllowInputFormatterExceptionMessages` left on; default limits; outdated packages.
Fix: api-pipeline-hardening golden Program.cs.
Test: Swagger disabled → 404; forced exception → generic ProblemDetails; unknown Origin gets no `Access-Control-Allow-Origin`; staging-slot smoke tests.

## API9 Improper Inventory Management
Looks like: an old API version still deployed with weaker checks; debug or test controllers in release builds; endpoints missing from OpenAPI.
Fix: explicit versioning with deprecation dates; no debug endpoints in release; endpoint inventory test; OpenAPI generated from code.
Test: endpoint inventory test, compared with the authorization matrix.

## API10 Unsafe Consumption of APIs
Looks like: trusting deserialized downstream responses; following redirects to other hosts; `ServerCertificateCustomValidationCallback = (_, _, _, _) => true`; unbounded responses; no timeouts.
Fix: typed HttpClients with fixed base addresses, timeouts, resilience, response validation and size caps; never disable certificate validation.
Test: adapter unit tests with malformed, oversized and slow responses.

## OWASP Top 10:2025 additions worth applying here
- **A03 Software Supply Chain Failures**: central package management; NuGet audit on restore covering transitive packages (`NuGetAuditMode` = `all`) with audit warnings failing CI; package source mapping in `nuget.config` to approved feeds; lock files (`RestorePackagesWithLockFile`) in CI; review new packages and licences.
- **A10 Mishandling of Exceptional Conditions**: authorization and validation fail closed; no `catch { return Ok(); }`; cancellations and timeouts handled; one global exception handler producing generic ProblemDetails.
