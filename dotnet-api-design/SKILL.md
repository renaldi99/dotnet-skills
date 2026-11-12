---
name: dotnet-api-design
description: >-
  Rancang, buat, review, dan maintain REST API controller-based .NET dengan standar
  Modular Monolith. Gunakan untuk API/modul baru, penambahan endpoint atau fitur,
  bug fix backend, perubahan kontrak/validasi, serta refactoring controller,
  service, dan repository. Target proyek baru: .NET 10 / C# 14.
  Mencakup CancellationToken, ApiResponse, versioning, TimeProvider, dan error handling.
---

# .NET Modular API Design

Skill ini menetapkan standar proyek, bukan klaim bahwa semua API .NET harus memakai arsitektur yang sama.
Untuk maintenance, pertahankan framework, provider database, dan kontrak yang sudah digunakan.

## Aktivasi dan batasan

- Gunakan untuk scaffolding, implementasi fitur, diagnosis bug, review, dan maintenance API dalam lingkup di atas.
- Jangan gunakan untuk Console/Worker tanpa HTTP, migrasi microservices/event bus, atau Minimal API tanpa layering.
- Permintaan review/analisis saja tidak memberi izin mengubah file.
- Jika tugas membutuhkan perubahan kontrak atau arsitektur di luar permintaan, minta persetujuan terlebih dahulu.

## Core guardrails: sumber aturan utama

Baca G1-G7 sebelum mengedit dan periksa kembali sebelum menyatakan selesai.
Referensi memberi contoh penerapan, bukan aturan alternatif.

| ID | Aturan wajib |
| :--- | :--- |
| **G1 Cancellation** | Method async Controller/Service/Repository menerima dan meneruskan token request sampai operasi I/O. Controller menerima `CancellationToken cancellationToken`; Service/Repository boleh memakai `= default`, tetapi pemanggil wajib meneruskan token yang tersedia. Jangan membuat method async palsu untuk operasi sinkron. |
| **G2 Kontrak HTTP** | Response JSON aplikasi memakai `ApiResponse<T>`/`ApiResponse`, termasuk error validasi otomatis. Jangan mengekspos entity EF. Pertahankan status HTTP yang benar. Jangan membungkus file/stream, HEAD, 204/304, health check, atau dokumen OpenAPI; dokumentasikan pengecualian endpoint. |
| **G3 Versioning** | Controller API v1 baru memakai `[Route("api/v1/[controller]")]`. Pada versi/proyek existing, ikuti versi dan route publiknya; jangan memaksa semua endpoint kembali ke v1. Perubahan route, schema, atau status publik memerlukan persetujuan jika belum diminta. |
| **G4 Repository** | Gunakan interface repository spesifik per modul. Jangan membuat `IRepository<T>`/`GenericRepository<T>` atau membocorkan `IQueryable` ke Controller/Service. Jangan merombak abstraksi existing di luar lingkup tugas. |
| **G5 Data** | Read-only entity query memakai `AsNoTracking()`; proyeksi scalar/DTO tanpa entity tidak memerlukan tracking. Load untuk mutasi boleh tracked dan harus eksplisit. Join/filter/pagination dilakukan sebelum materialisasi di database; hindari N+1. List endpoint baru wajib bounded dengan pengurutan unik. |
| **G6 C# dan waktu** | Gunakan primary constructor untuk dependency injection dan collection expression ketika sesuai tipe/semantik serta versi bahasa proyek mendukungnya. Gunakan injected `TimeProvider` untuk waktu saat ini, termasuk envelope dan audit; jangan memakai clock statis. DTO timestamp memakai `DateTimeOffset`. |
| **G7 Error** | Kegagalan bisnis memakai turunan `AppException`, ditangani terpusat oleh `IExceptionHandler`; jangan memasang catch-all di Controller. Jangan menyamarkan error tak dikenal sebagai sukses/konflik bisnis. Bedakan client disconnect dari timeout internal; jangan menulis body ke koneksi yang sudah dibatalkan. |

Tidak ada aturan teks yang menjamin kepatuhan absolut. Ikuti hierarki instruksi host;
laporkan konflik dengan aturan proyek dan minta klarifikasi, jangan diam-diam mengabaikannya.

## Pilih referensi sesuai tugas

Jangan membaca semua referensi sekaligus. Buka hanya baris yang relevan; lewati contoh
jika pola existing sudah cukup. Jangan muat ulang file yang masih tersedia utuh dalam konteks.

| Kebutuhan | Baca |
| :--- | :--- |
| Membuat/mengubah envelope, domain error, exception handler | [Foundation](references/foundation.md) |
| Membuat modul baru atau membutuhkan contoh layering | [Module scaffolding](references/module-scaffolding.md) |
| Membuat aplikasi baru atau mengubah DI/HTTP pipeline/validasi otomatis | [Bootstrap](references/bootstrap.md) |
| Mengubah query, pagination, uniqueness, relasi, atau bulk operation | [Database performance](references/database-performance.md) |
| Memasang skill atau memperkuat kepatuhan dengan instruksi host/CI | [Enforcement](references/enforcement.md) |

