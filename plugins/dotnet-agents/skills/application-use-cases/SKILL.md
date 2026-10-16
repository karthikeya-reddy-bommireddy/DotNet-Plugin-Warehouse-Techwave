---
name: application-use-cases
description: Application layer for the .NET 10 Clean Architecture APIs repo - use case interfaces and implementations, request/response DTOs, AutoMapper profiles, Data Annotations validation, ports (IRepository<T>, IUnitOfWork, ICurrentUser), owner/tenant filtering against BOLA, setting server-controlled fields, one transaction per use case, cancellation, error conventions, and background job logic without an HTTP user. Use whenever writing or changing business logic behind an endpoint, a DTO, a mapping profile, a port interface or a background job.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: application-use-cases/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# Application use cases

> **Placeholders:** names and numbers in examples (such as `Order`, `IOrderRepository`, `NotFoundException`) are illustrations. Follow the repo's real names and its error convention; if missing, ask.

## Rules (most important first)
1. **Object-level security happens here.** Every use case that reads or changes owned data takes the caller from `ICurrentUser` and passes the owner (and tenant, if multi-tenant) to the repository. Data the caller may not access comes back as "not found".
2. **Server-controlled fields** (owner, tenant, status, totals, audit fields) are set here from `ICurrentUser`, `TimeProvider` and domain logic; never copied from a request DTO.
3. **IDs inside requests** (product, assignee, account) are checked against the caller's scope before use.
4. **One use case per operation**, following the repo's naming and structure. Do not add a mediator or other library the spec does not list; that is a team decision.
5. **One unit of work per use case**: call domain methods, then `IUnitOfWork.SaveChangesAsync(ct)` once.
6. **Ports are business-shaped**: prefer `GetForOwnerAsync(id, tenantId, ownerId)` over a generic `GetByIdAsync(id)` for user-facing reads. Never return `IQueryable` from a port; it lets callers skip filters. A generic `GetByIdAsync` is only for code that has already authorized the object.
7. **DTOs per operation** (separate create, update and response types), validated with Data Annotations (api-input-validation); mapping per feature with safe AutoMapper profiles (api-input-validation/references/automapper-safety.md).
8. **Cancellation and async**: accept and pass `CancellationToken` everywhere; no `.Result` or `.Wait()`.
9. **Errors** follow the repo's convention (exceptions mapped by the global handler, or result types); confirm it in project-facts. Messages are safe for clients.
10. **Background jobs and message handlers** have no HTTP user. The message carries owner/tenant explicitly; validate the payload as untrusted input and apply the same checks.

## Example
```csharp
// Ports (Application)
public interface IOrderRepository
{
    Task<Order?> GetForOwnerAsync(Guid id, string tenantId, string ownerObjectId, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

public interface IProductRepository
{
    Task<bool> AllExistInTenantAsync(IReadOnlyCollection<Guid> productIds, string tenantId, CancellationToken ct);
}

public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}

// Read: owner-scoped, null when missing or not the caller's
public sealed class GetOrderUseCase(IOrderRepository orders, ICurrentUser currentUser, IMapper mapper)
    : IGetOrderUseCase
{
    public async Task<OrderResponse?> ExecuteAsync(Guid id, CancellationToken ct)
    {
        var order = await orders.GetForOwnerAsync(id, currentUser.TenantId, currentUser.ObjectId, ct);
        return order is null ? null : mapper.Map<OrderResponse>(order);
    }
}

// Write: server-controlled fields from ICurrentUser and TimeProvider; body IDs checked
public sealed class CreateOrderUseCase(
    IOrderRepository orders, IProductRepository products, IUnitOfWork unitOfWork,
    ICurrentUser currentUser, TimeProvider clock, IMapper mapper) : ICreateOrderUseCase
{
    public async Task<OrderResponse> ExecuteAsync(CreateOrderRequest request, CancellationToken ct)
    {
        var productIds = request.Lines.Select(l => l.ProductId!.Value).Distinct().ToArray();
        if (!await products.AllExistInTenantAsync(productIds, currentUser.TenantId, ct))
            throw new NotFoundException("One or more products were not found.");

        var order = Order.Create(Guid.NewGuid(), currentUser.TenantId, currentUser.ObjectId,
                                 request.CustomerReference, clock.GetUtcNow());
        foreach (var line in request.Lines)
            order.AddLine(line.ProductId!.Value, line.Quantity!.Value);

        await orders.AddAsync(order, ct);
        await unitOfWork.SaveChangesAsync(ct);
        return mapper.Map<OrderResponse>(order);
    }
}
```
In a single-tenant project (confirm in project-facts), drop the tenant parameters and keep the owner checks.

## Registration
Register use cases as scoped in the layer's registration method (for example `services.AddScoped<IGetOrderUseCase, GetOrderUseCase>()`). Register `TimeProvider.System` once in the composition root so tests can replace it.

## Tests
- Unit tests with fakes for repositories and `ICurrentUser`: another user's object returns null, body IDs from another scope are rejected, server-controlled fields come from the fakes.
- BOLA end to end with a real database: api-security-testing, full-stack level.
