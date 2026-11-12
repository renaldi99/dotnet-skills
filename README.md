# 🤖 Enterprise .NET Agent Skills

Kumpulan **Agent Skills** standar untuk AI Coding Assistants ([Google Antigravity](https://antigravity.google), Claude Code, Cursor, GitHub Copilot, dll.) yang dirancang untuk mempercepat pembuatan, refactoring, dan standarisasi arsitektur backend enterprise berbasis **.NET 10 (C# 14)** dan **Entity Framework Core**.

---

## 📦 Katalog Skills

Repository ini berisi kumpulan skill modular yang dapat di-load secara *on-demand* oleh AI Agent:

| Skill | Lingkup & Fokus | Target Platform | Dokumen |
| :--- | :--- | :--- | :---: |
| **`dotnet-api-design`** | Perancangan RESTful API berbasis **Modular Monolith**, standardisasi `ApiResponse<T>` envelope, explicit versioning `api/v1/`, primary constructors C# 14, `TimeProvider`, centralized `IExceptionHandler`, dan xUnit testing. | .NET 10 / C# 14 | [`SKILL.md`](./dotnet-api-design/SKILL.md) |
| **`efcore-pattern`** | Desain persistence layer berkinerja tinggi, konfigurasi terpisah `IEntityTypeConfiguration<T>`, eliminasi N+1 query, audit interceptors otomatis, bulk update/delete SQL native, dan paginasi standar. | EF Core 9 / 10 | [`SKILL.md`](./efcore-pattern/SKILL.md) |

---

## 🎯 Ringkasan Guardrails Utama

Setiap skill dibekali aturan mutlak (*core guardrails*) untuk memastikan AI menghasilkan kode berkualitas produksi:

### 1. `dotnet-api-design`
- **Wajib CancellationToken**: Semua method async (Controller, Service, Repository) wajib menerima & meneruskan `CancellationToken`.
- **Wajib Envelope Response**: Semua response dibungkus `ApiResponse<T>` atau `ApiResponse`. Dilarang me-return raw entity/object.
- **Explicit Versioning**: Route wajib menggunakan prefix `[Route("api/v1/[controller]")]`.
- **Dilarang Generic Repository**: Buat interface repository spesifik per modul (misal `IEmployeeRepository`).
- **Modern C# 14 & Centralized Error**: Menggunakan Primary Constructors, Collection Expressions (`[..]`), `TimeProvider`, dan turunan `AppException` yang ditangani oleh `GlobalExceptionHandler`.

### 2. `efcore-pattern`
- **Wajib `.AsNoTracking()`**: Semua operasi pembacaan (GET) wajib menggunakan `.AsNoTracking()` atau proyeksi langsung via `.Select()`.
- **Pemisahan Konfigurasi**: Skema, index, dan relasi tabel wajib diletakkan di class `IEntityTypeConfiguration<T>` terpisah (bukan Data Annotations di model).
- **Anti N+1 Query**: Wajib menggunakan eager loading (`.Include()`) atau proyeksi `.Select()`. Dilarang mengeksekusi query database di dalam perulangan loop.
- **Native Bulk Operations**: Wajib gunakan `.ExecuteUpdateAsync()` dan `.ExecuteDeleteAsync()` untuk mutasi masal tanpa me-load entitas ke memori.
- **TimeProvider Interceptors**: Otomatisasi kolom audit (`CreatedAt`, `UpdatedAt`) wajib menginjeksikan abstraksi `TimeProvider` (bukan `DateTime.UtcNow` statis).

---

## 🚀 Cara Instalasi

Pilih salah satu metode instalasi yang paling sesuai dengan alur kerja Anda:

### Opsi 1: Per-Project (Workspace Level) — Direkomendasikan
Pasang skill ini langsung ke dalam project backend Anda agar tim dan AI di repo tersebut otomatis mematuhi standar yang sama.

Buat folder `.agents/skills` di root project Anda, lalu salin folder skill yang diinginkan:

```bash
# Jalankan di root repository project .NET Anda:
mkdir -p .agents/skills

# Clone repo ini ke folder temporary lalu copy skill yang dibutuhkan:
git clone --depth 1 https://github.com/<username>/agent-skills.git /tmp/agent-skills
cp -r /tmp/agent-skills/dotnet-api-design .agents/skills/
cp -r /tmp/agent-skills/efcore-pattern .agents/skills/
rm -rf /tmp/agent-skills
```

> **Catatan Struktur Folder di Project Anda:**
> ```text
> my-dotnet-project/
> ├── .agents/
> │   └── skills/
> │       ├── dotnet-api-design/
> │       │   ├── SKILL.md
> │       │   └── references/
> │       └── efcore-pattern/
> │           └── SKILL.md
> ├── src/
> └── MyProject.sln
> ```

---

### Opsi 2: Menggunakan Git Submodule (Mudah Di-update)
Jika Anda ingin menyinkronkan pembaruan skill dari repository ini ke berbagai project dengan mudah:

```bash
# Di root project Anda:
git submodule add https://github.com/<username>/agent-skills.git .agents/skills/agent-skills
```

---

### Opsi 3: Global Install (Tersedia di Seluruh Project Laptop Anda)
Jika Anda menggunakan **Google Antigravity** dan ingin skill ini aktif di semua project tanpa perlu menyalin ke setiap workspace:

```bash
# Antigravity Global Customizations
mkdir -p ~/.gemini/config/skills
cp -r dotnet-api-design ~/.gemini/config/skills/
cp -r efcore-pattern ~/.gemini/config/skills/
```

---

## 💡 Contoh Trigger Pemakaian

Skill ini menerapkan prinsip **Progressive Disclosure**. Anda tidak perlu menghafal instruksi teknis; cukup ajukan kebutuhan Anda dalam bahasa natural, dan agen AI akan otomatis mengaktifkan skill yang relevan:

### Skenario 1: Scaffolding Modul Baru (.NET 10)
> *"Buatkan modul vertikal baru 'Orders' dengan endpoint POST, GET by id, dan GET paginated list. Pastikan sudah mengikuti arsitektur modular monolith kita."*

### Skenario 2: Optimasi Performa & Refactoring EF Core
> *"Review service `CustomerService.cs` ini. Kenapa query di endpoint get all terasa lambat? Tolong optimasi dan hilangkan potensi N+1 query."*

### Skenario 3: Implementasi Soft-Delete & Audit Trail
> *"Tolong buatkan interceptor EF Core untuk otomatis mengisi kolom audit `CreatedAt` dan `UpdatedAt` menggunakan `TimeProvider`."*

---

## 📁 Struktur Repository

```text
agent-skills/
├── README.md                           # Dokumentasi utama dan katalog skill
├── .gitignore                          # Aturan ignore file OS dan cache temporary
│
├── dotnet-api-design/                  # Skill Arsitektur RESTful API .NET 10
│   ├── SKILL.md                        # Panduan utama, core rules, & scaffolding blueprint
│   └── references/
│       └── database-performance.md     # Referensi optimasi query & AsSplitQuery
│
└── efcore-pattern/                     # Skill Pola Desain EF Core 9/10
    └── SKILL.md                        # Panduan entity configuration, interceptor, & bulk SQL
```

---

## 🤝 Berkontribusi

Ingin menambahkan skill baru atau memperbarui panduan yang ada?
1. Fork repository ini.
2. Buat direktori skill baru: `<nama-skill>/SKILL.md`.
3. Pastikan `SKILL.md` menyertakan YAML frontmatter dengan format:
   ```markdown
   ---
   name: nama-skill
   description: >-
     Deskripsi kapan skill ini digunakan (trigger aktivasi) dan batasannya.
   ---
   ```
4. Dokumentasikan panduan panjang di dalam subfolder `references/`.
5. Daftarkan skill baru Anda pada tabel di `README.md`.
6. Ajukan Pull Request.

---

## 📄 Lisensi

Didistribusikan di bawah lisensi [MIT License](LICENSE). Bebas digunakan, dimodifikasi, dan didistribusikan untuk proyek komersial maupun non-komersial.

