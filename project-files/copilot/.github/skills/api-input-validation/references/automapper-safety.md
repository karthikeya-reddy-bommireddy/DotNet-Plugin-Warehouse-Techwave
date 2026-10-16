# AutoMapper safety

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## 1. Version and licence (team decision; flag it in every report until resolved)
- **CVE-2026-32933 (CVSS 7.5)**: AutoMapper before 15.1.1, and 16.0.0 to 16.1.0, can be crashed by mapping deeply nested object graphs. Recursion without a depth limit exhausts the stack and the `StackOverflowException` terminates the whole process. Fixed in 15.1.1 and 16.1.1.
- From v15 AutoMapper is commercially licensed; a free Community licence covers organisations under USD 5M annual gross revenue (other conditions apply). The free 14.x line is not being patched.
- Treat any affected version as a **High** finding. Options to present: upgrade with a licence key supplied from configuration/Key Vault (recent versions also read the `AUTOMAPPER_LICENSE_KEY` environment variable), or migrate to source-generated or hand-written mapping. Do not choose for the team.
- Interim mitigation: low `JsonSerializerOptions.MaxDepth`, `MaxDepth(...)` on self-referencing maps, never mapping attacker-shaped graphs recursively.

## 2. Request DTO → entity: avoid, or do it explicitly
Prefer mapping a request DTO to an Application command and letting a domain method change the entity. If a profile must write into an entity:
```csharp
CreateMap<UpdateOrderRequest, Order>(MemberList.Destination)
    .ForMember(d => d.Id, o => o.Ignore())
    .ForMember(d => d.OwnerObjectId, o => o.Ignore())
    .ForMember(d => d.TenantId, o => o.Ignore())
    .ForMember(d => d.Status, o => o.Ignore())
    .ForMember(d => d.Total, o => o.Ignore())
    .ForMember(d => d.RowVersion, o => o.Ignore())
    .ForMember(d => d.CreatedAt, o => o.Ignore())
    .ForMember(d => d.CreatedBy, o => o.Ignore());
```
- `MemberList.Destination` plus configuration validation forces a decision for every new entity member, so a new `IsApproved` column cannot silently become client-writable.
- Never `ReverseMap()` between entities and DTOs: it creates an entity-writing map nobody reviewed.
- `_mapper.Map(request, trackedEntity)` writes every mapped member of an EF-tracked entity; review the profile before using it.

## 3. Entity → response DTO
- Explicit response DTOs per audience; never a DTO that mirrors the entity "for convenience".
- Flattening can pull sensitive navigation data (`Customer.Email` → `CustomerEmail`); review flattened names.
- `ProjectTo<T>()` builds SQL from the DTO shape: apply owner/tenant filters before projecting and keep sensitive columns out of the DTO.
- Self-referencing types: `CreateMap<Category, CategoryResponse>().MaxDepth(3);`

## 4. Configuration validation test
```csharp
public sealed class MappingConfigurationTests
{
    [Fact]
    public void AutoMapper_configuration_is_valid()
    {
        using var provider = new ServiceCollection()
            .AddLogging()
            .AddAutoMapper(cfg => cfg.LicenseKey = Environment.GetEnvironmentVariable("AUTOMAPPER_LICENSE_KEY"),
                           typeof(SomeApplicationProfile).Assembly)
            .BuildServiceProvider();

        provider.GetRequiredService<IMapper>().ConfigurationProvider.AssertConfigurationIsValid();
    }
}
```
Adjust the registration call to the AutoMapper version in use; what matters is that `AssertConfigurationIsValid()` runs in CI.
