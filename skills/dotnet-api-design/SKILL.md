---
name: dotnet-api-design
description: >-
  Design, build, review, and maintain controller-based .NET REST APIs following a
  Modular Monolith standard. Use for new APIs/modules, adding endpoints or features,
  backend bug fixes, contract/validation changes, and refactoring controllers,
  services, and repositories. New projects target .NET 10 / C# 14.
  Covers CancellationToken, ApiResponse, versioning, TimeProvider, and error handling.
---

# .NET Modular API Design

This skill defines a project standard; it does not claim every .NET API must use the same architecture.
For maintenance, keep the framework, database provider, and contracts already in use.

## Activation and boundaries

- Use for scaffolding, feature implementation, bug diagnosis, review, and API maintenance within the scope above.
- Do not use for Console/Worker apps without HTTP, microservices/event bus migrations, or Minimal APIs without layering.
- A review/analysis-only request does not grant permission to modify files.
- If the task requires contract or architecture changes beyond the request, ask for approval first.
- In existing projects, the project's established conventions (response shape, data access,
  error handling) take precedence for new code in the same area. Apply G1-G7 only where they
  do not contradict those conventions, and propose adopting the standard as a separate change.

## Core guardrails: primary source of rules

Read G1-G7 before editing and check them again before declaring the work done.
References provide implementation examples, not alternative rules.

<!-- prettier-ignore -->
| ID | Mandatory rule |
| :--- | :--- |
| **G1 Cancellation** | Async Controller/Service/Repository methods accept and forward the request token down to the I/O operation. Controllers accept `CancellationToken cancellationToken`; Services/Repositories may use `= default`, but callers must forward the available token. Do not create fake async methods for synchronous operations. |
| **G2 HTTP contract** | Application JSON responses use `ApiResponse<T>`/`ApiResponse`, including automatic validation errors. Do not expose EF entities. Keep correct HTTP status codes. Do not wrap file/stream, HEAD, 204/304, health check, or OpenAPI document responses; document endpoint exceptions. |
| **G3 Versioning** | New v1 API controllers use `[Route("api/v1/[controller]")]`. In existing versions/projects, follow their public version and routes; do not force every endpoint back to v1. Changes to public routes, schemas, or status codes require approval if not requested. |
| **G4 Repository** | Use a specific repository interface per module. Do not create `IRepository<T>`/`GenericRepository<T>` or leak `IQueryable` to Controllers/Services. Do not overhaul existing abstractions outside the task scope. |
| **G5 Data** | Read-only entity queries use `AsNoTracking()`; scalar/DTO projections without entities need no tracking. Loads for mutation may be tracked and must be explicit. Joins/filters/pagination run in the database before materialization; avoid N+1. New list endpoints must be bounded with a unique ordering. |
| **G6 C# and time** | Use primary constructors for dependency injection and collection expressions when they fit the type/semantics and the project's language version supports them. Use an injected `TimeProvider` for the current time, including envelopes and audit fields; do not use a static clock. DTO timestamps use `DateTimeOffset`. |
| **G7 Errors** | Business failures use `AppException` subclasses, handled centrally by `IExceptionHandler`; do not add catch-all blocks in Controllers. Do not disguise unknown errors as success/business conflicts. Distinguish client disconnects from internal timeouts; do not write a body to an already-cancelled connection. |

No text rule guarantees absolute compliance. Follow the host's instruction hierarchy;
report conflicts with project rules and ask for clarification instead of silently ignoring them.

## Choose references by task

Do not read all references at once. Open only the relevant sections; skip examples
when existing patterns are sufficient. Do not reload files that are still fully in context.

<!-- prettier-ignore -->
| Need | Read |
| :--- | :--- |
| Create/change the envelope, domain errors, or exception handler | [Foundation](references/foundation.md) |
| Create a new module, layering example, or unit test pattern | [Module scaffolding](references/module-scaffolding.md) |
| Create a new application or change DI/HTTP pipeline/automatic validation | [Bootstrap](references/bootstrap.md) |
| Change queries, pagination, uniqueness, relationships, or bulk operations | [Database performance](references/database-performance.md) |
| Install the skill or strengthen compliance with host instructions/CI | [Enforcement](references/enforcement.md) |

