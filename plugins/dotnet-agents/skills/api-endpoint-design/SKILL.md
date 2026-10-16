---
name: api-endpoint-design
description: Design and implement REST endpoints in the API layer of the .NET 10 Clean Architecture APIs repo - thin controllers that delegate to Application use cases, HTTP status codes, RFC 9457 ProblemDetails, bounded paging and sorting, API versioning, idempotency and ETag concurrency, and Swagger/OpenAPI with Swashbuckle v10. Use whenever adding, changing or reviewing a controller, action, route, response type, paging parameter or OpenAPI setup in this repo, even for a small endpoint tweak.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: api-endpoint-design/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# API endpoint design

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Rules (most important first)
1. **Thin controllers.** Bind → call one Application use case → return a response DTO. No business rules, no `DbContext` or repositories, no entity mapping in controllers. The API layer depends only on Application abstractions.
2. **Authorization is declared** on every action (class- or action-level policy) and objects are checked; see api-authorization.
3. **Async all the way**, with `CancellationToken ct` passed to every call.
4. **Response DTOs only.** Return `ActionResult<T>`; never entities, `IQueryable` or anonymous objects built from entities.
5. **Document the status codes the action can actually produce** with `[ProducesResponseType]`, so OpenAPI matches behaviour: 401 and 403 on protected endpoints, 429 where a rate limiter applies, and 400, 404, 409 or 412 only when the action can return them. Never add a code just to complete a checklist.
6. **Lists are paged** with a server-enforced maximum page size; sort and filter accept allowlisted values only.
7. **Errors are ProblemDetails.** Use `NotFound()`, `Conflict()`, `ValidationProblem()`, `Problem(...)`; `[ApiController]` turns them into RFC 9457 responses. Never hand-build error JSON.
8. **Explicit binding sources**: `[FromRoute]`, `[FromQuery]`, `[FromBody]`. Never accept write data from the query string.

## Status codes
| Situation | Code |
|---|---|
| Found | 200 |
| Not found, or exists but the caller may not access it | 404, identical in both cases so IDs cannot be probed |
| Created | 201 with `Location` via `CreatedAtAction` |
| Accepted for background processing | 202 with a status URL authorized like the original call |
| Updated or deleted, no body | 204 |
| Validation failed | 400 (automatic from `[ApiController]`) |
| Not authenticated | 401 (from the authentication middleware, never returned by hand) |
| Authenticated but missing the scope, app permission or role | 403 (from policies) |
| Business conflict or duplicate | 409 |
| ETag mismatch on `If-Match` / required `If-Match` missing | 412 / 428 |
| Body too large | 413 |
| Rate limited | 429 with `Retry-After` |
| Unexpected failure | 500 generic ProblemDetails |

## Route conventions
- Nouns and constrained IDs: `api/v{version:apiVersion}/orders/{id:guid}`. Child resources nest under their parent, and the parent is authorized too.
- Prefer non-sequential public IDs (GUIDs). They slow enumeration; they never replace authorization.
- Verbs only for real commands: `POST orders/{id:guid}/cancel`.

## Minimal APIs
If the repo uses Minimal APIs, the same rules apply: `.RequireAuthorization(policy)` on every group or endpoint, `TypedResults`, `.Produces(...)` metadata, and `builder.Services.AddValidation()` (.NET 10) so Data Annotations run.

## References
- [controller-template.md](references/controller-template.md): golden controller (owner-scoped read, paged list, rate-limited create) with the reasoning.
- [http-patterns.md](references/http-patterns.md): paging and sorting, versioning and deprecation, idempotency keys, ETag concurrency, long-running operations, PATCH.
- [openapi-swashbuckle.md](references/openapi-swashbuckle.md): Swashbuckle v10 on .NET 10 with Entra OAuth2 + PKCE outside production, and the built-in OpenAPI alternative.

## Done checklist
- [ ] Route constraints on IDs; parent resources authorized
- [ ] Policy declared; every resource ID covered by an owner/tenant check
- [ ] Request DTO validated (api-input-validation); explicit binding sources
- [ ] Response DTO reviewed field by field for exposure
- [ ] `[ProducesResponseType]` for every status code the action can actually return, and no others
- [ ] Rate-limit policy on writes, search, export and expensive operations
- [ ] Tests from api-security-testing
