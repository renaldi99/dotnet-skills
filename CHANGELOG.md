# Changelog

All notable changes to the **.NET Agent Skills** repository are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### ⚠ Breaking changes

- Skills moved into a `skills/` directory (`skills/dotnet-api-design`, `skills/efcore-pattern`).
  Update install scripts to copy from `skills/<skill-name>`; installed skill paths in consuming
  projects (`.agents/skills/<skill-name>`) are unchanged.
- `efcore-pattern`: new guardrail 7, `AppDbContext` may be injected only into repositories.
  Services and controllers that inject `AppDbContext` directly no longer comply; move their
  data access into a specific repository. Pattern 4 no longer offers the direct-service option.

### Added

- `efcore-pattern`: Pattern 4 service example that depends only on `IProductRepository`,
  an anti-pattern row for `AppDbContext` in services, and a matching checklist item.

### Fixed

- `efcore-pattern`: the read (Pattern 2A) and pagination (Pattern 7) examples now end their
  ordering on a unique key (`.ThenBy(p => p.Id)`), and the checklist adds a unique-ordering item.
- `efcore-pattern`: `ProductRepository.GetByIdAsync` is now `AsNoTracking()` as guardrail 1
  requires; a separate tracked `GetForUpdateAsync` is added for mutations.
- `efcore-pattern`: the checklist no longer suggests `TimeProvider.System.GetUtcNow()` in code;
  it requires an injected `TimeProvider`, matching guardrail 6.

## [2.1.0] - 2026-11-12

### Changed

- Translate both skills, their references, README, AGENTS.md, and this changelog to English.
  Rules, G1-G7 IDs, structure, and code behavior are unchanged; message strings in code examples
  are now in English. Use `v2.0.0` for the Indonesian version.

---

## [2.0.0] - 2026-11-12

### ⚠ Breaking changes

- G1: controllers accept `CancellationToken` without `= default`.
- G2: automatic validation errors use the `ApiResponse` envelope; file/stream, HEAD,
  204/304, health check, and OpenAPI responses are no longer wrapped.
- G5: new list endpoints must be bounded with a unique ordering.
- `dotnet-api-design` content moved from `SKILL.md` into `references/`;
  skills no longer link to each other.

### Added

- `PUT` (Update) endpoint example in the Employees module with the `UpdateEmployeeRequest` DTO.
- Service unit test examples (xUnit, NSubstitute, FakeTimeProvider) for email conflicts,
  audit timestamps, not found, and `CancellationToken` propagation.
- `AGENTS.md` with maintenance instructions for this skill catalog repository.
- `LICENSE` file with the MIT License text referenced by the README.
- README usage scenario showing how to combine two skills explicitly in one prompt.

### Fixed

- The Employees repository now translates `UX_Employees_Email` index violations (SQL Server
  2601/2627) into `ConflictException` at the persistence boundary; previously the example did not
  include this adapter, so concurrent duplicate email inserts resulted in 500.

### Changed

- README installation guide now uses the real repository URL (`renaldi99/dotnet-skills`),
  documents optional `--branch <tag>` pinning to a release, and adds the missing clone step
  to the global install (Option 3).
- README links each supported AI coding assistant to its product page.
- The README contributing guide now has a checklist of every section to update
  when adding a new skill; `AGENTS.md` points to that checklist.
- Restore the project structure diagram in the `module-scaffolding.md` reference, aligned
  with the example paths and without adding context to the main skill file.
- Restructure `dotnet-api-design`: G1-G7 rules and workflows stay in `SKILL.md`;
  foundation, module example, bootstrap, and enforcement move to on-demand references.
- Add triggers and maintenance/bug fix workflows that protect existing contracts,
  plus a review path without file changes.
- Align examples for clock injection, read-only versus tracked mutation, DTO projection,
  bounded pagination, and mapping automatic validation errors to the envelope.
- Clarify cancellation boundaries, optional authentication configuration, uniqueness constraints,
  and integration tests for concurrent insert conflicts.
- State that each skill stands alone without links or dependencies between skills;
  users name skill combinations explicitly in their prompts.
- Document standalone installation, host-dependent discovery, and instruction anchors
  in the backend repository without promising absolute AI compliance.

---

## [1.0.0] - 2026-11-11

### Added

- **`dotnet-api-design` skill** ([`skills/dotnet-api-design/SKILL.md`](./skills/dotnet-api-design/SKILL.md)):
  - Standard for designing, scaffolding, and refactoring RESTful APIs on **.NET 10 (C# 14)** with a **Modular Monolith** architecture.
  - **7 Core Guardrails**:
    1. Required `CancellationToken cancellationToken = default` parameter on every async method.
    2. Required standard `ApiResponse<T>` / `ApiResponse` response wrapper.
    3. Required explicit versioning on controller routes (`api/v1/[controller]`).
    4. No generic repositories (a specific repository interface per module is required).
    5. JOIN operations resolved at the database level & no in-memory joins.
    6. Modern C# 14 idioms (Primary Constructors, Collection Expressions `[..]`, and `TimeProvider`).
    7. Centralized error handling by throwing `AppException` subclasses handled by `GlobalExceptionHandler` (`IExceptionHandler`).
  - Database performance reference document ([`skills/dotnet-api-design/references/database-performance.md`](./skills/dotnet-api-design/references/database-performance.md)) covering N+1 query prevention, `AsSplitQuery` guidance, and pagination.
  - Golden scaffolding template for the `Employees` module covering Controller, Service, Repository, DTO records, and DI registration.

- **`efcore-pattern` skill** ([`skills/efcore-pattern/SKILL.md`](./skills/efcore-pattern/SKILL.md)):
  - Implementation guide and best practices for **Entity Framework Core (EF Core 9 / .NET 10 & C# 14)** for a high-performance persistence layer.
  - **6 Core Guardrails**:
    1. Required `.AsNoTracking()` or direct DTO projection on every data read query (GET).
    2. Entity configuration separated into dedicated `IEntityTypeConfiguration<T>` classes.
    3. No database queries inside loops (N+1 query elimination).
    4. Required `CancellationToken` propagation to every async EF Core method.
    5. Native SQL bulk operations (`ExecuteUpdateAsync` & `ExecuteDeleteAsync`) for bulk mutations.
    6. Required `TimeProvider` abstraction in interceptors and domain logic.
  - Automatic audit trail interceptor (`AuditSaveChangesInterceptor`) for filling `CreatedAt` and `UpdatedAt`.
  - Standard pagination extension method (`ToPagedResultAsync`).

- **Documentation & Repository Setup**:
  - Comprehensive `README.md` with the skill catalog, guardrails summary, 3 installation options (Workspace, Git Submodule, Global Antigravity), and activation prompt examples.
  - Comprehensive `.gitignore` covering operating systems, IDEs, AI/scratch caches, and .NET build artifacts.
