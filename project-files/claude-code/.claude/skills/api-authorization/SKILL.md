---
name: api-authorization
description: Authorization for the ASP.NET Core .NET 10 APIs repo - Entra ID scopes, app permissions and app roles turned into named policies, strict fallback and default policies, role- and claims-based rules, object-level (BOLA) checks through owner/tenant-scoped use cases or resource-based authorization handlers, the ICurrentUser adapter, admin endpoint separation and 401/403/404 semantics. Use whenever an endpoint is added or changed, when touching [Authorize], [AllowAnonymous], policies, roles, claims, IAuthorizationService or tenant/ownership logic, or when deciding what a caller may see or do.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: api-authorization/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# API authorization

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Rules (most important first)
1. **Four layers, all required**: fallback policy (no permission, no access) → function policy per action → object check per resource ID → field shaping per audience (different response DTOs).
2. **Named policies from one catalog** (`Policies` constants + one `AddApiAuthorization` extension). No role strings scattered across attributes, no `User.IsInRole` checks inside actions.
3. **Fallback and default policies are strict**: authenticated *and* holding one of this API's scopes or app permissions. A bare `[Authorize]` must never be weaker than the fallback.
4. **`[AllowAnonymous]` overrides every `[Authorize]`** on that endpoint, including action-level ones. Use it only on specific actions (health, metadata), each justified and in the anonymous allowlist test.
5. **Object checks for every resource ID.** Prefer filtering inside the Application use case by `ICurrentUser.ObjectId` and `TenantId`, so no data is loaded for others. Use resource-based authorization when the rule needs the loaded resource.
6. **404 when the caller may not access an object that exists**, 403 when the caller lacks the function permission, 401 only from the authentication middleware.
7. **Delegated vs app-only**: user endpoints require scopes (`RequireScope`); daemon endpoints require app permissions; endpoints serving both use `RequireScopeOrAppPermission`. App-only tokens never act on a user's resources unless the design says so.
8. **Admin endpoints live in their own controllers** with an admin policy at class level.
9. **Fail closed**: handlers never call `Succeed` in a `catch`; an exception means deny.

## Policy catalog shape
```csharp
public static class Policies
{
    public const string OrdersRead  = "orders:read";
    public const string OrdersWrite = "orders:write";
    public const string Admin       = "admin";
}
```
Each policy = authenticated user + scope and/or app permission (+ role when needed). Full registration, the strict fallback/default policies and the resource handler: [policies-and-handlers.md](references/policies-and-handlers.md).

## Choosing the object-level pattern
| Situation | Pattern |
|---|---|
| Read or list where ownership/tenant is a filter | Use case filters by `ICurrentUser`; null/empty → 404/empty list |
| Update or delete with simple ownership | Same filter inside the command; "not found" when not owned |
| Rule depends on state or relationships (approver, shared-with, status) | Load an ownership view, then `IAuthorizationService.AuthorizeAsync(User, resource, Operations.X)` |
| Child resource (`/orders/{orderId}/lines/{lineId}`) | Parent accessible and child belongs to parent |
| IDs inside request bodies (product, assignee, account) | Each one checked against the caller's tenant |

`ICurrentUser` is an Application-layer port; the API layer implements it from claims. Use cases filter by it (application-use-cases) and repositories apply it in every query for owned data (efcore-data-access).

## Authorization matrix
Keep the authorization matrix current (its location is a project fact: read project-facts or ask): every endpoint × caller type → expected status. Tests are generated from it. Template: [authorization-matrix.md](references/authorization-matrix.md).

## Pitfalls to catch
- Several `[Authorize]` attributes are AND-ed; `Roles = "A,B"` inside one attribute is OR.
- Group-claim overage (over 200 groups the token carries a link instead of groups): use app roles.
- Hiding a button in the web or app client is not authorization.
- `IgnoreQueryFilters()` in the data layer can bypass tenant filters; a finding unless justified.
- Background jobs and message handlers have no HTTP user: they carry the owner/tenant explicitly and apply the same checks (application-use-cases).
