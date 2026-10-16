---
name: api-security-testing
description: xUnit test strategy for the ASP.NET Core .NET 10 APIs repo with a security focus - WebApplicationFactory integration tests, a test authentication handler replacing Entra ID, test identities for users, tenants and daemon apps, authorization-matrix-driven tests (401/403/404 BOLA), validation and mass-assignment tests, endpoint inventory test, error-leakage, CORS, header and rate-limit tests, JWT options and AutoMapper configuration tests, and SQL Server via Testcontainers instead of EF Core InMemory. Use for any test work in this repo, as part of every endpoint change, and whenever tests fail with 401 or 403.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: api-security-testing/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# Security-focused API testing

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Rules (most important first)
1. **Tests are part of done.** Every new or changed endpoint ships with: success, 401 (anonymous), 403 (missing permission), 404 for another owner (BOLA), and invalid input (400). Add the other-tenant case only for tenant-scoped resources in a confirmed multi-tenant project; otherwise record it as N/A with a reason. Endpoints without a resource ID (health, metadata) record the BOLA cases as N/A.
2. **Never disable authentication or authorization to make tests pass.** Replace Entra ID with the test authentication handler; production code stays unchanged.
3. **The authorization matrix drives the tests** (api-authorization/references/authorization-matrix.md); a new endpoint without matrix rows fails review.
4. **Assert that nothing leaks**: response JSON property allowlists, ProblemDetails without stack traces or exception text, no `Access-Control-Allow-Origin` for unknown origins.
5. **Real SQL Server semantics** (Testcontainers or a dedicated test database) for anything touching data rules, tenant filters, constraints or raw SQL. EF Core InMemory is not relational and can hide bugs; use it only where database behaviour does not matter.
6. **Safety nets run in CI**: endpoint inventory test, JWT options test, AutoMapper configuration test, forbidden-pattern scan and layer dependency tests (api-guardrails), and staging-slot smoke tests after deployment.
7. **Fictional data only**: no real tenant IDs, tokens or personal data in fixtures.
8. **Report test status honestly**: PASS only when tests ran and passed; otherwise FAIL, SKIPPED or NOT RUN with the reason (for example "Docker not available: full-stack tests NOT RUN").

## Test identities
| Key | Claims |
|---|---|
| `owner` | oid A1, tid A, `scp` Orders.Read Orders.Write |
| `reader` | oid A2, tid A, `scp` Orders.Read |
| `other-user` | oid A3, tid A, `scp` Orders.Read Orders.Write |
| `other-tenant` | oid B1, tid B, `scp` Orders.Read Orders.Write |
| `no-perm` | oid A4, tid A, no `scp`, no `roles` |
| `daemon` | oid D1, tid A, `roles` Orders.Read.All, no `scp` |
| `admin` | oid A5, tid A, `scp` Orders.Read Orders.Write, `roles` Api.Admin |

## Test levels
- **Unit**: Domain invariants and Application use cases, with fakes for repositories and `ICurrentUser`; no database, no HTTP.
- **API slice**: real pipeline and controllers, Application use cases replaced by fakes. Fast; proves routing, policies, validation, error handling and headers.
- **Full stack**: real Application and Infrastructure wired by the composition root, SQL Server in a container. Proves BOLA and tenant isolation end to end; required for every query that returns or changes owned data.

## References
- [test-infrastructure.md](references/test-infrastructure.md): `ApiFactory`, `TestAuthHandler`, test identities, Testcontainers fixture, configuration gotchas.
- [security-test-catalog.md](references/security-test-catalog.md): matrix theory, mass assignment, validation, response allowlist, endpoint inventory, error leakage, CORS/headers, rate limits, JWT options, staging smoke tests.
