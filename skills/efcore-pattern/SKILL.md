---
name: efcore-pattern
description: >-
  Practical guide to designing, implementing, and refactoring Entity Framework Core design patterns (EF Core 9/.NET 10, C# 14).
  Covers separate IEntityTypeConfiguration, AsNoTracking & DTO projection optimization, N+1 query elimination,
  automatic audit trail interceptors, native SQL bulk update/delete, standard pagination, and an evaluation checklist.
  Use when designing a persistence layer, writing database repositories/services, optimizing slow queries, or reviewing EF Core code.
---

# EF Core Design Patterns & Best Practices

Standard architecture guide and implementation patterns for **Entity Framework Core (EF Core 9 / .NET 10 & C# 14)** for modern backend applications that are high-performance, clean, and maintainable.

---

## 1. When to Use This Skill

### ✅ Activation Triggers (Use this skill when):
- Designing database entity models, a DbContext, or repositories in .NET.
- Writing data read queries (GET) or data mutations (POST/PUT/DELETE) with EF Core.
- Refactoring slow queries, fixing *N+1 query* problems, or removing *tracking leakage*.
- Implementing automatic audit columns (`CreatedAt`, `UpdatedAt`) or soft-delete.
- Implementing bulk update or delete operations (*bulk operations*).
- Performing code reviews or compliance audits against EF Core database best practices.

### ❌ Boundaries (Do not use this skill when):
- The project uses a pure Micro-ORM (pure Dapper without EF Core).
- The project uses a NoSQL database without an EF Core provider (MongoDB native driver, native Cosmos SDK).
- The project is based on advanced Event Sourcing or a distributed ledger architecture.

---

## 2. Core Guardrails (Absolute AI Rules)

When writing or modifying EF Core code, the AI agent **MUST** follow these 6 absolute rules:

1. **`.AsNoTracking()` Required for Reads**: Every read query (GET/query) must use `.AsNoTracking()` or project directly to a DTO via `.Select()`.
2. **Separate Entity Configuration**: Do not use database Data Annotations (`[Table]`, `[Column]`, `[ForeignKey]`) in entity models. Use a separate `IEntityTypeConfiguration<T>` class.
3. **No Queries Inside Loops (N+1)**: Loading related data must be done via eager loading (`.Include()`) or `.Select()` projection. Do not call the database inside a `foreach`/`for` loop.
4. **Always Forward `CancellationToken`**: Every async EF Core call (`ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync`, `ExecuteUpdateAsync`, etc.) must accept and forward a `CancellationToken cancellationToken = default` parameter.
5. **Use Bulk Operations for Multi-Row Mutations**: Do not load tens/hundreds of rows into memory just to change a column or delete data. Use `ExecuteUpdateAsync()` or `ExecuteDeleteAsync()`.
6. **`TimeProvider` Abstraction Required**: Do not use a static `DateTime.UtcNow` in interceptors or domain logic. Use an injected `TimeProvider` dependency.

---

## 3. Standard EF Core Design Patterns (Implementation & Code)

### Pattern 1: Separate Entity Configuration (`IEntityTypeConfiguration<T>`)

Move schema, relationship, and index configuration into separate files so entity models stay clean.

```csharp
// 1. Clean Entity Model
public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public bool IsActive { get; set; } = true;
    public Guid CategoryId { get; set; }
    public Category Category { get; set; } = null!;
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? UpdatedAt { get; set; }
}

// 2. Separate Configuration Class (Data/Configurations/ProductConfiguration.cs)
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");

        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .HasMaxLength(150)
            .IsRequired();

        builder.Property(p => p.Price)
            .HasPrecision(18, 2);

        // Explicit one-to-many relationship
        builder.HasOne(p => p.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId)
            .OnDelete(DeleteBehavior.Restrict);

        // Index for frequently searched columns
        builder.HasIndex(p => p.CategoryId);
    }
}

// 3. Automatic Registration in DbContext
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        // Automatically registers every IEntityTypeConfiguration in the assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

---

### Pattern 2: Read vs Write Separation (Query Optimization)

Clearly separate read and write data flows for ChangeTracker memory efficiency.

#### A. Read-Only Query (AsNoTracking & DTO Projection)
```csharp
public async Task<List<ProductResponse>> GetActiveProductsAsync(CancellationToken cancellationToken)
{
    // Read only the needed columns and JOIN directly on the database server
    return await context.Products
        .AsNoTracking()
        .Where(p => p.IsActive)
        .OrderBy(p => p.Name)
        .Select(p => new ProductResponse(
            p.Id,
            p.Name,
            p.Price,
            p.Category.Name // Automatic SQL LEFT JOIN
        ))
        .ToListAsync(cancellationToken);
}
```

#### B. Single Entity Mutation (Tracked Entity)
```csharp
public async Task<bool> UpdateProductPriceAsync(Guid id, decimal newPrice, CancellationToken cancellationToken)
{
    var product = await context.Products.FirstOrDefaultAsync(p => p.Id == id, cancellationToken);
    if (product is null) return false;

    product.Price = newPrice;
    await context.SaveChangesAsync(cancellationToken);
    return true;
}
```

---

### Pattern 3: Preventing the N+1 Query Problem

```csharp
// ❌ ANTI-PATTERN: N+1 query inside a loop
var products = await context.Products.ToListAsync(cancellationToken);
foreach (var p in products)
{
    // Executes N roundtrips to the database!
    var category = await context.Categories.FirstOrDefaultAsync(c => c.Id == p.CategoryId, cancellationToken);
}

// ✅ SOLUTION 1: Eager loading with .Include()
var products = await context.Products
    .AsNoTracking()
    .Include(p => p.Category)
    .ToListAsync(cancellationToken);

// ✅ SOLUTION 2: .Select() projection (most memory- and bandwidth-efficient)
var productSummaries = await context.Products
    .AsNoTracking()
    .Select(p => new ProductSummary(p.Id, p.Name, p.Category.Name))
    .ToListAsync(cancellationToken);
```

---

### Pattern 4: Specific Repository / Direct Service

Avoid a generic `IRepository<T>` abstraction that leaks `IQueryable`. Create specific repositories that express business needs, or inject `AppDbContext` directly into the Service layer.

```csharp
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<List<ProductResponse>> GetByCategoryAsync(Guid categoryId, CancellationToken cancellationToken = default);
    Task<bool> ExistsByNameAsync(string name, CancellationToken cancellationToken = default);
    Task AddAsync(Product product, CancellationToken cancellationToken = default);
    Task SaveChangesAsync(CancellationToken cancellationToken = default);
}

public class ProductRepository(AppDbContext context) : IProductRepository
{
    public Task<Product?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default) =>
        context.Products.FirstOrDefaultAsync(p => p.Id == id, cancellationToken);

    public Task<List<ProductResponse>> GetByCategoryAsync(Guid categoryId, CancellationToken cancellationToken = default) =>
        context.Products
            .AsNoTracking()
            .Where(p => p.CategoryId == categoryId)
            .Select(p => new ProductResponse(p.Id, p.Name, p.Price, p.Category.Name))
            .ToListAsync(cancellationToken);

    public Task<bool> ExistsByNameAsync(string name, CancellationToken cancellationToken = default) =>
        context.Products.AnyAsync(p => p.Name == name, cancellationToken);

    public async Task AddAsync(Product product, CancellationToken cancellationToken = default) =>
        await context.Products.AddAsync(product, cancellationToken);

    public Task SaveChangesAsync(CancellationToken cancellationToken = default) =>
        context.SaveChangesAsync(cancellationToken);
}
```

---

### Pattern 5: Automatic Audit Trail via `SaveChangesInterceptor`

Fill the `CreatedAt` and `UpdatedAt` columns centrally without writing them manually in Controllers/Services.

```csharp
// 1. Marker Interface
public interface IAuditable
{
    DateTimeOffset CreatedAt { get; set; }
    DateTimeOffset? UpdatedAt { get; set; }
}

