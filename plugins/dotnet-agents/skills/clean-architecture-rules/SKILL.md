---
name: clean-architecture-rules
description: Layer placement and dependency rules for the .NET 10 Clean Architecture APIs repo (Domain, Application, Infrastructure, API) - what code belongs in which layer, allowed project references, the order to build a feature across layers, DI registration and service lifetimes, and the security duties of each layer. Use whenever deciding where new code goes, adding a project reference, registering services, or planning any feature that touches more than one layer, even if the request only names an endpoint.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: clean-architecture-rules/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# Clean Architecture rules

> **Placeholders:** names in examples (such as `Order`, `IOrderRepository`, `AddInfrastructure`) are illustrations. Follow the repo's real names and conventions; if a convention is missing, ask the user.

## Rules (most important first)
1. **The repo's existing structure wins.** Read the solution's projects and folders first and follow them. If the repo differs from this skill, follow the repo and mention the difference in your plan.
2. **Dependencies point inward**: API → Application → Domain, and Infrastructure → Application → Domain. Domain references nothing outside the .NET base library.
3. **The API project references Infrastructure only to register services** in the composition root (Program.cs or its extension methods). Controllers never use Infrastructure types.
4. **Ask before structural changes**: new projects, new project references and new packages are team decisions.
5. **Each layer has a security duty**; no layer relies only on another to be safe (table below).

## What goes where
| Layer | Holds | May reference | Must not contain |
|---|---|---|---|
| Domain | Entities, value objects, enums, domain exceptions, domain services, core business rules | Nothing (base library only) | EF Core types or attributes, DTOs, HTTP, logging, AutoMapper |
| Application | Use case interfaces and classes, request/response DTOs, mapping profiles, validation, ports (`IRepository<T>`, `IUnitOfWork`, `ICurrentUser`, external service interfaces) | Domain | `DbContext`, SQL, `HttpContext`, Azure SDKs, controller types |
| Infrastructure | `DbContext`, entity configurations, migrations, repository and unit-of-work implementations, external integrations (typed HttpClients), identity-provider integrations | Application, Domain | Controllers, HTTP pipeline code, business rules |
| API | Controllers, filters, middleware, Program.cs, authentication/authorization setup, OpenAPI, the `ICurrentUser` adapter | Application; Infrastructure only for DI registration | Business rules, `DbContext` in controllers, entity binding |

## Security duty per layer
| Layer | Duty |
|---|---|
| API | Authenticate, apply function policies, validate request shape, shape responses, limit rate and size |
| Application | Object-level checks through `ICurrentUser`; set owner/tenant/audit fields; orchestrate one transaction per use case |
| Domain | Enforce invariants so no caller can create an invalid state |
| Infrastructure | Scope every query for owned data, parameterize SQL, protect connection details, safe migrations |

## Building a feature across layers (plan in this order)
1. **Domain**: entity or value object changes and the rules they enforce (domain-model).
2. **Application**: port interface, use case, request/response DTOs, mapping profile, validation (application-use-cases, api-input-validation).
3. **Infrastructure**: repository method, entity configuration, migration (efcore-data-access).
4. **API**: controller action, policy, OpenAPI metadata (api-endpoint-design, api-authorization).
5. **Tests** at each level (api-security-testing).

## Dependency injection
- Each layer exposes one registration method, following the repo's naming (for example `AddApplication()` and `AddInfrastructure(configuration)`). Program.cs calls them.
- Lifetimes: `DbContext`, repositories, unit of work, use cases and `ICurrentUser` are scoped. HttpClients come from `IHttpClientFactory`. `TimeProvider.System` can be a singleton.
- No captive dependencies: a singleton must never depend on a scoped service; it would hold one request's user or `DbContext` for every request.
- Never resolve services manually inside business code (service locator); use constructor injection.

## Optional guard (team decision)
Architecture tests can fail the build when a layer references the wrong project (for example with an architecture-testing package). Suggest it; do not add the package without approval.
