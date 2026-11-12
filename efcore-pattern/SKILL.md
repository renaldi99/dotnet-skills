---
name: efcore-pattern
description: >-
  Panduan praktis perancangan, implementasi, dan refactoring pola desain Entity Framework Core (EF Core 9/.NET 10, C# 14).
  Mencakup konfigurasi IEntityTypeConfiguration terpisah, optimasi AsNoTracking & proyeksi DTO, eliminasi N+1 query,
  audit trail interceptor otomatis, bulk update/delete via SQL native, paginasi standar, dan ceklis evaluasi.
  Gunakan saat merancang persistence layer, menulis repository/service database, mengoptimalkan query lambat, atau mereview kode EF Core.
---

# EF Core Design Patterns & Best Practices

Panduan standar arsitektur dan pola implementasi **Entity Framework Core (EF Core 9 / .NET 10 & C# 14)** untuk aplikasi backend modern yang berkinerja tinggi, bersih, dan mudah dipelihara.

---

## 1. When to Use This Skill

### ✅ Trigger Aktivasi (Gunakan skill ini jika):
- Merancang entitas model database, DbContext, atau repository di .NET.
- Menulis query pembacaan data (GET) atau mutasi data (POST/PUT/DELETE) menggunakan EF Core.
- Merefaktor query lambat, memperbaiki masalah *N+1 query*, atau menghilangkan *tracking leakage*.
- Mengimplementasikan otomatisasi kolom audit (`CreatedAt`, `UpdatedAt`) atau soft-delete.
- Mengimplementasikan operasi update atau delete masal (*bulk operations*).
- Melakukan code review atau audit kepatuhan terhadap best practices database EF Core.

### ❌ Batasan (Jangan gunakan skill ini jika):
- Proyek menggunakan Micro-ORM murni (Dapper murni tanpa EF Core).
- Proyek menggunakan database NoSQL tanpa EF Core Provider (MongoDB native driver, Cosmos SDK native).
- Proyek berbasis arsitektur Event Sourcing tingkat lanjut atau distributed ledger.

---

## 2. Core Guardrails (Aturan Mutlak AI)

Saat menulis atau memodifikasi kode EF Core, agen AI **WAJIB** mematuhi 6 aturan mutlak ini:

1. **Wajib `.AsNoTracking()` pada Operasi Read**: Semua query pembacaan (GET/query) wajib menggunakan `.AsNoTracking()` atau diproyeksikan langsung ke DTO melalui `.Select()`.
2. **Pisahkan Konfigurasi Entitas**: Dilarang menggunakan Data Annotations database (`[Table]`, `[Column]`, `[ForeignKey]`) di dalam model entitas. Gunakan class `IEntityTypeConfiguration<T>` terpisah.
3. **Dilarang Query di Dalam Perulangan (N+1)**: Operasi pengambilan data berelasi wajib diselesaikan via eager loading (`.Include()`) atau proyeksi `.Select()`. Dilarang memanggil database di dalam loop `foreach`/`for`.
4. **Wajib Teruskan `CancellationToken`**: Setiap pemanggilan method async EF Core (`ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync`, `ExecuteUpdateAsync`, dll.) wajib menerima dan meneruskan parameter `CancellationToken cancellationToken = default`.
5. **Gunakan Bulk Operations untuk Mutasi Banyak Baris**: Dilarang me-load puluhan/ratusan baris data ke memori hanya untuk mengubah kolom atau menghapus data. Wajib gunakan `ExecuteUpdateAsync()` atau `ExecuteDeleteAsync()`.
6. **Wajib Abstraksi `TimeProvider`**: Dilarang menggunakan `DateTime.UtcNow` secara statis pada interceptor atau domain logic. Gunakan dependensi `TimeProvider` yang di-inject.

---

## 3. Pola Desain Standar EF Core (Implementasi & Kode)

### Pola 1: Konfigurasi Entitas Terpisah (`IEntityTypeConfiguration<T>`)

Memisahkan konfigurasi skema, relasi, dan indeks ke file terpisah agar model entitas tetap bersih.

```csharp
// 1. Model Entitas Bersih
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

// 2. Class Konfigurasi Terpisah (Data/Configurations/ProductConfiguration.cs)
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

        // Relasi One-to-Many eksplisit
        builder.HasOne(p => p.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId)
            .OnDelete(DeleteBehavior.Restrict);

        // Indeks untuk kolom yang sering dicari
        builder.HasIndex(p => p.CategoryId);
    }
}

// 3. Registrasi Otomatis di DbContext
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        // Otomatis meregistrasikan seluruh IEntityTypeConfiguration dalam assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

---

### Pola 2: Read vs Write Separation (Optimasi Query)

Membedakan secara tegas alur data baca (read) dan ubah (write) untuk efisiensi memori ChangeTracker.

#### A. Read-Only Query (AsNoTracking & DTO Projection)
```csharp
public async Task<List<ProductResponse>> GetActiveProductsAsync(CancellationToken cancellationToken)
{
    // Hanya membaca kolom yang dibutuhkan dan langsung di-JOIN di server database
    return await context.Products
        .AsNoTracking()
        .Where(p => p.IsActive)
        .OrderBy(p => p.Name)
        .Select(p => new ProductResponse(
            p.Id,
            p.Name,
            p.Price,
            p.Category.Name // Otomatis SQL LEFT JOIN
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

### Pola 3: Pencegahan N+1 Query Problem

```csharp
// ❌ ANTI-PATTERN: N+1 Query di dalam loop
var products = await context.Products.ToListAsync(cancellationToken);
foreach (var p in products)
{
    // Dieksekusi N kali roundtrip ke database!
    var category = await context.Categories.FirstOrDefaultAsync(c => c.Id == p.CategoryId, cancellationToken);
}

// ✅ SOLUSI 1: Eager Loading dengan .Include()
var products = await context.Products
    .AsNoTracking()
    .Include(p => p.Category)
    .ToListAsync(cancellationToken);

// ✅ SOLUSI 2: Proyeksi .Select() (Paling Hemat Memori & Bandwidth)
var productSummaries = await context.Products
    .AsNoTracking()
    .Select(p => new ProductSummary(p.Id, p.Name, p.Category.Name))
    .ToListAsync(cancellationToken);
```

---

### Pola 4: Specific Repository / Service Langsung

Hindari abstraksi `IRepository<T>` generik yang membocorkan `IQueryable`. Buat repository spesifik yang mengekspresikan kebutuhan bisnis, atau inject `AppDbContext` langsung ke Service layer.

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

### Pola 5: Audit Trail Otomatis via `SaveChangesInterceptor`

Mengisi kolom `CreatedAt` dan `UpdatedAt` secara terpusat tanpa penulisan manual di Controller/Service.

```csharp
// 1. Interface Penanda
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

// 3. Konfigurasi Dependency Injection di Program.cs
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddSingleton<AuditInterceptor>();
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"))
           .AddInterceptors(sp.GetRequiredService<AuditInterceptor>());
});
```

---

### Pola 6: Bulk Update & Delete Masal (`ExecuteUpdateAsync` / `ExecuteDeleteAsync`)

Mengeksekusi perintah SQL `UPDATE` atau `DELETE` langsung ke database tanpa memuat ribuan baris ke RAM.

```csharp
var now = timeProvider.GetUtcNow();

// Bulk Update langsung di database
int updatedRows = await context.Products
    .Where(p => p.CategoryId == categoryId && p.IsActive)
    .ExecuteUpdateAsync(setter => setter
        .SetProperty(p => p.Price, p => p.Price * 1.10m)
        .SetProperty(p => p.UpdatedAt, now),
        cancellationToken);

// Bulk Delete langsung di database
int deletedRows = await context.Products
    .Where(p => !p.IsActive && p.CreatedAt < cutOffDate)
    .ExecuteDeleteAsync(cancellationToken);
```

---

### Pola 7: Paginasi Standar (Offset / PagedResult)

Formula paginasi efisien yang membatasi pengambilan record di level database server:

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

## 4. Anti-Patterns & Solusi Refactoring

| Anti-Pattern | Dampak Negatif | Solusi Terstandarisasi |
| :--- | :--- | :--- |
| **Generic `IRepository<T>`** | Leaky abstraction, query filter tersebar | Gunakan **Specific Repository** atau DbContext langsung di service layer. |
| **N+1 Query Problem** | Puluhan/ratusan roundtrip ke database | Gunakan **`.Include()`** atau proyeksi **`.Select()`**. |
| **In-Memory Filtering** | Memuat jutaan record ke RAM (`.ToList()` lalu `.Where()`) | Letakkan filter **`.Where()` sebelum `.ToListAsync()`**. |
| **Missing `AsNoTracking`** | Memory bloat pada operasi GET | Tambahkan **`.AsNoTracking()`** pada semua query baca. |
| **Looping `SaveChanges`** | Transaksi database berulang dan lambat | Gunakan **`ExecuteUpdateAsync`** atau satu kali **`SaveChangesAsync`**. |
| **`CountAsync() > 0`** | Menghitung seluruh data yang cocok (Full Scan) | Gunakan **`.AnyAsync()`** (menghasilkan SQL `EXISTS`). |
| **Hardcoded `DateTime.UtcNow`** | Sulit diuji secara deterministik | Inject abstraksi **`TimeProvider`**. |

---

## 5. Ceklis Evaluasi & Verifikasi (Checklist)

Gunakan ceklis ini saat membuat fitur baru, merefaktor query database, atau melakukan pull request (PR) review:

### A. Model & Konfigurasi Tabel
- [ ] **Konfigurasi Terpisah**: File konfigurasi dipisah per model menggunakan `IEntityTypeConfiguration<T>`, bukan menumpuk atribut `[Table]` / `[Column]` di class entitas.
- [ ] **Auto Apply Configuration**: Di `AppDbContext.cs`, sudah dipasang `modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);`.
- [ ] **Panjang String & Presisi Desimal**:
  - Kolom teks memiliki batas panjang: `.HasMaxLength(...)`.
  - Kolom desimal (harga, uang) memiliki presisi: `.HasPrecision(18, 2)`.
- [ ] **Index pada Foreign Key**: Kolom foreign key atau kolom yang sering difilter diberi `.HasIndex(...)`.

### B. Query Pembacaan Data (GET)
- [ ] **Gunakan `.AsNoTracking()`**: Semua query yang hanya untuk dibaca/ditampilkan menyertakan `.AsNoTracking()`.
- [ ] **Proyeksi DTO (`.Select()`)**: Hindari me-return langsung model entitas database ke Controller; gunakan `.Select(x => new MyDto(...))` untuk mengambil kolom yang dibutuhkan saja.
- [ ] **Bebas dari N+1 Query**:
  - Tidak ada query database yang dipanggil di dalam loop `foreach`/`for`.
  - Menggunakan `.Include()` untuk relasi atau otomatis via `.Select()`.
- [ ] **Cek Keberadaan Efisien**: Menggunakan `.AnyAsync(...)` alih-alih `CountAsync() > 0` atau `FirstOrDefaultAsync() != null`.
- [ ] **Paginasi di Database**: Paginasi `.Skip().Take()` dijalankan pada `IQueryable` sebelum `.ToListAsync()`.

### C. Tambah, Update & Hapus (POST, PUT, DELETE)
- [ ] **Tracked Entity untuk Update Tunggal**: Entity diambil dari DbContext, nilainya diubah, lalu disimpan dengan `await context.SaveChangesAsync(cancellationToken)`.
- [ ] **Bulk Update / Bulk Delete**: Jika mengubah atau menghapus banyak baris sekaligus, gunakan `ExecuteUpdateAsync()` atau `ExecuteDeleteAsync()` alih-alih me-load semua ke memori.
- [ ] **Satu Kali `SaveChangesAsync`**: Operasi yang harus berhasil bersamaan disimpan dalam satu kali pemanggilan `SaveChangesAsync()` agar otomatis berada dalam 1 transaksi.

### D. Performa & Higienitas Kode
- [ ] **Wajib `CancellationToken`**: Setiap pemanggilan async (`ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync`, dll.) meneruskan parameter `cancellationToken`.
- [ ] **Audit Trail Otomatis**: Kolom `CreatedAt` dan `UpdatedAt` diisi otomatis melalui `SaveChangesInterceptor` (tidak diisi manual di controller/service).
- [ ] **Gunakan `TimeProvider`**: Pengambilan waktu saat ini menggunakan `TimeProvider.System.GetUtcNow()` agar mudah diuji dengan unit test.
- [ ] **Build Bersih**: Kode lolos build tanpa warning (`dotnet build`).

---

## 6. Definition of Done & Verification

Sebelum menyelesaikan tugas implementasi atau review EF Core, jalankan verifikasi berikut di terminal:

```bash
# 1. Pastikan solusi ter-build bersih tanpa error dan warning
dotnet build --configuration Release

# 2. Pastikan tidak ada model change yang tertinggal dari migrasi
dotnet ef migrations has-pending-model-changes

# 3. Jalankan pengujian unit dan integrasi
dotnet test
```
