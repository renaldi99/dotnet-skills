# Database performance

Baca hanya saat mengubah repository/query. G4-G5 di SKILL.md adalah sumber aturan;
contoh berikut adalah fragmen untuk diadaptasi pada entity/provider proyek, bukan file aplikasi lengkap.

## Keputusan query

| Kebutuhan | Pilihan |
| :--- | :--- |
| Read-only entity | `AsNoTracking()`; proyeksi DTO/scalar tanpa entity juga tidak tracked. |
| Load untuk mutasi | Query tracked yang eksplisit, ubah field yang diperlukan, lalu `SaveChangesAsync(token)`. Jangan `Update` seluruh detached graph tanpa kebutuhan. |
| Relasi/N+1 | Proyeksi `Select`, SQL join, atau eager loading. Jangan query per item dalam loop. |
| Beberapa sibling collection | Pertimbangkan `AsSplitQuery()` setelah menilai cartesian explosion, roundtrip, dan konsistensi; bukan default untuk setiap Include. |
| Existence | `AnyAsync(token)`, bukan `CountAsync() > 0`. Bentuk SQL bergantung provider; tidak selalu literal `IF EXISTS`. |
| List | Filter/order/pagination sebelum materialisasi, batas page size, dan urutan unik. Gunakan keyset/cursor untuk traversal besar bila sesuai kontrak. |
| Bulk update/delete | `ExecuteUpdateAsync`/`ExecuteDeleteAsync` bila invariant, audit, concurrency, dan transaksi tetap terpenuhi. Fitur ini tersedia sejak EF Core 7, bukan khusus .NET 10. |

## Join di database dan DTO projection

Misalnya model memiliki `Employee.DepartmentId` dan `Department.Id/Name`.
Gunakan DTO khusus summary, jangan memanggil constructor `EmployeeResponse` dengan bentuk berbeda.
Contoh menggunakan `offset` dan `pageSize` yang sudah divalidasi seperti bagian pagination.

```csharp
public sealed record EmployeeDepartmentSummary(
    Guid Id, string FullName, string Email, string Position, string DepartmentName, decimal Salary);
```

```csharp
var items = await context.Set<Employee>()
    .AsNoTracking()
    .Where(employee => employee.IsActive)
    .Join(context.Set<Department>(), employee => employee.DepartmentId, department => department.Id,
        (employee, department) => new { Employee = employee, Department = department })
    .OrderBy(row => row.Employee.Id)
    .Skip(offset).Take(pageSize)
    .Select(row => new EmployeeDepartmentSummary(
        row.Employee.Id, row.Employee.FirstName + " " + row.Employee.LastName,
        row.Employee.Email, row.Employee.Position, row.Department.Name, row.Employee.Salary))
    .ToListAsync(cancellationToken);
```

Join ini inner join. Jangan mengasumsikan navigation projection selalu menghasilkan LEFT JOIN;
required/optional relationship dan provider menentukan terjemahan SQL.
Jangan me-load kedua tabel lalu melakukan LINQ Join di aplikasi. Materialisasi relasi oleh
EF split query atau Dapper multi-mapping bukan larangan yang sama dengan join dua tabel tanpa batas di RAM.

## Dapper: projection SQL berparameter

Gunakan DTO summary di atas. Contoh SQL Server ini mempertahankan nama kolom yang cocok
dengan constructor DTO dan token melalui `CommandDefinition`. Sesuaikan dialek untuk provider lain.

```csharp
const string sql = """
    SELECT e.Id, CONCAT(e.FirstName, ' ', e.LastName) AS FullName,
           e.Email, e.Position, d.Name AS DepartmentName, e.Salary
    FROM Employees e
    INNER JOIN Departments d ON d.Id = e.DepartmentId
    WHERE e.IsActive = 1
    ORDER BY e.Id
    OFFSET @Offset ROWS FETCH NEXT @PageSize ROWS ONLY;
    """;

var items = await connection.QueryAsync<EmployeeDepartmentSummary>(
    new CommandDefinition(sql, new { Offset = offset, PageSize = pageSize },
        cancellationToken: cancellationToken));
```

Untuk one-to-many dengan result set terpisah, pertimbangkan `QueryMultipleAsync`.
Batasi child collection, dispose `GridReader`, dan jangan menggabungkan hasil parsial
seolah-olah entity lengkap. Untuk multi-mapping, periksa `splitOn` dan kolom identity setiap tipe.

## Pagination dan uniqueness

- Tolak `page < 1`, `pageSize` di luar batas kebijakan, dan offset yang melebihi rentang integer.
  Jangan diam-diam clamp input invalid. Contoh modul memakai default 20 dan maksimum 100.
- Order harus unik: misalnya `OrderByDescending(CreatedAt).ThenBy(Id)`.
  Urutan unik tidak menghilangkan pergeseran offset ketika data berubah; gunakan cursor jika dibutuhkan.
- `CountAsync` tetap tepat untuk total count. Evaluasi biaya query count dan konsistensi snapshot
  terhadap kebutuhan endpoint; jangan menghitung total jika kontrak tidak memerlukannya.
- Normalisasi key sebelum pre-check **dan** insert/update dengan fungsi yang sama.
  Hindari `ToLower()` pada kolom query yang menghambat pemakaian index.
- Uniqueness harus dijamin constraint/index database, dengan migration dan kebijakan collation
  yang sesuai. Pre-check saja tidak aman terhadap concurrent insert.
- Terjemahkan hanya pelanggaran constraint yang dikenali menjadi konflik domain. Error koneksi,
  timeout, FK, atau constraint lain tidak boleh disamarkan sebagai duplicate email.

## Split query dan bulk operation

Split query menambah roundtrip dan dapat membaca snapshot berbeda antar-query saat data berubah.
Ukur SQL/query plan dan pertimbangkan transaksi/isolation bila konsistensi dibutuhkan.

Bulk operation berjalan langsung tanpa change tracker atau `SaveChanges` interceptor:

```csharp
var now = timeProvider.GetUtcNow();
var affected = await context.Set<Employee>()
    .Where(employee => employee.DepartmentId == targetDeptId && employee.IsActive)
    .ExecuteUpdateAsync(update => update
        .SetProperty(employee => employee.IsActive, false)
        .SetProperty(employee => employee.UpdatedAt, now), cancellationToken);
```

Tetapkan hasil bisnis berdasarkan `affected`. Tambahkan filter concurrency jika diperlukan.
Jangan mengharapkan `SaveChanges` berikutnya menerapkan bulk operation; jangan memakai entity
tracked yang sudah stale tanpa reload/clear yang disengaja. Beberapa bulk statement tidak
otomatis menjadi satu transaksi; bungkus eksplisit jika harus atomik.

## Bukti yang relevan

Untuk perubahan query, periksa SQL, jumlah roundtrip, jumlah baris/kolom, dan query plan
sesuai skala kasus. Uji translation, constraint, dan transaksi dengan provider relational
yang kompatibel; EF InMemory tidak membuktikan perilaku tersebut.
