# EF Core patterns

> **Placeholders:** names and numbers in examples (such as `AppDbContext`, `Orders`, length 64, 30-second timeout) are illustrations. Use the repo's names and confirmed values from `.agents/api-layer/project-facts.md`, or ask the user.

Contents: 1 DbContext · 2 entity configuration · 3 repository · 4 raw SQL · 5 concurrency · 6 registration · 7 connection strings · 8 migrations

## 1. DbContext (implements the unit of work)
```csharp
public sealed class AppDbContext(DbContextOptions<AppDbContext> options, ICurrentUser currentUser)
    : DbContext(options), IUnitOfWork
{
    public DbSet<Order> Orders => Set<Order>();

    // Evaluated per query, so each request gets its own tenant.
    // A missing tenant throws, so queries fail closed.
    private string CurrentTenantId => currentUser.TenantId;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);

        // ONLY if the project is confirmed multi-tenant. EF Core 10 named filters let other
        // filters (e.g. soft delete) be ignored without dropping the tenant filter.
        modelBuilder.Entity<Order>().HasQueryFilter("Tenant", o => o.TenantId == CurrentTenantId);
    }
}
```
- Background jobs need an `ICurrentUser` implementation that carries the job's tenant and owner; otherwise filtered queries throw.
- `IUnitOfWork.SaveChangesAsync(CancellationToken)` is satisfied by `DbContext.SaveChangesAsync`.

## 2. Entity configuration (Fluent API)
```csharp
internal sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.ToTable("Orders");
        b.HasKey(o => o.Id);
        b.Property(o => o.TenantId).HasMaxLength(36).IsRequired();
        b.Property(o => o.OwnerObjectId).HasMaxLength(36).IsRequired();
        b.Property(o => o.Reference).HasMaxLength(64).IsRequired();   // same as the DTO's [StringLength(64)]
        b.Property(o => o.Status).HasConversion<string>().HasMaxLength(32);
        b.Property(o => o.RowVersion).IsRowVersion();
        b.HasIndex(o => new { o.TenantId, o.OwnerObjectId, o.CreatedAt });
        // Configure Lines (one-to-many or owned) following the repo's convention;
        // EF Core finds the _lines backing field by convention.
    }
}
```

## 3. Owner-scoped repository
```csharp
internal sealed class OrderRepository(AppDbContext db) : IOrderRepository
{
    public Task<Order?> GetForOwnerAsync(Guid id, string tenantId, string ownerObjectId, CancellationToken ct) =>
        db.Orders
          .Include(o => o.Lines)
          .SingleOrDefaultAsync(o => o.Id == id && o.TenantId == tenantId && o.OwnerObjectId == ownerObjectId, ct);

    public async Task AddAsync(Order order, CancellationToken ct) => await db.Orders.AddAsync(order, ct);

    public async Task<PagedResult<OrderSummary>> ListForOwnerAsync(
        string tenantId, string ownerObjectId, int page, int pageSize, CancellationToken ct)
    {
        pageSize = Math.Clamp(pageSize, 1, MaxPageSize);     // MaxPageSize: confirmed value from project-facts

        var query = db.Orders.AsNoTracking()
            .Where(o => o.TenantId == tenantId && o.OwnerObjectId == ownerObjectId);

        var total = await query.CountAsync(ct);
        var items = await query
            .OrderByDescending(o => o.CreatedAt).ThenBy(o => o.Id)       // stable order before paging
            .Skip((page - 1) * pageSize).Take(pageSize)
            .Select(o => new OrderSummary(o.Id, o.Reference, o.Status, o.CreatedAt))
            .ToListAsync(ct);

        return new PagedResult<OrderSummary>(items, total);
    }
}
```
The explicit predicates stay even with a tenant query filter (defence in depth). In a single-tenant project, drop the tenant parts.

## 4. Raw SQL: safe vs unsafe
```csharp
// SAFE: interpolated values become DbParameters; query filters still compose on top
var rows = await db.Orders
    .FromSql($"SELECT * FROM dbo.Orders WHERE Reference = {reference}")
    .AsNoTracking()
    .ToListAsync(ct);

// UNSAFE (SQL injection): concatenation, or a pre-built string passed to a *Raw method
var bad = db.Orders.FromSqlRaw("SELECT * FROM dbo.Orders WHERE Reference = '" + reference + "'");
```
Never build `ORDER BY` columns or table names from input; map allowlisted enums to fixed expressions.

## 5. Concurrency
```csharp
try
{
    await unitOfWork.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException)
{
    throw new ConflictException("The record was changed by someone else. Reload and try again.");
}
```
`ConflictException` stands for the repo's own error type; the global handler maps it to 409 (or 412 with ETags).

## 6. Registration
```csharp
public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration configuration)
{
    services.AddDbContext<AppDbContext>((sp, options) =>
    {
        options.UseSqlServer(configuration.GetConnectionString("Default"), sql =>
        {
            sql.EnableRetryOnFailure();
            sql.CommandTimeout(30);
        });

        if (sp.GetRequiredService<IHostEnvironment>().IsDevelopment())
            options.EnableSensitiveDataLogging().EnableDetailedErrors();
    });

    services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());
    services.AddScoped<IOrderRepository, OrderRepository>();
    return services;
}
```

## 7. Connection strings (no passwords in the repo)
- Azure SQL with managed identity (only if confirmed): `Server=tcp:<server>.database.windows.net,1433;Database=<database>;Authentication=Active Directory Managed Identity;Encrypt=True;` in App Service settings. Locally, developers can use `Authentication=Active Directory Default` with their own sign-in.
- The database administrator grants the app identity only data rights (for example `db_datareader` and `db_datawriter`), never `db_owner` or schema-change rights.
- Other SQL Server setups: the full connection string lives in Key Vault (api-pipeline-hardening).

## 8. Migrations
Workflow (project names from project-facts):
1. `dotnet ef migrations add <Name> --project <Infrastructure project> --startup-project <API project>`
2. Review the generated migration with the checklist below, and show it in your report.
3. Produce a script or bundle for the pipeline (`dotnet ef migrations script --idempotent` or `dotnet ef migrations bundle`), applied by a deployment identity that has schema rights. How it is applied per environment is a project fact; ask if not recorded.

Review checklist:
- [ ] No unintended `DropTable`, `DropColumn` or rename done as drop + add (data loss)
- [ ] No type narrowing or nullable-to-required change without a data plan
- [ ] Indexes on large tables considered for locking impact
- [ ] Seed data contains no secrets and no real personal data
- [ ] Never edit a migration already applied to a shared environment; add a new one
