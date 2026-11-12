# Changelog

Semua perubahan penting pada repository **Enterprise .NET Agent Skills** ini didokumentasikan dalam berkas ini.

Format dokumen ini mengacu pada [Keep a Changelog](https://keepachangelog.com/id/1.0.0/) dan mengikuti standar [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

## [2.0.0] - 2026-11-12

### ⚠ Breaking changes

- G1: Controller menerima `CancellationToken` tanpa `= default`.
- G2: Error validasi otomatis memakai envelope `ApiResponse`; response file/stream, HEAD,
  204/304, health check, dan dokumen OpenAPI tidak lagi dibungkus envelope.
- G5: List endpoint baru wajib dibatasi dengan pengurutan unik.
- Isi `dotnet-api-design` dipindahkan dari `SKILL.md` ke `references/`; skill tidak lagi saling menautkan.

### Added

- File `LICENSE` (MIT).
- Contoh endpoint `PUT` (Update) pada modul Employees beserta DTO `UpdateEmployeeRequest`.
- Contoh unit test Service (xUnit, NSubstitute, FakeTimeProvider) untuk konflik email,
  audit timestamp, not found, dan propagasi `CancellationToken`.
- `AGENTS.md` berisi instruksi maintenance untuk repository katalog skill ini.
- Skenario README yang menunjukkan cara menggabungkan dua skill secara eksplisit dalam satu prompt.

### Fixed

- Repository Employees kini menerjemahkan pelanggaran index `UX_Employees_Email` (SQL Server
  2601/2627) menjadi `ConflictException` di boundary persistence; sebelumnya contoh belum
  menyertakan adapter ini sehingga insert email duplikat secara bersamaan menjadi 500.

### Changed

- Panduan instalasi README kini memakai URL repository yang benar (`renaldi99/dotnet-skills`),
  mendokumentasikan opsi `--branch <tag>` untuk memakai rilis tertentu, dan menambahkan langkah
  clone pada instalasi global (Opsi 3).
- README menautkan setiap AI coding assistant yang didukung ke halaman produknya.
- Panduan kontribusi README kini berisi checklist seluruh bagian yang wajib diperbarui
  saat menambah skill baru; `AGENTS.md` merujuk checklist tersebut.
- Pulihkan diagram struktur project di referensi `module-scaffolding.md`, selaras
  dengan path contoh dan tanpa menambah konteks di file utama skill.
- Restrukturisasi `dotnet-api-design`: aturan G1-G7 dan workflow tetap di `SKILL.md`;
  fondasi, contoh modul, bootstrap, dan enforcement dipindahkan ke referensi on-demand.
- Tambahkan trigger dan workflow maintenance/bug fix dengan perlindungan kontrak existing,
  serta jalur review tanpa perubahan file.
- Selaraskan contoh clock injection, read-only versus tracked mutation, DTO projection,
  pagination bounded, dan pemetaan error validasi otomatis ke envelope.
- Perjelas batas cancellation, konfigurasi autentikasi opsional, constraint uniqueness,
  serta integration test untuk konflik concurrent insert.
- Tegaskan setiap skill berdiri sendiri tanpa tautan atau ketergantungan antar-skill;
  kombinasi skill disebutkan pengguna secara eksplisit di prompt.
- Dokumentasikan pemasangan mandiri, discovery yang bergantung host, dan anchor instruksi
  pada repository backend tanpa menjanjikan kepatuhan AI absolut.

---

## [1.0.0] - 2026-11-11

### Added

- **Skill `dotnet-api-design`** ([`dotnet-api-design/SKILL.md`](./dotnet-api-design/SKILL.md)):
  - Standar perancangan, scaffolding, dan refactoring RESTful API pada **.NET 10 (C# 14)** berbasis arsitektur **Modular Monolith**.
  - **7 Core Guardrails**:
    1. Wajib parameter `CancellationToken cancellationToken = default` pada semua method async.
    2. Wajib standarisasi wrapper response `ApiResponse<T>` / `ApiResponse`.
    3. Wajib explicit versioning pada route controller (`api/v1/[controller]`).
    4. Larangan generic repository (wajib interface repository spesifik per modul).
    5. Penyelesaian operasi JOIN di database level & larangan in-memory join.
    6. Penggunaan idiom modern C# 14 (Primary Constructors, Collection Expressions `[..]`, dan `TimeProvider`).
    7. Penanganan error terpusat dengan melempar turunan `AppException` dan ditangani oleh `GlobalExceptionHandler` (`IExceptionHandler`).
  - Dokumen referensi performa database ([`dotnet-api-design/references/database-performance.md`](./dotnet-api-design/references/database-performance.md)) yang mencakup pencegahan N+1 query, panduan `AsSplitQuery`, dan paginasi.
  - Template golden scaffolding modul (`Employees`) mencakup Controller, Service, Repository, DTO records, dan registrasi DI.

- **Skill `efcore-pattern`** ([`efcore-pattern/SKILL.md`](./efcore-pattern/SKILL.md)):
  - Panduan implementasi dan best practices **Entity Framework Core (EF Core 9 / .NET 10 & C# 14)** untuk persistence layer berkinerja tinggi.
  - **6 Core Guardrails**:
    1. Wajib `.AsNoTracking()` atau proyeksi langsung DTO pada semua query pembacaan data (GET).
    2. Pemisahan konfigurasi entitas menggunakan class `IEntityTypeConfiguration<T>` tersendiri.
    3. Larangan query database di dalam perulangan loop (eliminasi N+1 query).
    4. Wajib propagasi `CancellationToken` ke semua method async EF Core.
    5. Penggunaan native SQL bulk operations (`ExecuteUpdateAsync` & `ExecuteDeleteAsync`) untuk mutasi masal.
    6. Wajib penggunaan abstraksi `TimeProvider` pada interceptor dan domain logic.
  - Interceptor audit trail otomatis (`AuditSaveChangesInterceptor`) untuk pengisian `CreatedAt` dan `UpdatedAt`.
  - Extension method paginasi standar (`ToPagedResultAsync`).

- **Dokumentasi & Repository Setup**:
  - `README.md` komprehensif berisi katalog skill, ringkasan guardrails, 3 opsi instalasi (Workspace, Git Submodule, Global Antigravity), dan contoh prompt aktivasi.
  - `.gitignore` komprehensif mencakup sistem operasi, IDE, cache AI/scratch, dan build artifacts .NET.
