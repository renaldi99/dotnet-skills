# Database Performance Reference (.NET 10 / EF Core & Dapper)

Petunjuk teknis dan aturan penulisan query database performa tinggi untuk agen AI. Buka dokumen ini saat merancang query repository, menangani data berelasi, melakukan optimasi pembacaan, atau menangani operasi masal (bulk).

---

## 1. Quick Decision Matrix (Diagnosa & Solusi Cepat)

| Gejala / Kebutuhan | Penyebab / Anti-Pattern ❌ | Solusi Wajib ✅ |
| :--- | :--- | :--- |
| Query dieksekusi berulang kali dalam loop | N+1 Query Problem | **EF Core:** `.Include()` atau proyeksi `.Select()`.<br>**Dapper:** `QueryMultipleAsync` atau Multi-Mapping. |
| Dua tabel di-load ke RAM lalu di-join dengan LINQ | In-Memory Join (`.ToList()` lalu `.Join()`) | Lakukan **SQL JOIN di database engine** via navigation property atau `.Join()` sebelum materialisasi. |
| Memory bloat pada operasi baca | ChangeTracker menyimpan snapshot data | Tambahkan **`.AsNoTracking()`** pada semua operasi query GET. |
| Mengambil seluruh kolom padahal hanya butuh 2-3 field | Over-fetching (`SELECT *`) | Gunakan **Proyeksi `.Select(x => new Dto(...))`**. |
| Query lambat saat `.Include()` lebih dari 1 collection | Cartesian Explosion ($M \times N$ baris) | Tambahkan **`.AsSplitQuery()`**. |
| Update 10.000 data lambat & boros RAM | Load ke RAM $\rightarrow$ loop update $\rightarrow$ SaveChanges | Gunakan **`ExecuteUpdateAsync()`** / **`ExecuteDeleteAsync()`** (.NET 10). |
| Mengecek apakah data ada terasa lambat | `CountAsync() > 0` (Full Scan) | Gunakan **`.AnyAsync()`** (menghasilkan SQL `IF EXISTS`). |
| Paginasi lambat pada tabel besar | `ToList()` dulu baru `.Skip().Take()` | Pasang **`.Skip().Take()` pada `IQueryable`** sebelum materialisasi. |

---

## 2. Aturan Abstraksi: Specific Repository Only

DILARANG membuat `GenericRepository<T>` atau `IRepository<T>` karena `DbSet<T>` sudah merupakan repository, dan abstraksi generik membocorkan `IQueryable` serta menghalangi optimasi kustom.

```csharp
// ❌ DILARANG: Generic Repository
public interface IRepository<T> { IQueryable<T> Query(); }

// ✅ WAJIB: Specific Repository Interface per Modul
public interface IEmployeeRepository
{
    Task<IReadOnlyList<EmployeeResponse>> GetActiveSummariesAsync(Guid? deptId, CancellationToken cancellationToken = default);
    Task<Employee?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<bool> ExistsByEmailAsync(string email, Guid? excludeId = null, CancellationToken cancellationToken = default);
    Task AddAsync(Employee employee, CancellationToken cancellationToken = default);
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

---

## 3. Pencegahan N+1 Query Problem

### A. EF Core 10: Eager Loading vs Proyeksi

```csharp
// ❌ ANTI-PATTERN: N+1 Query di dalam loop
var employees = await context.Employees.ToListAsync(cancellationToken);
foreach (var emp in employees)
{
    // Dieksekusi N kali roundtrip ke database!
    var dept = await context.Departments.FirstOrDefaultAsync(d => d.Id == emp.DepartmentId, cancellationToken);
}

// ✅ SOLUSI 1: Eager Loading dengan .Include()
var result = await context.Employees
    .AsNoTracking()
    .Include(e => e.Department)
    .Where(e => e.IsActive)
    .ToListAsync(cancellationToken);

// ✅ SOLUSI 2: Proyeksi .Select() (Paling Optimal - Otomatis JOIN & hemat kolom)
var result = await context.Employees
    .AsNoTracking()
    .Where(e => e.IsActive)
    .Select(e => new EmployeeResponse(
        e.Id,
        $"{e.FirstName} {e.LastName}",
        e.Email,
        e.Position,
        e.Department.Name, // Otomatis SQL LEFT JOIN
        e.Salary
    ))
    .ToListAsync(cancellationToken);
```

### B. Dapper: Multi-Mapping & Batch Query

```csharp
// ✅ SOLUSI 1: Single Query dengan Multi-Mapping (One-to-One)
const string sql = """
    SELECT e.Id, e.FirstName, e.LastName, e.Email, e.Position, e.Salary,
           d.Id AS DepartmentId, d.Name, d.Code
    FROM Employees e
    INNER JOIN Departments d ON e.DepartmentId = d.Id
    WHERE e.IsActive = 1
    """;