// 2. Interceptor
public class AuditInterceptor(TimeProvider timeProvider) : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Context is null)
            return base.SavingChangesAsync(eventData, result, cancellationToken);

        var now = timeProvider.GetUtcNow();

        foreach (var entry in eventData.Context.ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = now;
            }
            else if (entry.State == EntityState.Modified)
            {
                entry.Entity.UpdatedAt = now;
            }
        }

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}

// 3. Dependency Injection Configuration in Program.cs
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddSingleton<AuditInterceptor>();
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"))
           .AddInterceptors(sp.GetRequiredService<AuditInterceptor>());
});
```

---

### Pattern 6: Bulk Update & Delete (`ExecuteUpdateAsync` / `ExecuteDeleteAsync`)

Execute SQL `UPDATE` or `DELETE` statements directly against the database without loading thousands of rows into RAM.

```csharp
var now = timeProvider.GetUtcNow();

// Bulk update directly in the database
int updatedRows = await context.Products
    .Where(p => p.CategoryId == categoryId && p.IsActive)
    .ExecuteUpdateAsync(setter => setter
        .SetProperty(p => p.Price, p => p.Price * 1.10m)
        .SetProperty(p => p.UpdatedAt, now),
        cancellationToken);

// Bulk delete directly in the database
int deletedRows = await context.Products
    .Where(p => !p.IsActive && p.CreatedAt < cutOffDate)
    .ExecuteDeleteAsync(cancellationToken);
```

---

### Pattern 7: Standard Pagination (Offset / PagedResult)

An efficient pagination formula that limits record retrieval at the database server level:

```csharp
public record PagedResult<T>(List<T> Items, int TotalCount, int PageNumber, int PageSize);

