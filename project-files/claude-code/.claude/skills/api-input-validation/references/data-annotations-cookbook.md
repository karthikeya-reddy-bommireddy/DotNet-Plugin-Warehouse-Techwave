# Data Annotations cookbook

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Request DTO example
```csharp
using System.ComponentModel.DataAnnotations;

public sealed class CreateOrderRequest
{
    [Required, StringLength(64, MinimumLength = 1)]
    [RegularExpression("^[A-Za-z0-9-]+$", MatchTimeoutInMilliseconds = 100)]
    public required string CustomerReference { get; init; }

    [Required, Length(1, 50)]
    public required IReadOnlyList<OrderLineRequest> Lines { get; init; }

    [StringLength(500)]
    public string? Notes { get; init; }

    [EnumDataType(typeof(DeliverySpeed))]
    public DeliverySpeed Delivery { get; init; } = DeliverySpeed.Standard;

    // Deliberately absent: Id, OwnerObjectId, TenantId, Status, Total, CreatedAt.
}

public sealed class OrderLineRequest
{
    [Required, NotDefault]
    public Guid? ProductId { get; init; }

    [Required, Range(1, 1000)]
    public int? Quantity { get; init; }
}
```
- MVC validates nested objects and every collection element.
- With nullable reference types on, MVC treats non-nullable reference properties as required; keep `[Required]` for clarity.
- `required` members make System.Text.Json reject bodies that omit them (400).
- IDs inside the body (like `ProductId`) are still checked against the caller's tenant in the Application layer: BOLA applies to body IDs too.

## Custom attribute
```csharp
[AttributeUsage(AttributeTargets.Property | AttributeTargets.Parameter)]
public sealed class NotDefaultAttribute : ValidationAttribute
{
    public NotDefaultAttribute() : base("The {0} field must not be empty.") { }

    public override bool IsValid(object? value) => value switch
    {
        null => true,                 // presence is [Required]'s job
        Guid g => g != Guid.Empty,
        DateTime d => d != default,
        DateOnly d => d != default,
        _ => true
    };
}
```

## Cross-field rules
```csharp
public sealed class DateRangeRequest : IValidatableObject
{
    [Required] public DateOnly? From { get; init; }
    [Required] public DateOnly? To { get; init; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (From is null || To is null) yield break;
        if (To < From)
            yield return new ValidationResult("'to' must be on or after 'from'.", [nameof(To)]);
        if (To.Value.DayNumber - From.Value.DayNumber > 366)
            yield return new ValidationResult("The range cannot exceed one year.", [nameof(To)]);
    }
}
```
`Validate` runs only after the property attributes pass.

## Binding pitfalls
| Pitfall | Effect | Fix |
|---|---|---|
| `int Quantity` (not nullable) | A missing value binds as 0 and passes `[Required]` | `int?` + `[Required]` |
| Enum without `[EnumDataType]` | `"delivery": 999` binds to an undefined value | `[EnumDataType(typeof(DeliverySpeed))]` |
| Client-supplied `Id` on create | Client chooses identifiers | Server generates IDs |
| `[FromQuery]` complex object for writes | Data ends up in URLs and logs | `[FromBody]` |
| One DTO for create, update and response | Fields leak between operations and roles | Separate DTOs |
| `[BindNever]` on JSON DTO properties | Ignored by the JSON input formatter | Remove the property from the DTO |
| Regex without timeout | ReDoS | `MatchTimeoutInMilliseconds` or no regex |
| `[Url]` on a URL the server will fetch | Accepts internal hosts | SSRF allowlist (api-pipeline-hardening) |

## Validation tests
- Unit-test attributes with `Validator.TryValidateObject(dto, new ValidationContext(dto), results, validateAllProperties: true)`. It does not descend into nested objects or collection elements; test those separately or through the endpoint.
- Integration-test the endpoint: each invalid case returns 400 ProblemDetails naming the field, with no server internals (api-security-testing).