var result = await connection.QueryAsync<Employee, Department, EmployeeResponse>(
    new CommandDefinition(sql, cancellationToken: cancellationToken),
    (emp, dept) => new EmployeeResponse(emp.Id, $"{emp.FirstName} {emp.LastName}", emp.Email, emp.Position, dept.Name, emp.Salary),
    splitOn: "DepartmentId"
);

// ✅ SOLUSI 2: Batch Query Multiple (One-to-Many tanpa perkalian baris)
const string batchSql = """
    SELECT Id, Name FROM Departments WHERE Id = @Id;
    SELECT Id, FirstName, LastName, Position FROM Employees WHERE DepartmentId = @Id AND IsActive = 1;
    """;

using var multi = await connection.QueryMultipleAsync(
    new CommandDefinition(batchSql, new { Id = departmentId }, cancellationToken: cancellationToken));

var dept = await multi.ReadFirstOrDefaultAsync<Department>();
var emps = (await multi.ReadAsync<Employee>()).ToList();
```

---

## 4. Anti-Pattern: In-Memory Join

```csharp
// ❌ SANGAT BURUK: Menarik seluruh tabel ke RAM lalu di-join di memori C#
var emps = await context.Employees.ToListAsync(cancellationToken);
var depts = await context.Departments.ToListAsync(cancellationToken);
var joined = emps.Join(depts, e => e.DepartmentId, d => d.Id, (e, d) => new { e, d }).ToList();

// ✅ BENAR: Eksekusi SQL JOIN langsung di Database Engine
var joined = await context.Employees
    .AsNoTracking()
    .Where(e => e.IsActive)
    .Join(
        context.Departments,
        e => e.DepartmentId,
        d => d.Id,
        (e, d) => new EmployeeResponse(e.Id, $"{e.FirstName} {e.LastName}", e.Email, e.Position, d.Name, e.Salary)
    )
    .ToListAsync(cancellationToken);
```

---

## 5. Mencegah Cartesian Explosion (.AsSplitQuery)

Ketika me-load lebih dari satu relasi koleksi (`One-to-Many`), SQL JOIN biasa akan melipatgandakan data ($M \times N$). Gunakan `.AsSplitQuery()`:

```csharp
// ✅ Pisahkan menjadi query independen yang digabungkan aman di memori
var department = await context.Departments
    .AsNoTracking()
    .Include(d => d.Employees)
    .Include(d => d.Projects)
    .AsSplitQuery() // Mencegah Cartesian Product
    .FirstOrDefaultAsync(d => d.Id == id, cancellationToken);
```

---

## 6. Bulk Operations (.NET 10 ExecuteUpdate & ExecuteDelete)

Dilarang menarik ribuan entitas ke memori hanya untuk memodifikasi satu kolom. Gunakan direct SQL execution:

```csharp
// ✅ UPDATE MASAL: 1 statement SQL langsung dieksekusi tanpa change tracking
await context.Employees
    .Where(e => e.DepartmentId == targetDeptId)
    .ExecuteUpdateAsync(s => s
        .SetProperty(e => e.IsActive, false)
        .SetProperty(e => e.UpdatedAt, timeProvider.GetUtcNow()), 
        cancellationToken);

// ✅ DELETE MASAL: 1 statement SQL langsung
await context.AuditLogs
    .Where(log => log.CreatedAt < retentionDate)
    .ExecuteDeleteAsync(cancellationToken);
```

---

## 7. Existence Check & Paginasi Efisien

```csharp
// ❌ BURUK: Full scan atau alokasi object
var exists = await context.Employees.CountAsync(e => e.Email == email, cancellationToken) > 0;
var exists = await context.Employees.FirstOrDefaultAsync(e => e.Email == email, cancellationToken) != null;

// ✅ CEPAT: Menghasilkan SQL 'IF EXISTS (SELECT 1 ...)'
var exists = await context.Employees.AnyAsync(e => e.Email == email, cancellationToken);

// ✅ PAGINASI BENAR: Skip & Take sebelum ToListAsync
public async Task<IReadOnlyList<EmployeeResponse>> GetPagedAsync(int page, int pageSize, CancellationToken cancellationToken)
{
    return await context.Employees
        .AsNoTracking()
        .Where(e => e.IsActive)
        .OrderByDescending(e => e.CreatedAt)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(e => new EmployeeResponse(e.Id, $"{e.FirstName} {e.LastName}", e.Email, e.Position, e.Salary, e.DepartmentId, e.IsActive, e.CreatedAt))
        .ToListAsync(cancellationToken);
}
```
