# 🤖 Enterprise .NET Agent Skills

Kumpulan **Agent Skills** standar untuk AI Coding Assistants ([Google Antigravity](https://antigravity.google), [Claude Code](https://claude.com/product/claude-code), [Cursor](https://cursor.com/), [GitHub Copilot](https://github.com/features/copilot), dll.) yang dirancang untuk mempercepat pembuatan, refactoring, dan standarisasi arsitektur backend enterprise berbasis **.NET 10 (C# 14)** dan **Entity Framework Core**.

---

## 📦 Katalog Skills

Repository ini berisi kumpulan skill modular yang dapat di-load secara _on-demand_ oleh AI Agent:

<!-- prettier-ignore -->
| Skill | Lingkup & Fokus | Target Platform | Dokumen |
| :--- | :--- | :--- | :---: |
| **`dotnet-api-design`** | Scaffolding, fitur baru, bug fix, review, dan maintenance REST API **Modular Monolith** dengan guardrail G1-G7 dan referensi on-demand. | Proyek baru: .NET 10 / C# 14; maintenance mengikuti proyek | [`SKILL.md`](./dotnet-api-design/SKILL.md) |
| **`efcore-pattern`** | Desain persistence layer berkinerja tinggi, konfigurasi terpisah `IEntityTypeConfiguration<T>`, eliminasi N+1 query, audit interceptors otomatis, bulk update/delete SQL native, dan paginasi standar. | EF Core 9 / 10 | [`SKILL.md`](./efcore-pattern/SKILL.md) |

---

## 🎯 Ringkasan Guardrails Utama

Setiap skill berdiri sendiri dengan _core guardrails_ masing-masing dan tidak saling bergantung.
Untuk memakai lebih dari satu skill, sebutkan skill tersebut secara eksplisit di prompt. Kepatuhan perlu diperiksa;
instruksi teks saja tidak menjamin AI selalu mengaktifkan skill atau mengikuti semua aturan.

### 1. `dotnet-api-design`

Sumber aturan utama adalah **G1-G7 di [SKILL.md](./dotnet-api-design/SKILL.md)**:
cancellation, kontrak HTTP, versioning, repository spesifik, query database, C#/waktu,
dan error handling. Pengecualian HTTP serta perbedaan read-only dan tracked mutation
didefinisikan di sana, bukan diduplikasi di README.

Workflow dipisahkan menjadi **scaffolding**, **maintenance**, dan **review tanpa edit**.
Template panjang dimuat hanya saat diperlukan.

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

Pasang skill di project backend agar standar dapat digunakan bersama oleh tim dan AI.

Contoh berikut memakai `.agents/skills`. Pastikan host AI mendukung lokasi tersebut;
sesuaikan dengan direktori skill yang didokumentasikan host Anda bila berbeda.
Salin seluruh folder skill, termasuk `references/`:

```bash
# Jalankan di root repository project .NET Anda:
mkdir -p .agents/skills

# Clone repo ini ke folder temporary lalu copy skill yang dibutuhkan:
git clone --depth 1 https://github.com/renaldi99/dotnet-skills.git /tmp/dotnet-skills
cp -r /tmp/dotnet-skills/dotnet-api-design .agents/skills/
cp -r /tmp/dotnet-skills/efcore-pattern .agents/skills/
rm -rf /tmp/dotnet-skills
```

Perintah di atas mengambil versi terbaru dari branch default. Untuk versi rilis tertentu,
tambahkan `--branch <tag>` pada `git clone` (misalnya `--branch v1.1.0`).

> **Catatan Struktur Folder di Project Anda:**
>
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
git submodule add https://github.com/renaldi99/dotnet-skills.git .agents/skills/dotnet-skills
```

Periksa apakah host menemukan skill dalam folder bertingkat. Jika tidak, gunakan instalasi
langsung per skill atau lokasi yang didukung host; keberadaan submodule tidak menjamin discovery.

---

### Opsi 3: Global Install (Tersedia di Seluruh Project Laptop Anda)

Jika Anda menggunakan **Google Antigravity** dan ingin skill ini aktif di semua project tanpa perlu menyalin ke setiap workspace:

```bash
# Antigravity Global Customizations
git clone --depth 1 https://github.com/renaldi99/dotnet-skills.git /tmp/dotnet-skills
mkdir -p ~/.gemini/config/skills
cp -r /tmp/dotnet-skills/dotnet-api-design ~/.gemini/config/skills/
cp -r /tmp/dotnet-skills/efcore-pattern ~/.gemini/config/skills/
rm -rf /tmp/dotnet-skills
```

---

## 💡 Contoh Trigger Pemakaian

Skill ini menerapkan **Progressive Disclosure**: metadata membantu pemilihan skill,
`SKILL.md` memuat aturan/workflow inti, dan referensi dibaca sesuai tugas. Aktivasi otomatis
bergantung pada host/model; verifikasi skill tersedia dan gunakan pemanggilan eksplisit bila diperlukan.

### Skenario 1: Scaffolding Modul Baru (.NET 10)

> _"Buatkan modul vertikal baru 'Orders' dengan endpoint POST, GET by id, dan GET paginated list. Pastikan sudah mengikuti arsitektur modular monolith kita."_

### Skenario 2: Optimasi Performa & Refactoring EF Core

> _"Review service `CustomerService.cs` ini. Kenapa query di endpoint get all terasa lambat? Tolong optimasi dan hilangkan potensi N+1 query."_

### Skenario 3: Implementasi Soft-Delete & Audit Trail

> _"Tolong buatkan interceptor EF Core untuk otomatis mengisi kolom audit `CreatedAt` dan `UpdatedAt` menggunakan `TimeProvider`."_

### Skenario 4: Maintenance tanpa mengubah kontrak

> _"Gunakan dotnet-api-design untuk memperbaiki validasi endpoint Orders ini. Pertahankan route, schema response, dan versi framework; tambahkan regression test yang relevan."_

### Skenario 5: Menggabungkan dua skill secara eksplisit

> _"Gunakan dotnet-api-design dan efcore-pattern untuk menambahkan endpoint laporan Orders per bulan. Tambahkan bulk update status via ExecuteUpdateAsync."_

### Memperkuat kepatuhan aturan

Gunakan [panduan enforcement](./dotnet-api-design/references/enforcement.md) untuk memasang
anchor instruksi singkat di **repository backend pengguna** dan memilih analyzer/test/CI
yang relevan. `AGENTS.md` di katalog skill ini tidak otomatis berlaku pada proyek pemakai.
Jangan menyalin seluruh template ke instruksi permanen; arahkan ke G1-G7 dan baca referensi seperlunya.

---

## 📁 Struktur Repository

```text
dotnet-skills/
├── README.md                           # Dokumentasi utama dan katalog skill
├── CHANGELOG.md                        # Riwayat perubahan
├── AGENTS.md                           # Instruksi maintenance katalog (bukan untuk proyek pemakai)
├── .gitignore                          # Aturan ignore file OS dan cache temporary
│
├── dotnet-api-design/                  # Skill Arsitektur RESTful API .NET 10
│   ├── SKILL.md                        # G1-G7, routing referensi, dan workflow ringkas
│   └── references/
│       ├── foundation.md               # Envelope, error codes, exception handler
│       ├── module-scaffolding.md       # Contoh modul Employees
│       ├── bootstrap.md                # DI, validasi otomatis, HTTP pipeline
│       ├── database-performance.md     # Query, pagination, uniqueness, bulk
│       └── enforcement.md              # Instruksi host dan pemeriksaan otomatis
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
5. Pastikan skill berdiri sendiri: jangan menautkan atau mewajibkan skill lain.
6. Daftarkan skill baru di setiap bagian `README.md` berikut:
   - [ ] **Katalog Skills**: baris tabel (nama, lingkup, target platform, tautan `SKILL.md`).
   - [ ] **Ringkasan Guardrails Utama**: subbagian `### N. <nama-skill>` yang merujuk ke
         `SKILL.md`, bukan menyalin seluruh aturan.
   - [ ] **Cara Instalasi**: perintah `cp -r` di Opsi 1 dan Opsi 3, serta folder skill di
         diagram "Catatan Struktur Folder".
   - [ ] **Contoh Trigger Pemakaian**: minimal satu skenario; tambahkan contoh prompt gabungan
         bila skill lazim dipakai bersama skill lain.
   - [ ] **Struktur Repository**: folder skill beserta `SKILL.md` dan `references/` bila ada.
7. Catat penambahan di `CHANGELOG.md` bagian `[Unreleased]` → `Added`.
8. Ajukan Pull Request.

---

## 📄 Lisensi

Didistribusikan di bawah lisensi [MIT License](LICENSE). Bebas digunakan, dimodifikasi, dan didistribusikan untuk proyek komersial maupun non-komersial.
