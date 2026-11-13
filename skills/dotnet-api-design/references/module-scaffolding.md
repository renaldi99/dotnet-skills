# Module scaffolding: Employees

Read for new modules or layering examples, not for every bug fix.
The example uses Foundation, SQL Server, and implicit usings/nullable enabled.
Each heading path is a separate file; do not paste the whole document into one file.
The names, email policy, page limits, and contracts below are examples for new projects.
Do not replace existing contracts with this example.

## Project structure

Modules are organized vertically under `Modules/`, while the foundation is shared through
`Common/` and `Data/`. The diagram below follows the file paths in the example:

```text
MyService.Api/
|-- Modules/
|   |-- Employees/
|   |   |-- Controllers/
|   |   |   `-- EmployeesController.cs
|   |   |-- Services/
|   |   |   `-- EmployeeService.cs          # Includes IEmployeeService
|   |   |-- Repositories/
|   |   |   `-- EmployeeRepository.cs       # Includes IEmployeeRepository
|   |   |-- Dtos/
|   |   |   `-- EmployeeDtos.cs             # Create, Update, Response, PageRequest
|   |   |-- Entities/
|   |   |   |-- Employee.cs
|   |   |   `-- EmployeeConfiguration.cs
|   |   |-- Validators/                    # Optional, when using separate validators
|   |   `-- EmployeesModuleExtensions.cs
|   `-- {NextModule}/
|-- Common/
|   |-- Models/
|   |   |-- ApiResponse.cs
|   |   `-- PagedResult.cs
|   |-- Errors/
|   |   |-- ErrorCode.cs
|   |   `-- ErrorCodeExtensions.cs
|   |-- Exceptions/
|   |   `-- AppException.cs                # Includes domain exception subclasses
|   `-- Handlers/
|       `-- GlobalExceptionHandler.cs
|-- Data/
|   `-- AppDbContext.cs
`-- Program.cs

MyService.Api.Tests/
`-- Modules/
    `-- Employees/
        `-- EmployeeServiceTests.cs        # Unit test xUnit + NSubstitute + FakeTimeProvider
```

The `Common/` code is in `foundation.md`, and `Program.cs` is in `bootstrap.md`;
read them only when you need to create or change those parts. The `Validators/` folder
is not needed for the attribute validation example below. Interfaces and implementations
are combined in the example for brevity; follow the existing project's file-splitting convention.

## Modules/Employees/Dtos/EmployeeDtos.cs

Validation attributes on positional records go on the constructor parameters.
If the project uses FluentValidation, use that mechanism consistently.

```csharp
using System.ComponentModel.DataAnnotations;

namespace MyService.Api.Modules.Employees.Dtos;

public sealed record CreateEmployeeRequest(
    [Required, StringLength(100)] string FirstName,
    [Required, StringLength(100)] string LastName,
    [Required, EmailAddress, StringLength(254)] string Email,
    [Required, StringLength(100)] string Position,
    decimal Salary, Guid DepartmentId);