If the user asks for another skill to be used together with this one, G1-G7 still apply to
the API contract and layering; report rule conflicts instead of silently picking one.

## Preflight: every code change

1. Determine the mode: **new project/module**, **maintenance**, or **review**.
2. Read repository instructions and the files that own the feature: project/TFM, Controller, Service,
   Repository, DTOs, DI configuration, and related tests. Trace only the affected paths.
3. Identify the relevant HTTP contract, authentication/authorization strategy, database provider,
   validation, transaction/concurrency, and existing error patterns.
4. Note which G1-G7 rules apply and any agreed exceptions.
   If earlier context was summarized/lost, reread the rules and related files before editing.
5. Pick references from the table. Do not install packages or build new foundations
   if the project already provides the needed capability.

## Workflow A: new project or module

1. Define the endpoint contract, DTOs, validation, version, pagination, and access requirements.
   New projects use .NET 10 / C# 14; new modules in existing projects follow the project.
2. Use the vertical structure `Modules/{ModuleName}/`:
   `Controllers/`, `Services/`, `Repositories/`, `Dtos/`, `Entities/`, `Validators/`
   (when needed), and `{ModuleName}ModuleExtensions.cs`. Existing projects keep their folder layout.
3. Reuse the `Common/` and `Data/` foundations. In a new project without them, read Foundation and
   Bootstrap; in an existing project without them, follow its established conventions instead.
   Do not copy/overwrite `Program.cs` wholesale into an existing project.
4. Implement the Entity/configuration and DTOs, then the specific repository, service,
   thin controller, and scoped DI registration. Apply G1-G7 in every layer, within the
   project's conventions (see Activation and boundaries).
5. For writes: validate before mutating, normalize input consistently, and
   use database constraints for invariants such as uniqueness. Define transaction boundaries.
6. Add focused tests for the happy path, invalid input, business errors,
   cancellation propagation, and the HTTP contract. Use the existing test stack;
   the default for new projects is xUnit, with NSubstitute/FakeTimeProvider when needed.
7. Run the completion gate below.

## Workflow B: maintenance, existing feature, or bug fix

1. Reproduce the problem or define acceptance criteria for the changed path.
2. Preserve routes, JSON schema, HTTP status codes, auth policy, and data behavior that were not asked to change.
   Do not upgrade the TFM, replace the ORM, add layers, or rewrite an entire module.
3. Apply the smallest change plus the callers/DI/tests that directly depend on it.
   Pre-existing violations outside the scope are reported, not fixed en masse.
4. When fixing a bug, add a regression test that fails without the fix.
   Database changes must consider migrations, constraints, and data compatibility.
5. Run the completion gate. Rule conflicts that require a breaking change
   must be confirmed, not resolved with a silent contract change.

## Workflow C: review only

Report evidence-based findings with path/line, impact, and the smallest suggested fix.
Prioritize correctness, public contracts, and G1-G7; do not change files or dependencies.
Separate project rule violations from general design preferences.

## Completion gate

- Check the diff against **G1-G7**, not only newly written code. The scope is the changed paths.
  In existing projects, established project conventions take precedence where they differ from G1-G7.
- Verify DTOs/status/response shape (the envelope, or the project's existing format) for success,
  validation failure, and affected domain errors.
  For auth/pipeline changes, also check 401/403/404/405/415 according to the contract.
- Verify read/write tracking, list bounds, unique pagination ordering, and tokens down to I/O.
- Run the narrowest build and tests that cover the change, using the project/CI configuration:
  `dotnet build <project-or-solution> --configuration Release` and
  `dotnet test <test-project> --configuration Release` with a relevant filter.
- Do not add errors/warnings. Report unrelated baseline failures;
  do not claim success if dependencies, the database, or tests could not be run.
- For new code without a baseline, target a clean build and passing related tests.
- Summarize results, approved contract changes, and real limitations; do not repeat the whole skill.