public async Task<PagedResult<ProductResponse>> GetPagedProductsAsync(
    int pageNumber, 
    int pageSize, 
    CancellationToken cancellationToken)
{
    var query = context.Products
        .AsNoTracking()
        .Where(p => p.IsActive);

    int totalCount = await query.CountAsync(cancellationToken);

    var items = await query
        .OrderBy(p => p.Name)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductResponse(p.Id, p.Name, p.Price, p.Category.Name))
        .ToListAsync(cancellationToken);

    return new PagedResult<ProductResponse>(items, totalCount, pageNumber, pageSize);
}
```

---

## 4. Anti-Patterns & Refactoring Solutions

| Anti-Pattern | Negative Impact | Standard Solution |
| :--- | :--- | :--- |
| **Generic `IRepository<T>`** | Leaky abstraction, scattered query filters | Use a **Specific Repository** or DbContext directly in the service layer. |
| **N+1 Query Problem** | Tens/hundreds of database roundtrips | Use **`.Include()`** or a **`.Select()`** projection. |
| **In-Memory Filtering** | Loads millions of records into RAM (`.ToList()` then `.Where()`) | Put the **`.Where()` filter before `.ToListAsync()`**. |
| **Missing `AsNoTracking`** | Memory bloat on GET operations | Add **`.AsNoTracking()`** to every read query. |
| **Looping `SaveChanges`** | Repeated, slow database transactions | Use **`ExecuteUpdateAsync`** or a single **`SaveChangesAsync`**. |
| **`CountAsync() > 0`** | Counts every matching row (full scan) | Use **`.AnyAsync()`** (produces SQL `EXISTS`). |
| **Hardcoded `DateTime.UtcNow`** | Hard to test deterministically | Inject the **`TimeProvider`** abstraction. |

---

## 5. Evaluation & Verification Checklist

Use this checklist when building new features, refactoring database queries, or reviewing pull requests (PRs):

### A. Models & Table Configuration
- [ ] **Separate Configuration**: Configuration files are split per model using `IEntityTypeConfiguration<T>`, not piled up as `[Table]` / `[Column]` attributes on entity classes.
- [ ] **Auto Apply Configuration**: `AppDbContext.cs` calls `modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);`.
- [ ] **String Length & Decimal Precision**:
  - Text columns have a maximum length: `.HasMaxLength(...)`.
  - Decimal columns (prices, money) have precision: `.HasPrecision(18, 2)`.
- [ ] **Indexes on Foreign Keys**: Foreign key columns or frequently filtered columns have `.HasIndex(...)`.

### B. Data Read Queries (GET)
- [ ] **Use `.AsNoTracking()`**: Every query that only reads/displays data includes `.AsNoTracking()`.
- [ ] **DTO Projection (`.Select()`)**: Avoid returning database entity models directly to Controllers; use `.Select(x => new MyDto(...))` to fetch only the needed columns.
- [ ] **Free of N+1 Queries**:
  - No database query is called inside a `foreach`/`for` loop.
  - Use `.Include()` for relationships or rely on `.Select()`.
- [ ] **Efficient Existence Checks**: Use `.AnyAsync(...)` instead of `CountAsync() > 0` or `FirstOrDefaultAsync() != null`.
- [ ] **Pagination in the Database**: `.Skip().Take()` pagination runs on the `IQueryable` before `.ToListAsync()`.

### C. Create, Update & Delete (POST, PUT, DELETE)
- [ ] **Tracked Entity for Single Updates**: The entity is loaded from the DbContext, its values are changed, then saved with `await context.SaveChangesAsync(cancellationToken)`.
- [ ] **Bulk Update / Bulk Delete**: When changing or deleting many rows at once, use `ExecuteUpdateAsync()` or `ExecuteDeleteAsync()` instead of loading everything into memory.
- [ ] **Single `SaveChangesAsync`**: Operations that must succeed together are saved in a single `SaveChangesAsync()` call so they run in one transaction automatically.

### D. Performance & Code Hygiene
- [ ] **`CancellationToken` Required**: Every async call (`ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync`, etc.) forwards the `cancellationToken` parameter.
- [ ] **Automatic Audit Trail**: `CreatedAt` and `UpdatedAt` are filled automatically via `SaveChangesInterceptor` (not manually in controllers/services).
- [ ] **Use `TimeProvider`**: The current time is obtained via `TimeProvider.System.GetUtcNow()` so it is easy to unit test.
- [ ] **Clean Build**: The code builds without warnings (`dotnet build`).

---

## 6. Definition of Done & Verification

Before completing an EF Core implementation or review task, run the following verification in the terminal:

```bash
# 1. Make sure the solution builds cleanly without errors or warnings
dotnet build --configuration Release

# 2. Make sure no model changes are missing from migrations
dotnet ef migrations has-pending-model-changes

# 3. Run unit and integration tests
dotnet test
```
