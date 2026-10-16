# Authorization matrix

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

One row per endpoint, one column per caller type; each cell is the expected HTTP status. Tests read this table (or a C# copy of it) as `[Theory]` data, so a new endpoint without rows fails review.

## Caller types (test identities in api-security-testing)
| Key | Description |
|---|---|
| anon | No token |
| no-perm | Valid token for this API, no scopes or app permissions |
| owner | User in tenant A with `Orders.Read Orders.Write`; owns order **X** |
| reader | Another user in tenant A with `Orders.Read` only |
| other-user | Another user in tenant A with `Orders.Read Orders.Write` |
| other-tenant | User in tenant B with `Orders.Read Orders.Write` |
| daemon | App-only token in tenant A with app permission `Orders.Read.All` |
| admin | User in tenant A with `Orders.Read Orders.Write` and app role `Api.Admin` |

Use the `other-tenant` column only if the project is confirmed multi-tenant (project-facts); otherwise delete it. Endpoints without a resource ID mark object-level cells N/A.

## Matrix (fixture: order X belongs to `owner`, tenant A)
| Endpoint | anon | no-perm | owner | reader | other-user | other-tenant | daemon | admin |
|---|---|---|---|---|---|---|---|---|
| GET /api/v1/orders/X | 401 | 403 | 200 | 404 | 404 | 404 | 200 (a) | 200 |
| GET /api/v1/orders | 401 | 403 | 200, includes X | 200, excludes X | 200, excludes X | 200, excludes X | 200 (a) | 200 (b) |
| POST /api/v1/orders | 401 | 403 | 201 | 403 | 201 | 201 | 403 | 201 |
| PUT /api/v1/orders/X | 401 | 403 | 204 | 403 | 404 | 404 | 403 | 204 |
| DELETE /api/v1/orders/X | 401 | 403 | 204 | 403 | 404 | 404 | 403 | 404 (c) |
| GET /health/live | 200 | 200 | 200 | 200 | 200 | 200 | 200 | 200 |

(a) Decide and document what a daemon may see (all orders in its tenant, or none).
(b) Decide whether admins list all orders in their tenant.
(c) Follows the example rule "admins may not delete"; adjust to the real rule.

Note the order of checks: a caller without the function permission gets 403 before any object lookup (reader on PUT); a caller with the permission but no access to the object gets 404.

## Maintaining it
- Add rows in the same PR as the endpoint.
- Any cell changing from 4xx to 2xx needs review by the team's security owner.
- Keep it in sync with the endpoint inventory test.
