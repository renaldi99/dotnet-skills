# Module scaffolding: Employees

Baca untuk modul baru atau contoh layering, bukan untuk setiap bug fix.
Contoh memakai Foundation, SQL Server, dan implicit usings/nullable enabled.
Setiap heading path adalah file terpisah; jangan menempelkan seluruh dokumen ke satu file.
Nama, kebijakan email, batas halaman, dan kontrak berikut adalah contoh untuk proyek baru.
Jangan mengganti kontrak existing dengan contoh ini.

## Modules/Employees/Dtos/EmployeeDtos.cs

Validasi attribute pada positional record diletakkan pada parameter constructor.
Jika proyek memakai FluentValidation, gunakan mekanisme itu secara konsisten.

```csharp
using System.ComponentModel.DataAnnotations;

namespace MyService.Api.Modules.Employees.Dtos;

public sealed record CreateEmployeeRequest(
    [Required, StringLength(100)] string FirstName,
    [Required, StringLength(100)] string LastName,
    [Required, EmailAddress, StringLength(254)] string Email,
    [Required, StringLength(100)] string Position,
    decimal Salary, Guid DepartmentId);

public sealed record EmployeeResponse(
    Guid Id, string FullName, string Email, string Position, decimal Salary,
    Guid DepartmentId, bool IsActive, DateTimeOffset CreatedAt);

public sealed class PageRequest
{
    [Range(1, int.MaxValue)]
    public int Page { get; init; } = 1;
    [Range(1, 100)]
    public int PageSize { get; init; } = 20;
}
```

## Modules/Employees/Entities/Employee.cs

```csharp
namespace MyService.Api.Modules.Employees.Entities;

public sealed class Employee
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public required string FirstName { get; set; }
    public required string LastName { get; set; }
    public required string Email { get; set; }
    public required string Position { get; set; }
    public decimal Salary { get; set; }
    public Guid DepartmentId { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? UpdatedAt { get; set; }
}
```

## Modules/Employees/Entities/EmployeeConfiguration.cs

Email disimpan dalam bentuk normalisasi dan unik termasuk baris soft-deleted.
Jika reuse email setelah delete diperlukan, ubah kebijakan/constraint secara eksplisit.
`DepartmentId` di sini hanya identifier; tambahkan FK atau validasi lintas modul sesuai model nyata.

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace MyService.Api.Modules.Employees.Entities;

public sealed class EmployeeConfiguration : IEntityTypeConfiguration<Employee>
{
    public void Configure(EntityTypeBuilder<Employee> builder)
    {
        builder.ToTable("Employees");
        builder.HasKey(employee => employee.Id);
        builder.Property(employee => employee.FirstName).HasMaxLength(100).IsRequired();
        builder.Property(employee => employee.LastName).HasMaxLength(100).IsRequired();
        builder.Property(employee => employee.Email).HasMaxLength(254).IsRequired();
        builder.Property(employee => employee.Position).HasMaxLength(100).IsRequired();
        builder.Property(employee => employee.Salary).HasPrecision(18, 2);
        builder.HasIndex(employee => employee.Email).IsUnique().HasDatabaseName("UX_Employees_Email");
        builder.HasIndex(employee => new { employee.IsActive, employee.CreatedAt, employee.Id });
    }
}
```

## Data/AppDbContext.cs

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyService.Api.Data;

public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

## Modules/Employees/Repositories/EmployeeRepository.cs

Read-only query tidak tracked; load untuk mutasi memiliki nama terpisah.
`Add` hanya melakukan tracking secara sinkron. Service menentukan kapan `SaveChangesAsync`
dipanggil; tidak perlu `Task.CompletedTask` atau method async palsu untuk `Update`.
Pagination divalidasi juga di repository agar caller non-HTTP tidak melewati batas.

```csharp
using Microsoft.EntityFrameworkCore;
using MyService.Api.Common.Exceptions;
using MyService.Api.Common.Models;
using MyService.Api.Data;
using MyService.Api.Modules.Employees.Dtos;
using MyService.Api.Modules.Employees.Entities;

namespace MyService.Api.Modules.Employees.Repositories;

public interface IEmployeeRepository
{
    Task<PagedResult<EmployeeResponse>> GetPageAsync(int page, int pageSize, CancellationToken cancellationToken = default);
    Task<Employee?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<Employee?> GetForUpdateAsync(Guid id, CancellationToken cancellationToken = default);
    Task<bool> ExistsByEmailAsync(string normalizedEmail, CancellationToken cancellationToken = default);
    void Add(Employee employee);
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}

