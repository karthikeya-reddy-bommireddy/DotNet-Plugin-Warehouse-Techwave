---
name: api-security-baseline
description: Mandatory security baseline and pre-completion Security Gate for the ASP.NET Core .NET 10 APIs repo (Clean Architecture, Entra ID, EF Core, SQL Server). Holds the non-negotiable rules, trust boundaries, the five-question threat check, an OWASP API Security Top 10 (2023) map to .NET controls, a severity rubric and the cross-repo change request template. Use before declaring any backend change done (API, Application, Domain or Infrastructure), and whenever asked to review, audit, threat-model or security-check backend code, a PR, a branch or a diff, even when the request does not say "security".
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: api-security-baseline/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# API security baseline

Rules are ordered by importance; the most critical come first.

## Non-negotiable rules
1. **Deny by default.** A fallback authorization policy rejects callers without this API's scopes or app permissions. Every endpoint names a policy, or carries `[AllowAnonymous]` with a justification and an entry in the anonymous allowlist test.
2. **Authenticate, then authorize the function, then authorize the object.** Policies alone do not stop BOLA. Any endpoint that takes a resource ID must prove the caller may touch that resource (owner, tenant or ACL).
3. **Identity only from validated claims**: `oid`/`tid` for who, `scp` for delegated permissions, `roles` for app roles and app permissions.
4. **DTOs at the edge.** Requests bind to validated request DTOs; responses are response DTOs listing only fields the caller may see. Entities never cross the HTTP boundary.
5. **Everything is bounded**: page size, body size, collection size, string length, request rate, request time.
6. **Errors fail closed and say little**: RFC 9457 ProblemDetails with a traceId; no stack traces, exception messages, SQL, type names or internal IDs. Details go to logs.
7. **No secrets in the repo**: no keys, connection strings, tokens or certificates in code, `appsettings*.json`, tests, pipelines or OpenAPI. Use Key Vault or App Service settings with Managed Identity.
8. **Logs never contain** tokens, `Authorization` headers, passwords, secrets, full request/response bodies, or personal data beyond the caller's object ID.
9. **Never weaken a control** to make something work. Escalate instead.
10. **Supply chain**: no new package without team approval, no package with a known vulnerability, versions pinned (central package management), restore only from approved feeds.
11. **No invented values**: names and numbers in skill examples (scopes, roles, routes, limits, origins, IDs, business rules) are placeholders. Use confirmed values from `.agents/api-layer/project-facts.md`, or ask the user. Never guess, including the tenancy model.

## Threat check: five questions for every new or changed endpoint
1. **Who** may call it (anonymous, user, daemon app), with which scope, app permission or role?
2. **Which objects** does it read or change, and how are owner and tenant enforced?
3. **What input** does it accept, and what are the size and shape limits?
4. **What output** does it return, and could any field leak data the caller should not see?
5. **What abuse at volume** is possible (enumeration, scraping, cost amplification, business-flow abuse), and what limits it?

Put one line per answer in your report.

## Security Gate (run before declaring any change done)
Mark each PASS / FAIL / N/A with file:line evidence. A FAIL blocks completion: fix it, or report it as a blocking finding.
- **G1** Every new or changed endpoint has an explicit policy or a justified, allowlisted `[AllowAnonymous]`.
- **G2** Every resource ID from route, query or body is covered by an owner/tenant/ACL check.
- **G3** Caller identity comes only from validated claims.
- **G4** Request DTOs validate lengths, ranges, formats, enums and collection sizes; no entity binding; no client-settable owner, tenant, role or status fields.
- **G5** Response DTOs expose only intended fields; no entity, `IQueryable` or internal detail leaks.
- **G6** Paging is capped; body limits fit the endpoint; abusable endpoints have a rate-limit policy.
- **G7** Errors are ProblemDetails with no internal detail; 401, 403 and 404 are used consistently.
- **G8** No secrets or personal data in code, configuration, logs, tests or OpenAPI.
- **G9** Security-relevant events are logged without sensitive data.
- **G10** Tests cover success, 401, 403, cross-owner (BOLA) and invalid input; cross-tenant only for tenant-scoped resources in a confirmed multi-tenant project (otherwise N/A with a reason).
- **G11** `dotnet build` adds no warnings (analyzers on) and `dotnet test` is green, and both actually ran. A check that did not run is NOT RUN, never PASS.
- **G12** If packages changed: no known vulnerabilities, licence acceptable, team approved.
- **G13** OpenAPI shows auth requirements and error responses; no internal endpoint is exposed.
- **G14** Data access: owned-data queries filter by owner (and tenant where the project is multi-tenant); raw SQL is parameterized; migrations are reviewed for data loss and contain no secrets or real personal data.
- **G15** Guardrails (team add-on; N/A unless adopted): no guardrail file (security tests, forbidden-pattern scan, allowlist, build security settings, pipeline security stage, CODEOWNERS) created or changed unless approved as a guardrail change.

## References (open when the task needs them)
- [review-procedure.md](references/review-procedure.md): every review or audit request; includes the severity rubric and the finding template.
- [owasp-api-top10-dotnet.md](references/owasp-api-top10-dotnet.md): trust boundaries, the OWASP API Top 10 map to this repo's controls, and the vulnerable pattern, fix and test per risk. Open it when threat-checking a new kind of endpoint or mapping a finding to a risk.
- [change-request.md](references/change-request.md): the cross-repo change request template, whenever a fix belongs outside this repo.