Skill ini dapat dipasang sendiri. `efcore-pattern`, jika tersedia, hanya perlu diaktifkan
untuk pekerjaan persistence yang membutuhkan pendalaman; bukan prasyarat atau bacaan wajib.
Untuk tugas gabungan, G1-G7 tetap berlaku pada kontrak API.

## Preflight: semua perubahan kode

1. Tentukan mode: **proyek/modul baru**, **maintenance**, atau **review**.
2. Baca instruksi repository dan file pemilik fitur: project/TFM, Controller, Service,
   Repository, DTO, konfigurasi DI, serta test terkait. Telusuri hanya jalur yang terpengaruh.
3. Identifikasi kontrak HTTP, strategi autentikasi/otorisasi, provider database,
   validasi, transaksi/concurrency, dan pola error existing yang relevan.
4. Catat aturan G1-G7 yang berlaku serta pengecualian yang sudah disepakati.
   Jika konteks sebelumnya diringkas/hilang, baca kembali aturan dan file terkait sebelum mengedit.
5. Pilih referensi dari tabel. Jangan menginstal paket atau membuat fondasi baru
   jika kemampuan yang dibutuhkan sudah tersedia di proyek.

## Workflow A: proyek atau modul baru

1. Tetapkan kontrak endpoint, DTO, validasi, versi, pagination, dan kebutuhan akses.
   Untuk proyek baru, gunakan .NET 10 / C# 14; untuk modul baru di proyek existing, ikuti proyek.
2. Gunakan struktur vertikal `Modules/{ModuleName}/`:
   `Controllers/`, `Services/`, `Repositories/`, `Dtos/`, `Entities/`, `Validators/`
   (bila diperlukan), dan `{ModuleName}ModuleExtensions.cs`.
3. Reuse fondasi `Common/` dan `Data/`. Jika belum ada, baca Foundation dan Bootstrap;
   jangan menyalin/menimpa `Program.cs` secara utuh ke proyek existing.
4. Implementasikan Entity/configuration dan DTO, lalu repository spesifik, service,
   controller tipis, dan registrasi DI scoped. Laksanakan G1-G7 pada setiap layer.
5. Untuk write: validasi sebelum mutasi, normalisasi input secara konsisten, dan
   gunakan constraint database untuk invariant seperti uniqueness. Tentukan batas transaksi.
6. Tambahkan test terfokus untuk happy path, input tidak valid, error bisnis,
   propagasi cancellation, dan kontrak HTTP. Gunakan test stack existing;
   default proyek baru adalah xUnit, dengan NSubstitute/FakeTimeProvider bila diperlukan.
7. Jalankan completion gate di bawah.

## Workflow B: maintenance, fitur existing, atau bug fix

1. Reproduksi masalah atau definisikan acceptance criteria pada jalur yang berubah.
2. Pertahankan route, schema JSON, status HTTP, auth policy, dan perilaku data yang tidak diminta berubah.
   Jangan upgrade TFM, mengganti ORM, menambahkan layer, atau mengganti seluruh modul.
3. Terapkan perubahan terkecil beserta caller/DI/test yang langsung bergantung padanya.
   Pelanggaran lama di luar lingkup dilaporkan, bukan diperbaiki secara massal.
4. Jika memperbaiki bug, tambahkan regression test yang gagal tanpa perbaikan.
   Perubahan DB wajib mempertimbangkan migration, constraint, dan kompatibilitas data.
5. Jalankan completion gate. Konflik aturan yang membutuhkan breaking change
   harus dikonfirmasi, bukan diselesaikan dengan perubahan kontrak diam-diam.

## Workflow C: review saja

Laporkan temuan berbukti dengan path/baris, dampak, dan usulan terkecil.
Prioritaskan correctness, kontrak publik, dan G1-G7; jangan mengubah file atau dependency.
Pisahkan pelanggaran aturan proyek dari preferensi desain umum.

## Completion gate

- Periksa diff terhadap **G1-G7**, bukan hanya kode yang baru ditulis. Scope pemeriksaan adalah jalur yang berubah.
- Verifikasi DTO/status/envelope untuk success, validation failure, dan domain error yang terpengaruh.
  Untuk perubahan auth/pipeline, periksa juga 401/403/404/405/415 sesuai kontrak.
- Verifikasi tracking read/write, batas list, urutan pagination unik, dan token sampai operasi I/O.
- Jalankan build dan test paling sempit yang mencakup perubahan, menggunakan konfigurasi proyek/CI:
  `dotnet build <project-or-solution> --configuration Release` dan
  `dotnet test <test-project> --configuration Release` dengan filter yang relevan.
- Jangan menambah error/warning. Laporkan kegagalan baseline yang tidak terkait;
  jangan mengklaim lulus jika dependency, database, atau test tidak dapat dijalankan.
- Untuk kode baru tanpa baseline, targetkan build bersih dan test terkait lulus.
- Ringkas hasil, perubahan kontrak yang disetujui, dan keterbatasan nyata; jangan mengulang seluruh skill.
