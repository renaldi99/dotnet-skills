# Instruksi maintenance katalog skill

File ini berlaku untuk repository katalog skill ini saja, bukan untuk proyek backend
yang memasang skill. Anchor untuk proyek pemakai ada di
`dotnet-api-design/references/enforcement.md`.

## Sumber aturan

- `dotnet-api-design/SKILL.md` adalah satu-satunya sumber G1-G7. Referensi, README, dan
  CHANGELOG hanya merujuk atau memberi contoh; jangan menulis versi aturan yang berbeda.
- Setiap skill berdiri sendiri: jangan menautkan, mewajibkan, atau menyelaraskan isi satu
  skill dengan skill lain. Kombinasi skill ditentukan pengguna melalui prompt.

## Saat mengubah skill

- Jaga `SKILL.md` ringkas (target di bawah ~180 baris, batas keras 500 baris). Contoh kode
  panjang masuk `references/` dan didaftarkan di tabel "Pilih referensi sesuai tugas".
- Frontmatter wajib memiliki `name` (sama dengan nama folder) dan `description` yang memuat
  trigger serta lingkup.
- Contoh antar-file harus konsisten: nama tipe, namespace, signature, bentuk `PagedResult<T>`,
  kode error, dan path di diagram struktur. Ubah semua tempat yang terdampak sekaligus.
- Jangan menambah dependency pada contoh (library assertion, validasi, mapping) kecuali
  dibutuhkan untuk menunjukkan aturan; sebutkan alternatif "ikuti proyek existing".
- Contoh harus bisa dikompilasi secara masuk akal: `using` lengkap, tidak ada helper yang
  dirujuk tanpa definisi, dan unit test benar-benar meng-assert (bukan memanggil mock).

## Sebelum menyatakan selesai

1. Periksa tautan relatif di `SKILL.md`, README, dan CHANGELOG masih valid.
2. Periksa jumlah baris `SKILL.md` (`wc -l */SKILL.md`).
3. Cari sisa pola lama yang sudah dihapus (misalnya `grep -rn "<pola lama>" .`).
4. Catat perubahan di `CHANGELOG.md` bagian `[Unreleased]`.
5. Jangan commit/push tanpa permintaan pemilik repository.
