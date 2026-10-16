# Golden controller

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

Adapt names to the repo's Application use cases; the shape matters more than the names.

```csharp
using Asp.Versioning;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.RateLimiting;

namespace Contoso.Api.Controllers;

[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/orders")]
[Authorize(Policy = Policies.OrdersRead)]                 // function level, applies to every action
[Produces("application/json")]
[ProducesResponseType(StatusCodes.Status401Unauthorized)]
[ProducesResponseType(StatusCodes.Status403Forbidden)]
[ProducesResponseType(StatusCodes.Status429TooManyRequests)]
public sealed class OrdersController(
    IGetOrderUseCase getOrder,                            // Application abstractions only
    IListOrdersUseCase listOrders,
    ICreateOrderUseCase createOrder) : ControllerBase
{
    [HttpGet("{id:guid}")]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderResponse>> Get([FromRoute] Guid id, CancellationToken ct)
    {
        // Object level: the use case filters by the caller's oid + tid (ICurrentUser) and
        // returns null when the order is missing OR belongs to someone else.
        var order = await getOrder.ExecuteAsync(id, ct);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpGet]
    [ProducesResponseType<PagedResponse<OrderSummaryResponse>>(StatusCodes.Status200OK)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<PagedResponse<OrderSummaryResponse>>> List(
        [FromQuery] ListOrdersRequest request, CancellationToken ct)
        => Ok(await listOrders.ExecuteAsync(request, ct)); // scoped to the caller inside the use case

    [HttpPost]
    [Authorize(Policy = Policies.OrdersWrite)]             // AND-ed with the class-level policy
    [EnableRateLimiting(RateLimitPolicies.Writes)]
    [RequestSizeLimit(64 * 1024)]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status201Created)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<ActionResult<OrderResponse>> Create(
        [FromBody] CreateOrderRequest request, CancellationToken ct)
    {
        // Owner and tenant are set inside the use case from ICurrentUser, never from the request.
        var created = await createOrder.ExecuteAsync(request, ct);
        return CreatedAtAction(nameof(Get),
            new { id = created.Id, version = HttpContext.GetRequestedApiVersion()?.ToString() },
            created);
    }
}
```

## Why it is shaped this way
- Class-level read policy plus action-level write policy: several `[Authorize]` attributes must all pass.
- `[ApiController]` returns 400 ValidationProblemDetails automatically and turns `NotFound()` into ProblemDetails.
- 404 for "not yours" keeps IDs unprobeable, and the use case never loads other users' data.
- `RequestSizeLimit` keeps this endpoint far below the global limit.
- Policy and rate-limit names come from central constant classes (api-authorization, api-pipeline-hardening).
- If the repo does not use Asp.Versioning, drop `[ApiVersion]`, use a literal `api/v1/...` route and remove `version` from the route values.

When the decision needs more than a filter (for example "owner or delegated approver may update"), load the resource's ownership view and call `IAuthorizationService.AuthorizeAsync(User, resource, Operations.Update)`; see api-authorization.

## Paged response shape
```csharp
public sealed record PagedResponse<T>(IReadOnlyList<T> Items, int Page, int PageSize, long TotalCount);
```
Return `TotalCount` only if it is cheap to compute and reveals nothing about data the caller cannot see.
