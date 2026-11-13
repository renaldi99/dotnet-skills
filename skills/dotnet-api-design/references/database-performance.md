# Database performance

Read only when changing repositories/queries. G4-G5 in SKILL.md are the source of rules;
the examples below are fragments to adapt to the project's entities/provider, not complete application files.

## Query decisions

| Need | Choice |
| :--- | :--- |
| Read-only entity | `AsNoTracking()`; DTO/scalar projections without entities are not tracked either. |
| Load for mutation | An explicit tracked query, change the needed fields, then `SaveChangesAsync(token)`. Do not `Update` an entire detached graph without need. |
| Relationships/N+1 | `Select` projection, SQL join, or eager loading. Do not query per item in a loop. |
| Several sibling collections | Consider `AsSplitQuery()` after weighing cartesian explosion, roundtrips, and consistency; it is not the default for every Include. |
| Existence | `AnyAsync(token)`, not `CountAsync() > 0`. The SQL shape depends on the provider; it is not always a literal `IF EXISTS`. |
| List | Filter/order/paginate before materialization, with a page size limit and a unique ordering. Use keyset/cursor for large traversals when the contract allows. |
| Bulk update/delete | `ExecuteUpdateAsync`/`ExecuteDeleteAsync` when invariants, auditing, concurrency, and transactions are still satisfied. Available since EF Core 7, not specific to .NET 10. |

## Database joins and DTO projection

Suppose the model has `Employee.DepartmentId` and `Department.Id/Name`.
Use a dedicated summary DTO; do not call the `EmployeeResponse` constructor with a different shape.
The example uses `offset` and `pageSize` already validated as in the pagination section.

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

This is an inner join. Do not assume a navigation projection always produces a LEFT JOIN;
required/optional relationships and the provider determine the SQL translation.
Do not load both tables and then perform a LINQ Join in the application. Materializing relationships via
EF split queries or Dapper multi-mapping is not the same prohibition as an unbounded in-memory join of two tables.

## Dapper: parameterized SQL projection

Use the summary DTO above. This SQL Server example keeps column names that match
the DTO constructor and passes the token via `CommandDefinition`. Adjust the dialect for other providers.

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

For one-to-many with separate result sets, consider `QueryMultipleAsync`.
Bound child collections, dispose the `GridReader`, and do not combine partial results
as if they were complete entities. For multi-mapping, check `splitOn` and the identity column of each type.

## Pagination and uniqueness

- Reject `page < 1`, `pageSize` outside the policy limits, and offsets beyond the integer range.
  Do not silently clamp invalid input. The module example uses a default of 20 and a maximum of 100.
- Ordering must be unique: for example `OrderByDescending(CreatedAt).ThenBy(Id)`.
  A unique ordering does not remove offset drift when data changes; use a cursor if needed.
- `CountAsync` remains correct for the total count. Weigh the cost of the count query and snapshot consistency
  against the endpoint's needs; do not compute a total if the contract does not require it.
- Normalize keys before the pre-check **and** the insert/update with the same function.
  Avoid `ToLower()` on query columns, which prevents index usage.
- Uniqueness must be guaranteed by a database constraint/index, with a migration and an appropriate
  collation policy. A pre-check alone is not safe against concurrent inserts.
- Translate only recognized constraint violations into domain conflicts. Connection errors,
  timeouts, FK violations, or other constraints must not be disguised as a duplicate email.

## Split queries and bulk operations

Split queries add roundtrips and may read different snapshots across queries while data changes.
Measure the SQL/query plan and consider transactions/isolation when consistency is required.

Bulk operations run directly, without the change tracker or `SaveChanges` interceptors:

```csharp
var now = timeProvider.GetUtcNow();
var affected = await context.Set<Employee>()
    .Where(employee => employee.DepartmentId == targetDeptId && employee.IsActive)
    .ExecuteUpdateAsync(update => update
        .SetProperty(employee => employee.IsActive, false)
        .SetProperty(employee => employee.UpdatedAt, now), cancellationToken);
```

Decide the business outcome based on `affected`. Add a concurrency filter if needed.
Do not expect a later `SaveChanges` to apply the bulk operation; do not use stale tracked
entities without a deliberate reload/clear. Multiple bulk statements do not
automatically form one transaction; wrap them explicitly if they must be atomic.

## Relevant evidence

For query changes, check the SQL, roundtrip count, row/column counts, and the query plan
at the scale of the case. Test translation, constraints, and transactions with a compatible
relational provider; EF InMemory does not prove that behavior.
