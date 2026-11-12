# Enforcement dan pemasangan

Baca saat memasang skill atau menyiapkan guardrail proyek, bukan setiap implementasi.
Skill on-demand mengurangi konteks, tetapi tidak menjamin aktivasi/kepatuhan pada setiap prompt.
Instruksi host juga bukan pengganti pemeriksaan deterministik.

## Tiga lapisan

| Lapisan | Tujuan |
| :--- | :--- |
| Instruksi repository yang dikenali host | Mengarahkan tugas API ke skill sebelum perubahan; pendek agar murah dimuat berulang. |
| SKILL.md dan referensi on-demand | Sumber utama G1-G7, workflow, dan contoh sesuai kebutuhan. |
| Analyzer/test/CI | Menolak pelanggaran yang dapat diukur, bukan hanya meminta AI mengingatnya. |

## Anchor untuk repository backend pengguna

Gabungkan snippet berikut dengan instruksi repository **backend yang menggunakan skill**,
bukan hanya AGENTS.md di repository katalog skill. Jangan menimpa instruksi pengguna.
Pilih file instruksi/path skill sesuai dokumentasi host dan lokasi instalasi nyata.
Contoh ini mengasumsikan host mendukung `.agents/skills` dan instruksi repository yang Anda pilih.

```markdown
Untuk membuat, memperbaiki, mereview, atau maintain API .NET controller-based:
- Sebelum mengedit, baca `.agents/skills/dotnet-api-design/SKILL.md`.
- Terapkan G1-G7; baca hanya referensi yang relevan dengan perubahan.
- Pertahankan kontrak dan pola existing. Minta konfirmasi jika perubahan
  memerlukan breaking change atau pengecualian aturan yang belum disetujui.
- Setelah konteks diringkas/hilang, baca ulang aturan sebelum melanjutkan.
- Sebelum selesai, periksa diff terhadap G1-G7 dan jalankan pemeriksaan
  paling sempit yang relevan; laporkan keterbatasan, jangan mengklaim lulus.
```

Tidak perlu menyalin seluruh guardrail dan template ke semua file instruksi.
Jika memakai path lain, ubah anchor; jika host tidak mendukung instruksi repository,
gunakan mekanisme aturan host atau pemanggilan skill eksplisit. Verifikasi discovery di host
target setelah pemasangan, termasuk bila memakai submodule bertingkat.

## Enforcement yang dapat ditambahkan sesuai proyek

| Aturan | Bukti yang disarankan |
| :--- | :--- |
| G1 | Analyzer signature ditambah test propagation dari request ke dependency; signature saja tidak membuktikan token dipakai. |
| G2/G3 | Integration test HTTP melalui host nyata: body/status/schema, route/version, invalid JSON, dan endpoint khusus tanpa envelope. |
| G4 | Architecture test dependency boundary/interface; hindari pemeriksaan nama string sebagai satu-satunya bukti. |
| G5 | Relational integration test untuk tracking, query translation, pagination, constraint, dan concurrent write; ukur query untuk kasus N+1. |
| G6 | Analyzer penggunaan static clock dan test waktu dengan injected FakeTimeProvider. |
| G7 | Integration test domain error, unexpected error, auth failure bila ada, dan request cancellation; verifikasi log tanpa membocorkan detail internal ke response. |

Gunakan tooling proyek yang sudah ada. Menambahkan analyzer/test framework baru memerlukan
kebutuhan nyata, bukan syarat memasang skill. Grep/checklist membantu review tetapi bukan
bukti semantik bahwa seluruh aturan dipatuhi.

## Skenario evaluasi skill

Coba di sesi bersih: membuat modul baru, menambah satu field, memperbaiki bug endpoint,
mereview tanpa edit, serta melanjutkan maintenance setelah context compaction.
Periksa apakah referensi dipilih seperlunya, kontrak dipertahankan, dan pelanggaran dilaporkan.
Jangan menganggap satu prompt berhasil sebagai jaminan untuk semua model/host.