public sealed record UpdateEmployeeRequest(
    [Required, StringLength(100)] string FirstName,
    [Required, StringLength(100)] string LastName,
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

Email is stored normalized and is unique, including soft-deleted rows.
If email reuse after deletion is required, change the policy/constraint explicitly.
`DepartmentId` here is only an identifier; add an FK or cross-module validation according to the real model.

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

Read-only queries are not tracked; loads for mutation have a separate name.
`Add` only starts tracking synchronously. The service decides when `SaveChangesAsync`
is called; no `Task.CompletedTask` or fake async method is needed for `Update`.
Pagination is also validated in the repository so non-HTTP callers cannot exceed the limits.
`SaveChangesAsync` is the persistence boundary: only recognized `UX_Employees_Email`
index violations are translated into `ConflictException`; other database errors stay 500.

```csharp
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using MyService.Api.Common.Errors;
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
            throw new ValidationException(new { Pagination = "Page/pageSize out of range." });

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

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        try
        {
            return await context.SaveChangesAsync(cancellationToken);
        }
        catch (DbUpdateException exception) when (IsDuplicateEmail(exception))
        {
            throw new ConflictException(ErrorCode.EntityAlreadyExists, "Email is already registered.");
        }
    }

    // SQL Server: 2601 = duplicate key in a unique index, 2627 = unique constraint.
    // Other providers expose different error metadata; adapt this, do not map every DbUpdateException.
    private static bool IsDuplicateEmail(DbUpdateException exception) =>
        exception.InnerException is SqlException { Number: 2601 or 2627 } sqlException
        && sqlException.Message.Contains("UX_Employees_Email", StringComparison.OrdinalIgnoreCase);
}
```

## Modules/Employees/Services/EmployeeService.cs

The HTTP boundary validates DTOs before the service is called; non-HTTP callers must also
validate input. The service still validates business invariants.
The email pre-check only improves the error message; it is **not a replacement for the unique constraint**.

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
    Task<EmployeeResponse> UpdateAsync(Guid id, UpdateEmployeeRequest request, CancellationToken cancellationToken = default);
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
            throw new AppException(ErrorCode.InvalidBusinessRule, "Salary and department must be valid.");

        var email = request.Email.Trim().ToUpperInvariant();
        if (await repository.ExistsByEmailAsync(email, cancellationToken))
            throw new ConflictException(ErrorCode.EntityAlreadyExists, "Email is already registered.");

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

    public async Task<EmployeeResponse> UpdateAsync(
        Guid id, UpdateEmployeeRequest request, CancellationToken cancellationToken = default)
    {
        if (request.Salary <= 0 || request.DepartmentId == Guid.Empty)
            throw new AppException(ErrorCode.InvalidBusinessRule, "Salary and department must be valid.");

        var employee = await repository.GetForUpdateAsync(id, cancellationToken)
            ?? throw new NotFoundException(ErrorCode.EntityNotFound);

        employee.FirstName = request.FirstName.Trim();
        employee.LastName = request.LastName.Trim();
        employee.Position = request.Position.Trim();
        employee.Salary = request.Salary;
        employee.DepartmentId = request.DepartmentId;
        employee.UpdatedAt = timeProvider.GetUtcNow();

        await repository.SaveChangesAsync(cancellationToken);
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

    [HttpPut("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<EmployeeResponse>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ApiResponse), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ApiResponse), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(
        Guid id, UpdateEmployeeRequest request, CancellationToken cancellationToken)
    {
        var result = await service.UpdateAsync(id, request, cancellationToken);
        return Ok(ApiResponse<EmployeeResponse>.SuccessResult(result, timeProvider));
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

## MyService.Api.Tests/Modules/Employees/EmployeeServiceTests.cs

Service unit tests use **xUnit**, **NSubstitute**, and **FakeTimeProvider**
(`Microsoft.Extensions.TimeProvider.Testing`). If the project already uses other assertion
or mocking libraries, follow the project; do not add dependencies just for this example.
These unit tests do not prove the HTTP envelope, model validation, or the unique constraint;
those are covered by integration tests (see the last section).

```csharp
using Microsoft.Extensions.Logging.Abstractions;
using Microsoft.Extensions.Time.Testing;
using NSubstitute;
using Xunit;
using MyService.Api.Common.Errors;
using MyService.Api.Common.Exceptions;
using MyService.Api.Common.Models;
using MyService.Api.Modules.Employees.Dtos;
using MyService.Api.Modules.Employees.Entities;
using MyService.Api.Modules.Employees.Repositories;
using MyService.Api.Modules.Employees.Services;

namespace MyService.Api.Tests.Modules.Employees;

public sealed class EmployeeServiceTests
{
    private readonly IEmployeeRepository _repository = Substitute.For<IEmployeeRepository>();
    private readonly FakeTimeProvider _clock = new(new DateTimeOffset(2026, 1, 1, 0, 0, 0, TimeSpan.Zero));
    private readonly EmployeeService _sut;

    public EmployeeServiceTests() =>
        _sut = new EmployeeService(_repository, NullLogger<EmployeeService>.Instance, _clock);

    [Fact]
    public async Task CreateAsync_WhenEmailAlreadyExists_ThrowsConflictWithoutSaving()
    {
        var request = new CreateEmployeeRequest("John", "Doe", "john@corp.com", "Engineer", 5000, Guid.NewGuid());
        _repository.ExistsByEmailAsync("JOHN@CORP.COM", Arg.Any<CancellationToken>()).Returns(true);

        var exception = await Assert.ThrowsAsync<ConflictException>(
            () => _sut.CreateAsync(request, CancellationToken.None));

        Assert.Equal(ErrorCode.EntityAlreadyExists, exception.ErrorCode);
        _repository.DidNotReceive().Add(Arg.Any<Employee>());
        await _repository.DidNotReceive().SaveChangesAsync(Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task CreateAsync_WhenValid_NormalizesEmailUsesClockAndForwardsToken()
    {
        using var cts = new CancellationTokenSource();
        var request = new CreateEmployeeRequest(" Jane ", "Smith", "jane@corp.com", "Lead", 8000, Guid.NewGuid());

        var result = await _sut.CreateAsync(request, cts.Token);

        Assert.Equal("Jane Smith", result.FullName);
        Assert.Equal(_clock.GetUtcNow(), result.CreatedAt);
        await _repository.Received(1).ExistsByEmailAsync("JANE@CORP.COM", cts.Token);
        _repository.Received(1).Add(Arg.Is<Employee>(employee =>
            employee.Email == "JANE@CORP.COM" && employee.CreatedAt == _clock.GetUtcNow()));
        await _repository.Received(1).SaveChangesAsync(cts.Token);
    }

    [Fact]
    public async Task GetByIdAsync_WhenNotFound_ThrowsNotFound()
    {
        var id = Guid.NewGuid();
        _repository.GetByIdAsync(id, Arg.Any<CancellationToken>()).Returns((Employee?)null);

        var exception = await Assert.ThrowsAsync<NotFoundException>(
            () => _sut.GetByIdAsync(id, CancellationToken.None));

        Assert.Equal(ErrorCode.EntityNotFound, exception.ErrorCode);
    }

    [Fact]
    public async Task UpdateAsync_WhenFound_UpdatesFieldsAndAuditTimestamp()
    {
        using var cts = new CancellationTokenSource();
        var employee = new Employee
        {
            FirstName = "Old", LastName = "Name", Email = "OLD@CORP.COM", Position = "Staff",
            Salary = 1000, DepartmentId = Guid.NewGuid(), CreatedAt = _clock.GetUtcNow()
        };
        _repository.GetForUpdateAsync(employee.Id, cts.Token).Returns(employee);
        _clock.Advance(TimeSpan.FromHours(1));

        var result = await _sut.UpdateAsync(
            employee.Id, new UpdateEmployeeRequest(" New ", "Name", "Lead", 9000, employee.DepartmentId), cts.Token);

        Assert.Equal("New Name", result.FullName);
        Assert.Equal(_clock.GetUtcNow(), employee.UpdatedAt);
        await _repository.Received(1).SaveChangesAsync(cts.Token);
    }

    [Fact]
    public async Task UpdateAsync_WhenNotFound_ThrowsNotFoundWithoutSaving()
    {
        var id = Guid.NewGuid();
        _repository.GetForUpdateAsync(id, Arg.Any<CancellationToken>()).Returns((Employee?)null);
        var request = new UpdateEmployeeRequest("A", "B", "C", 1000, Guid.NewGuid());

        await Assert.ThrowsAsync<NotFoundException>(() => _sut.UpdateAsync(id, request, CancellationToken.None));

        await _repository.DidNotReceive().SaveChangesAsync(Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task GetPageAsync_ForwardsCancellationTokenToRepository()
    {
        using var cts = new CancellationTokenSource();
        _repository.GetPageAsync(1, 20, cts.Token).Returns(new PagedResult<EmployeeResponse>([], 1, 20, 0));

        await _sut.GetPageAsync(1, 20, cts.Token);

        await _repository.Received(1).GetPageAsync(1, 20, cts.Token);
    }
}
```

## Before using the example in a real feature

- Add a migration for constraints/indexes. Do not assume an index exists just because the configuration was written.
- The example repository translates SQL Server `UX_Employees_Email` violations into 409,
  including concurrent inserts. For other providers, index names, or unique keys, adapt
  the error metadata check; do not map every `DbUpdateException` to 409.
  Prove it with a concurrent insert integration test on the real relational provider.
- Check salary precision/range, email normalization, Department ownership/access, and
  soft-delete rules according to the domain. Do not assume a valid identifier means the user may access it.
- If mutations need optimistic concurrency, add a concurrency token and a
  `DbUpdateConcurrencyException` mapping that fits the contract. The example does not define an edit conflict policy yet.
- Count and page are two queries; they do not guarantee the same snapshot under concurrent writes.
  Use an appropriate isolation level/contract if snapshot consistency is required.
- Test invalid DTO/JSON, zero/too large/overflowing pages, not found, controlled timestamps,
  forwarded tokens, and concurrent duplicate emails on the real relational provider.