public sealed class EmployeeRepository(AppDbContext context) : IEmployeeRepository
{
    public async Task<PagedResult<EmployeeResponse>> GetPageAsync(
        int page, int pageSize, CancellationToken cancellationToken = default)
    {
        if (page < 1 || pageSize is < 1 or > 100 || (long)(page - 1) * pageSize > int.MaxValue)
            throw new ValidationException(new { Pagination = "Page/pageSize di luar batas." });

        var query = context.Set<Employee>().AsNoTracking().Where(employee => employee.IsActive);
        var total = await query.CountAsync(cancellationToken);
        var items = await query.OrderByDescending(employee => employee.CreatedAt)
            .ThenBy(employee => employee.Id)
            .Skip((page - 1) * pageSize).Take(pageSize)
            .Select(employee => new EmployeeResponse(
                employee.Id, employee.FirstName + " " + employee.LastName, employee.Email,
                employee.Position, employee.Salary, employee.DepartmentId, employee.IsActive, employee.CreatedAt))
            .ToListAsync(cancellationToken);
        return new(items, page, pageSize, total);
    }

    public Task<Employee?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default) =>
        context.Set<Employee>().AsNoTracking()
            .SingleOrDefaultAsync(employee => employee.Id == id && employee.IsActive, cancellationToken);

    public Task<Employee?> GetForUpdateAsync(Guid id, CancellationToken cancellationToken = default) =>
        context.Set<Employee>().AsTracking()
            .SingleOrDefaultAsync(employee => employee.Id == id && employee.IsActive, cancellationToken);

    public Task<bool> ExistsByEmailAsync(string normalizedEmail, CancellationToken cancellationToken = default) =>
        context.Set<Employee>().AnyAsync(employee => employee.Email == normalizedEmail, cancellationToken);

    public void Add(Employee employee) => context.Set<Employee>().Add(employee);

    public Task<int> SaveChangesAsync(CancellationToken cancellationToken = default) =>
        context.SaveChangesAsync(cancellationToken);
}
```

## Modules/Employees/Services/EmployeeService.cs

Boundary HTTP memvalidasi DTO sebelum service dipanggil; caller non-HTTP juga wajib
melakukan validasi input. Service tetap memvalidasi invariant bisnis.
Pre-check email hanya memperbaiki pesan error, **bukan pengganti unique constraint**.

```csharp
using MyService.Api.Common.Errors;
using MyService.Api.Common.Exceptions;
using MyService.Api.Common.Models;
using MyService.Api.Modules.Employees.Dtos;
using MyService.Api.Modules.Employees.Entities;
using MyService.Api.Modules.Employees.Repositories;

namespace MyService.Api.Modules.Employees.Services;

public interface IEmployeeService
{
    Task<PagedResult<EmployeeResponse>> GetPageAsync(int page, int pageSize, CancellationToken cancellationToken = default);
    Task<EmployeeResponse> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<EmployeeResponse> CreateAsync(CreateEmployeeRequest request, CancellationToken cancellationToken = default);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken = default);
}

