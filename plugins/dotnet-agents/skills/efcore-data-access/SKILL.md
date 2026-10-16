---
name: efcore-data-access
description: Infrastructure data access for the .NET 10 APIs repo with EF Core and SQL Server - DbContext and Fluent API configuration, generic and specific repositories with unit of work, owner-scoped queries and tenant query filters, safe raw SQL (no injection), rowversion concurrency, paging and query performance, migrations workflow and review, and connection security (Key Vault or passwordless Azure SQL, least-privilege database users). Use whenever touching DbContext, entity configurations, repositories, LINQ or raw SQL queries, migrations or connection strings.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: efcore-data-access/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# EF Core data access (Infrastructure)

> **Placeholders:** names and numbers in examples (such as `AppDbContext`, `Orders`, length 64, timeouts) are illustrations. Use the repo's names and confirmed values from project-facts; if missing, ask.

## Rules (most important first)
1. **Every query for owned data is scoped to the caller**: an explicit owner predicate in the repository method, plus a tenant predicate if the project is multi-tenant. Confirm the tenancy model in project-facts before adding tenant filters.
2. **No string-built SQL.** Use LINQ, or `FromSql($"...")` / `SqlQuery($"...")` / `ExecuteSql($"...")`, where interpolated values become parameters. `FromSqlRaw` / `ExecuteSqlRaw` only with explicit parameters, never concatenation. Table and column names never come from input.
3. **Global query filters are a safety net, not the only control.** `IgnoreQueryFilters()` needs a written reason and must re-apply the owner/tenant check.
4. **Reads**: `AsNoTracking()`; filter, then order, then page (`Skip`/`Take`) in the database; project to read models; never load whole tables; use `AsSplitQuery()` when several collection includes multiply rows.
5. **Writes**: through repositories and the unit of work; one `SaveChangesAsync` per use case. With `EnableRetryOnFailure`, wrap explicit transactions in `CreateExecutionStrategy().ExecuteAsync(...)`.
6. **Concurrency**: `RowVersion` configured with `.IsRowVersion()`; `DbUpdateConcurrencyException` maps to 409, or 412 with ETags.
7. **Column sizes match validation**: `HasMaxLength` equals the DTO's `[StringLength]`, so the database is a second line of defence.
8. **Sensitive data**: `EnableSensitiveDataLogging` and `EnableDetailedErrors` only in Development. Column encryption (for example Always Encrypted) is a team decision; ask.
9. **Connections**: no passwords in the repo. Use the passwordless managed identity option if the database is Azure SQL (confirm in project-facts); otherwise the connection string comes from Key Vault. `Encrypt=True`; never `TrustServerCertificate=True` outside local development.
10. **Least privilege**: the app's runtime database identity reads and writes data only; schema changes run under a separate deployment identity.
11. **Migrations**: generate, review for data loss, never edit an applied migration, and apply through the pipeline, not at app startup in production. Never `EnsureCreated()` outside tests.
12. **Tests** use SQL Server (Testcontainers or a test database), not the EF Core InMemory provider, for anything involving filters, constraints or raw SQL.

## References
- [efcore-patterns.md](references/efcore-patterns.md): DbContext with optional tenant filter, entity configuration, owner-scoped repository, paging, raw SQL safe vs unsafe, concurrency, DI registration, connection strings, migrations workflow and review checklist.

## Done checklist
- [ ] Owned-data queries filter by owner (and tenant if multi-tenant)
- [ ] No concatenated SQL; no identifiers from input
- [ ] `AsNoTracking` reads, database-side paging, projections
- [ ] Max lengths match DTO validation; `RowVersion` where needed
- [ ] Migration reviewed: no unintended drops, renames or type narrowing; no secrets or real personal data in seed data
- [ ] Sensitive logging only in Development
- [ ] Full-stack tests prove another owner cannot read or change the data
