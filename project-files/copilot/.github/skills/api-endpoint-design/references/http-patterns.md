# HTTP patterns

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Paging and sorting
```csharp
public sealed class ListOrdersRequest
{
    public const int MaxPageSize = 100;

    [Range(1, 10_000)]
    public int Page { get; init; } = 1;

    [Range(1, MaxPageSize)]
    public int PageSize { get; init; } = 20;

    [EnumDataType(typeof(OrderSort))]   // enums bind any integer; this rejects undefined values
    public OrderSort Sort { get; init; } = OrderSort.CreatedDesc;

    [StringLength(100)]
    public string? Search { get; init; }
}

public enum OrderSort { CreatedDesc, CreatedAsc, TotalDesc, TotalAsc }
```
- Map the sort enum to expressions with a `switch` in the Application layer. Never pass client strings to dynamic LINQ, `OrderBy(string)` or SQL.
- Deep offset paging is slow and costly: cap `Page`, and prefer keyset (cursor) paging for large lists. Cursors must be opaque; a decoded cursor is untrusted input, so owner/tenant filters are re-applied.

## Versioning and deprecation (Asp.Versioning.Mvc + Asp.Versioning.Mvc.ApiExplorer)
```csharp
builder.Services.AddApiVersioning(o =>
    {
        o.DefaultApiVersion = new ApiVersion(1, 0);
        o.AssumeDefaultVersionWhenUnspecified = false;   // explicit versions only
        o.ReportApiVersions = true;                      // api-supported-versions / api-deprecated-versions headers
    })
    .AddMvc()
    .AddApiExplorer(o =>
    {
        o.GroupNameFormat = "'v'VVV";
        o.SubstituteApiVersionInUrl = true;
    });
```
- Mark retiring versions `[ApiVersion("1.0", Deprecated = true)]`, publish a sunset date and remove them on schedule. Every live version receives the same security fixes; an old version with weaker checks is an open door (API9).
- Never ship debug, test or "temporary" controllers in release builds.

## Idempotency for create and payment-like operations (API6)
Clients send `Idempotency-Key: <uuid>`. The server stores (key, caller oid, route, body hash) with the response for a fixed window, for example 24 hours:
- Same key, same caller, same body → return the stored response; perform nothing again.
- Same key, different body → 409 (or 422).
- Keys are scoped per caller, so nobody can replay or read another user's result.

This needs storage (a database table or a distributed cache), and the spec has no cache yet: ask the user which store to use before building it.

## Optimistic concurrency with ETags
- Expose the entity's `rowversion` as a strong ETag (base64) on GET; require `If-Match` on PUT, PATCH and DELETE for concurrency-sensitive resources.
- Missing required `If-Match` → 428; mismatch → 412. The Application layer compares versions; the API layer only reads and writes headers.
- Allow `If-Match` and expose `ETag` in the CORS policy.

## Long-running operations
- Return 202 with `Location: /api/v1/operations/{operationId}`.
- The status endpoint is authorized like the original call (same owner/tenant check). Operation IDs are random GUIDs.
- Expire results; keep sensitive results no longer than the business needs.

## PATCH
Avoid `JsonPatchDocument<T>` on entities: it lets clients address any property path. Prefer a dedicated request DTO with only patchable fields (null = not provided), or apply JSON Patch to a DTO and allowlist the paths.