public sealed class EmployeeService(
    IEmployeeRepository repository, ILogger<EmployeeService> logger, TimeProvider timeProvider) : IEmployeeService
{
    public Task<PagedResult<EmployeeResponse>> GetPageAsync(
        int page, int pageSize, CancellationToken cancellationToken = default) =>
        repository.GetPageAsync(page, pageSize, cancellationToken);

    public async Task<EmployeeResponse> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        var employee = await repository.GetByIdAsync(id, cancellationToken)
            ?? throw new NotFoundException(ErrorCode.EntityNotFound);
        return Map(employee);
    }

    public async Task<EmployeeResponse> CreateAsync(
        CreateEmployeeRequest request, CancellationToken cancellationToken = default)
    {
        if (request.Salary <= 0 || request.DepartmentId == Guid.Empty)
            throw new AppException(ErrorCode.InvalidBusinessRule, "Gaji dan departemen harus valid.");

        var email = request.Email.Trim().ToUpperInvariant();
        if (await repository.ExistsByEmailAsync(email, cancellationToken))
            throw new ConflictException(ErrorCode.EntityAlreadyExists, "Email sudah terdaftar.");

        var employee = new Employee
        {
            FirstName = request.FirstName.Trim(), LastName = request.LastName.Trim(),
            Email = email, Position = request.Position.Trim(), Salary = request.Salary,
            DepartmentId = request.DepartmentId, CreatedAt = timeProvider.GetUtcNow()
        };
        repository.Add(employee);
        await repository.SaveChangesAsync(cancellationToken);
        logger.LogInformation("Employee created: {Id}", employee.Id);
        return Map(employee);
    }

    public async Task DeleteAsync(Guid id, CancellationToken cancellationToken = default)
    {
        var employee = await repository.GetForUpdateAsync(id, cancellationToken)
            ?? throw new NotFoundException(ErrorCode.EntityNotFound);
        employee.IsActive = false;
        employee.UpdatedAt = timeProvider.GetUtcNow();
        await repository.SaveChangesAsync(cancellationToken);
    }

    private static EmployeeResponse Map(Employee employee) =>
        new(employee.Id, employee.FirstName + " " + employee.LastName, employee.Email,
            employee.Position, employee.Salary, employee.DepartmentId, employee.IsActive, employee.CreatedAt);
}
```

## Modules/Employees/Controllers/EmployeesController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using MyService.Api.Common.Models;
using MyService.Api.Modules.Employees.Dtos;
using MyService.Api.Modules.Employees.Services;

namespace MyService.Api.Modules.Employees.Controllers;

[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
public sealed class EmployeesController(IEmployeeService service, TimeProvider timeProvider) : ControllerBase
{
    [HttpGet]
    [ProducesResponseType(typeof(ApiResponse<PagedResult<EmployeeResponse>>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ApiResponse), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> GetPage([FromQuery] PageRequest query, CancellationToken cancellationToken)
    {
        var result = await service.GetPageAsync(query.Page, query.PageSize, cancellationToken);
        return Ok(ApiResponse<PagedResult<EmployeeResponse>>.SuccessResult(result, timeProvider));
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<EmployeeResponse>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ApiResponse), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(Guid id, CancellationToken cancellationToken)
    {
        var result = await service.GetByIdAsync(id, cancellationToken);
        return Ok(ApiResponse<EmployeeResponse>.SuccessResult(result, timeProvider));
    }

    [HttpPost]
    [ProducesResponseType(typeof(ApiResponse<EmployeeResponse>), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ApiResponse), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ApiResponse), StatusCodes.Status409Conflict)]
    public async Task<IActionResult> Create(CreateEmployeeRequest request, CancellationToken cancellationToken)
    {
        var result = await service.CreateAsync(request, cancellationToken);
        return CreatedAtAction(nameof(GetById), new { id = result.Id },
            ApiResponse<EmployeeResponse>.SuccessResult(result, timeProvider));
    }

    [HttpDelete("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ApiResponse), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(Guid id, CancellationToken cancellationToken)
    {
        await service.DeleteAsync(id, cancellationToken);
        return Ok(ApiResponse.SuccessResult(timeProvider));
    }
}
```

## Modules/Employees/EmployeesModuleExtensions.cs

```csharp
using MyService.Api.Modules.Employees.Repositories;
using MyService.Api.Modules.Employees.Services;

namespace MyService.Api.Modules.Employees;

public static class EmployeesModuleExtensions
{
    public static IServiceCollection AddEmployeesModule(this IServiceCollection services)
    {
        services.AddScoped<IEmployeeRepository, EmployeeRepository>();
        services.AddScoped<IEmployeeService, EmployeeService>();
        return services;
    }
}
```

## Sebelum memakai contoh pada fitur nyata

- Tambahkan migration untuk constraint/index. Jangan menganggap index sudah ada hanya karena configuration ditulis.
- Pada boundary persistence, terjemahkan **pelanggaran unique email yang teridentifikasi**
  menjadi `ConflictException`, termasuk insert bersamaan. Gunakan adapter/provider error
  metadata proyek; jangan memetakan semua `DbUpdateException` ke 409.
  Contoh dasar di atas belum menyertakan adapter provider ini: tanpa adapter, race ditolak DB
  tetapi menjadi 500. Lengkapi pemetaan dan integration test sebelum menyatakan create selesai.
- Periksa precision/range salary, normalisasi email, ownership/akses Department, dan aturan
  soft-delete sesuai domain. Jangan mengasumsikan identifier yang valid berarti user boleh mengaksesnya.
- Jika mutasi memerlukan optimistic concurrency, tambahkan concurrency token dan pemetaan
  `DbUpdateConcurrencyException` yang sesuai kontrak. Contoh belum menetapkan kebijakan konflik edit.
- Count dan page adalah dua query; tidak menjamin snapshot yang sama saat concurrent write.
  Gunakan isolation/kontrak yang sesuai jika konsistensi snapshot diperlukan.
- Uji invalid DTO/JSON, halaman nol/terlalu besar/overflow, not found, timestamp terkontrol,
  token diteruskan, dan duplicate email concurrent dengan provider relational yang sebenarnya.
