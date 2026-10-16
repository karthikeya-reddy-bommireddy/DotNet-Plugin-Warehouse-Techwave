---
name: domain-model
description: Domain layer modeling for the .NET 10 / C# 14 Clean Architecture APIs repo - entities with protected invariants, value objects, business rules and status transitions, owner/tenant and audit fields, concurrency tokens, domain errors, time and ID handling, and C# 14 field-backed properties. Use whenever creating or changing an entity, value object, enum, status or business rule, or when a request mentions rules, states, statuses or conditions that must always hold.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: domain-model/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# Domain model

> **Placeholders:** names, rules and numbers in examples (such as `Order`, "lines only on draft orders", length 64) are illustrations. Real business rules come from the user or project-facts; never invent one.

## Rules (most important first)
1. **Never invent a business rule.** If a rule is not in the request, the code or project-facts, ask. Example: "Can a shipped order still be cancelled, and by whom?"
2. **Invariants live in the entity.** State changes go through methods (`order.Cancel(...)`), not public setters. Constructors or factory methods reject invalid state.
3. **Owner, tenant and audit fields** (`OwnerObjectId`, `TenantId`, `CreatedAt`, `CreatedBy`) are set once at creation by the Application layer from `ICurrentUser` and `TimeProvider`. They are never changeable later and never come from client input.
4. **Concurrency token** (`byte[] RowVersion`) on aggregates that users edit concurrently; Infrastructure configures it.
5. **Value objects** for concepts with rules (money, email address, reference codes): immutable `record` types that validate on creation.
6. **Domain errors** follow the repo's convention (domain exceptions or result types; check project-facts). Messages are safe to show and contain no personal data or internal details.
7. **No outside dependencies**: no EF Core attributes, logging, DTOs or AutoMapper. Persistence mapping lives in Infrastructure (Fluent API).
8. **Collections** are exposed read-only, with changes through methods that enforce limits.
9. **Time and IDs**: take the current time as a parameter (from `TimeProvider` in the use case). Use the repo's ID strategy; if none is recorded, ask (options: `Guid.NewGuid()`, `Guid.CreateVersion7()` for index-friendly ordering, database identity).

## Example entity
```csharp
public sealed class Order
{
    private readonly List<OrderLine> _lines = [];

    private Order() { }                       // for EF Core materialization

    public Guid Id { get; private set; }
    public string TenantId { get; private set; } = default!;
    public string OwnerObjectId { get; private set; } = default!;
    public OrderStatus Status { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; }
    public byte[] RowVersion { get; private set; } = [];
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();

    // C# 14 field-backed property: validation without a hand-written backing field
    public string Reference
    {
        get;
        private set => field = string.IsNullOrWhiteSpace(value) || value.Length > 64
            ? throw new DomainRuleException("Reference is required and at most 64 characters.")
            : value;
    } = default!;

    public static Order Create(Guid id, string tenantId, string ownerObjectId, string reference, DateTimeOffset now)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(tenantId);
        ArgumentException.ThrowIfNullOrWhiteSpace(ownerObjectId);
        return new Order
        {
            Id = id, TenantId = tenantId, OwnerObjectId = ownerObjectId,
            Reference = reference, Status = OrderStatus.Draft, CreatedAt = now
        };
    }

    public void AddLine(Guid productId, int quantity)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainRuleException("Lines can only be added to draft orders.");   // example rule
        if (quantity <= 0)
            throw new DomainRuleException("Quantity must be positive.");
        _lines.Add(new OrderLine(productId, quantity));
    }
}
```
`DomainRuleException` stands for the repo's own domain error type. The global exception handler maps it to 400 or 409 (api-pipeline-hardening).

## Status transitions
Model allowed transitions explicitly (a method per transition, or a transition table) and reject everything else. Every transition rule is a business rule: confirm it with the user before coding it.

## Tests
Unit-test every invariant and transition: valid cases pass, invalid ones throw or return the domain error (api-security-testing, unit level).
