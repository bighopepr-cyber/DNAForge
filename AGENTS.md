# AGENTS.md — Aturan Perilaku Pengembangan DNAForge

Dokumen ini mengatur bagaimana agen (manusia atau AI) bekerja di repositori DNAForge. Dokumen ini WAJIB dibaca dan dipatuhi sebelum mengubah kode, dokumentasi, konfigurasi, maupun CI.

Tujuan utama: produk selesai secara utuh, konsisten dengan spesifikasi, dan setiap perubahan dapat diverifikasi. Kecepatan tidak boleh mengorbankan kebenaran.

---

## 1. Sumber Kebenaran

1. Satu-satunya sumber kebenaran adalah **Spesifikasi Teknis DNAForge v1** (`docs/SPEC.md`). Jika file tersebut belum ada di repositori, agen WAJIB berhenti dan melaporkan hal itu sebelum menulis kode apa pun.
2. Kode DILARANG bertentangan dengan spesifikasi. Jika kode dan spesifikasi berbeda, spesifikasi yang berlaku, dan kode WAJIB diperbaiki.
3. Agen DILARANG menambahkan perilaku, format, konstanta, field, parameter CLI, atau dependensi yang tidak tercantum di spesifikasi.
4. Jika spesifikasi ambigu, tidak lengkap, atau kontradiktif, agen WAJIB:
   a. Berhenti pada bagian tersebut.
   b. Membuka issue dengan label `spec-clarification` yang menjelaskan ambiguitas, dua interpretasi yang mungkin, dan dampak masing-masing.
   c. Tidak mengambil keputusan sendiri yang memengaruhi format atau perilaku yang dapat diamati.
5. Perubahan spesifikasi hanya boleh dilakukan melalui PR tersendiri yang diberi label `spec-change`, disetujui pemilik proyek, dan memperbarui `format_version` bila perubahan memengaruhi format biner atau pool.

---

## 2. Prinsip Inti

1. **Tidak ada implementasi setengah jadi.** Setiap PR harus lengkap untuk lingkup yang diklaim. Fitur yang belum selesai tidak boleh digabung ke branch utama.
2. **Tidak ada placeholder di kode yang digabung.** Dilarang: `todo!()`, `unimplemented!()`, `unreachable!()` sebagai pengganti logika yang belum ditulis, `// TODO`, `// FIXME`, `// XXX`, fungsi stub yang mengembalikan nilai palsu, dan kode yang dikomentari.
3. **Satu perubahan, satu tujuan.** Setiap PR hanya menyelesaikan satu unit kerja yang jelas. Refaktor, perbaikan format, dan fitur baru tidak boleh dicampur dalam satu PR.
4. **Determinisme adalah syarat, bukan optimasi.** Keluaran encode, simulate, dan decode harus identik untuk masukan dan seed yang sama, terlepas dari jumlah thread atau urutan eksekusi.
5. **Bukti lebih penting dari keyakinan.** Klaim tentang perilaku, performa, atau kebenaran harus didukung oleh pengujian, benchmark, atau hasil yang dapat direproduksi.
6. **Kode library tidak boleh panik karena masukan.** Semua masukan dari pengguna atau kanal diperlakukan sebagai tidak tepercaya.

---

## 3. Arsitektur yang Wajib Dijaga

### 3.1 Struktur Workspace

Workspace berisi tepat tujuh crate library dan satu crate binary, sesuai Bagian 4.1 spesifikasi:

- `dnaforge-core`
- `dnaforge-codec`
- `dnaforge-ecc`
- `dnaforge-fragment`
- `dnaforge-channel`
- `dnaforge-recovery`
- `dnaforge-engine`
- `dnaforge-bench`
- `dnaforge-cli` (binary `dnaforge`)

Agen DILARANG menambah crate, mengubah nama crate, atau memindahkan tanggung jawab antar crate tanpa `spec-change`.

### 3.2 Graf Dependensi

- Graf dependensi WAJIB asiklik.
- Crate yang lebih rendah DILARANG mengimpor crate yang lebih tinggi.
- Hanya `dnaforge-engine` yang boleh melakukan I/O berkas. Crate lain DILARANG membuka berkas, membaca stdin, menulis stdout/stderr, atau mengakses jaringan.
- `dnaforge-cli` dan `dnaforge-bench` boleh melakukan I/O hanya sebatas yang dibutuhkan untuk antarmuka pengguna dan pelaporan.

### 3.3 Aturan Kode Lintas Crate

- Setiap crate WAJIB memiliki `#![forbid(unsafe_code)]` di `lib.rs` atau `main.rs`.
- Setiap crate WAJIB mendefinisikan enum error sendiri dengan `thiserror`.
- Konversi error antarcrate WAJIB eksplisit (`impl From` atau `map_err` yang jelas). Dilarang menggunakan `Box<dyn Error>` pada API library.
- Tidak ada state global yang dapat berubah. Dilarang `static mut`, `lazy_static` untuk state mutable, atau singleton tersembunyi.
- Konstanta format WAJIB didefinisikan hanya di `dnaforge-core` dan diimpor dari sana. Angka ajaib yang mewakili konstanta format di crate lain dilarang.

---

## 4. Lingkungan dan Dependensi

### 4.1 Toolchain

- Versi Rust dikunci melalui `rust-toolchain.toml`. Agen DILARANG mengubah versi toolchain tanpa PR tersendiri.
- MSRV: 1.75. Setiap perubahan WAJIB lulus pengujian pada MSRV.
- Edition: 2021.
- Profil `release` WAJIB sesuai spesifikasi Bagian 5.1.

### 4.2 Dependensi

- Hanya dependensi yang tercantum pada spesifikasi Bagian 5.2 yang boleh ditambahkan.
- Penambahan dependensi baru WAJIB melalui PR dengan label `dependency` yang menjelaskan alasan, alternatif yang dipertimbangkan, dan hasil `cargo deny` serta `cargo audit`.
- Dependensi WAJIB ditentukan dengan versi yang dikunci di `Cargo.lock` dan dicatat di `Cargo.toml` dengan rentang kompatibel.
- Jalur format (encode dan decode) DILARANG menggunakan `rand` atau RNG eksternal. Semua pembangkit acak berasal dari `dnaforge-core`.
- Dependensi `dev` hanya boleh dipakai di kode uji, benchmark, atau fuzz.

### 4.3 Lisensi

Setiap dependensi WAJIB lolos kebijakan lisensi di `deny.toml`. Lisensi yang tidak tercantum di daftar izin dianggap gagal.

---

## 5. Standar Kode

### 5.1 Gaya dan Format

- `cargo fmt --all` WAJIB dijalankan sebelum commit. Tidak ada pengecualian format manual.
- `cargo clippy --workspace --all-targets -- -D warnings` WAJIB lulus. Dilarang menambahkan `#[allow(clippy::...)]` tanpa komentar yang menjelaskan alasannya dan disetujui dalam review.
- `#[allow(...)]` apa pun WAJIB disertai komentar alasan di baris yang sama atau tepat di atasnya.

### 5.2 Penanganan Error

- Library DILARANG memakai `unwrap()`, `expect()`, `panic!()`, indeks array tanpa pemeriksaan batas pada data dari masukan, atau aritmetika yang dapat overflow tanpa penanganan eksplisit.
- Aritmetika pada nilai dari masukan WAJIB memakai `checked_*`, `saturating_*`, atau `wrapping_*` dengan niat yang jelas.
- Pada kode uji, `unwrap()` dan `expect()` diizinkan dengan pesan yang menjelaskan asumsi.
- Setiap error WAJIB membawa konteks yang cukup (misalnya seq, indeks grup, atau offset) untuk diagnosis.

### 5.3 Dokumentasi Kode

- Setiap fungsi, struct, enum, dan trait publik WAJIB memiliki `///` yang menyebut:
  - Prasyarat masukan.
  - Kondisi error yang mungkin.
  - Referensi ke bagian spesifikasi jika relevan, misalnya `Spesifikasi §8.2`.
- Kode internal yang logikanya tidak jelas dari nama WAJIB memiliki komentar yang menjelaskan *mengapa*, bukan *apa*.
- `cargo doc --workspace --no-deps` WAJIB berjalan tanpa peringatan.

### 5.4 Penamaan

- Nama mengikuti istilah pada spesifikasi Bagian 3. Dilarang membuat sinonim baru untuk istilah yang sudah didefinisikan (misalnya tidak boleh menulis `chunk` jika spesifikasi menggunakan `fragmen` atau `oligo`).
- Konstanta spesifikasi memakai nama persis seperti Bagian 6, misalnya `OLIGO_NT`, `PAYLOAD_BYTES`.

---

## 6. Determinisme dan Konkurensi

- Struktur data yang urutannya memengaruhi hasil WAJIB `Vec`, `BTreeMap`, atau urutan eksplisit.
- `HashMap` atau `HashSet` hanya boleh dipakai untuk pencarian, dan hasilnya WAJIB diurutkan sebelum digunakan.
- Paralelisme (Rayon) hanya diizinkan pada: encoding per grup, konsensus per klaster, dan decoding RS per grup.
- Clustering WAJIB sekuensial.
- Hasil paralel WAJIB dikumpulkan berdasarkan indeks asal.
- Setiap fitur yang mengubah urutan atau jumlah operasi paralel WAJIB disertai pengujian determinisme pada `--threads 1` dan `--threads 8`.
- Waktu, lingkungan, dan urutan iterasi sistem operasi DILARANG memengaruhi keluaran format.

---

## 7. Format dan Kompatibilitas

- Konstanta format berada pada `dnaforge-core` dan tidak boleh diubah tanpa `spec-change`.
- Perubahan apa pun yang memengaruhi byte pool, layout oligo, CRC, whitening, keystream, seed kanal, atau pemetaan seq WAJIB:
  1. Menaikkan `FORMAT_VERSION` jika berlaku.
  2. Memperbarui nilai golden (Bagian 8 di bawah).
  3. Menjelaskan dampak kompatibilitas di deskripsi PR.
- Mengubah nilai golden tanpa perubahan format atau revisi spesifikasi DILARANG.
- Parser WAJIB menolak format versi yang tidak didukung dengan error yang jelas, tidak menebak.

---

## 8. Pengujian

### 8.1 Kewajiban Pengujian

- Setiap fungsi logika baru WAJIB memiliki pengujian unit.
- Setiap aturan spesifikasi yang diimplementasikan WAJIB memiliki pengujian yang secara eksplisit merujuk ke bagian spesifikasi tersebut di nama atau komentar pengujian.
- Setiap bug yang diperbaiki WAJIB memiliki pengujian regresi yang gagal sebelum perbaikan dan lulus sesudahnya.
- Pengujian yang diberi `#[ignore]` DILARANG kecuali disertai ID issue yang terbuka dan batas waktu penyelesaian.

### 8.2 Jenis Pengujian Wajib

Sesuai Bagian 23 spesifikasi:
- **Unit:** per crate, mencakup kasus batas.
- **Property-based (proptest):** round-trip encode/decode, pack/unpack, parsing oligo, dan jarak edit.
- **Integrasi:** seluruh ukuran berkas pada Bagian 23.2 dan skenario erasure.
- **Golden:** SHA-256 pool untuk berkas 64 KiB dengan seed 42.
- **Determinisme paralel:** Bagian 23.6.
- **Memori:** Bagian 23.5.
- **Fuzzing:** Bagian 23.4, dijalankan terjadwal.

### 8.3 Aturan Pengujian

- Pengujian WAJIB deterministik. Dilarang bergantung pada waktu sistem, jumlah core, atau RNG tanpa seed.
- Pengujian probabilistik WAJIB memakai seed tetap dan batas statistik yang dinyatakan di spesifikasi.
- Pengujian WAJIB independen satu sama lain. Dilarang ada urutan eksekusi yang diasumsikan.
- Berkas sementara WAJIB dibuat dengan `tempfile` dan dibersihkan otomatis.

### 8.4 Cakupan

Ambang minimum garis (`cargo llvm-cov`):
- `core`, `codec`, `ecc`, `fragment`: 90%.
- `recovery`, `engine`: 80%.

Cakupan tidak boleh diraih dengan pengujian kosong. Pengujian harus memverifikasi perilaku, bukan sekadar menjalankan kode.

---

## 9. Benchmark dan Eksperimen

- Benchmark mikro menggunakan Criterion dan WAJIB mengikuti daftar pada Bagian 24.1 spesifikasi.
- Setiap benchmark WAJIB memakai `black_box`.
- Hasil benchmark tidak boleh dilaporkan tanpa lingkungan lengkap (CPU, core, RAM, OS, rustc, commit).
- Klaim performa dalam PR, dokumentasi, atau laporan WAJIB disertai data mentah dan metode pengukuran.
- Regresi performa mengikuti Bagian 24.4: penurunan throughput lebih dari 10% atau kenaikan peak memori lebih dari 10% memblokir merge.
- Eksperimen ketahanan WAJIB dijalankan sesuai protokol Bagian 24.2. Dilarang mengubah grid, seed, atau kriteria keberhasilan setelah melihat hasil.
- Kegagalan eksperimen WAJIB dilaporkan. Dilarang menyembunyikan kombinasi parameter yang gagal.
- Setiap metrik WAJIB dihitung sesuai definisi Bagian 25. Interval kepercayaan Wilson untuk proporsi dan median serta IQR untuk waktu WAJIB dilaporkan.

---

## 10. Batasan Klaim

- Setiap dokumen, laporan, README, dan pesan commit yang mengklaim hasil WAJIB menyatakan bahwa hasil berasal dari simulasi komputasi.
- Dilarang mengklaim keberhasilan sintesis, penyimpanan fisik, atau sekuensing nyata.
- Dilarang menggunakan kata "terbukti", "menjamin", atau "100%" untuk hasil simulasi tanpa batasan yang jelas.

---

## 11. Workflow Pengembangan

### 11.1 Sebelum Menulis Kode

1. Baca bagian spesifikasi yang relevan secara lengkap, termasuk bagian pengujian dan metrik terkait.
2. Pastikan tidak ada `spec-clarification` terbuka yang memblokir pekerjaan.
3. Pastikan unit kerja memiliki lingkup yang jelas: crate, modul, bagian spesifikasi, dan kriteria selesai.
4. Jika lingkup belum jelas, berhenti dan tanyakan.

### 11.2 Urutan Implementasi

1. Tulis pengujian yang merepresentasikan aturan spesifikasi. Jalankan dan pastikan gagal karena belum ada implementasi.
2. Implementasikan perilaku minimum yang memenuhi pengujian.
3. Refaktor tanpa mengubah perilaku, dengan pengujian tetap hijau.
4. Tambahkan dokumentasi `///` dan pastikan `cargo doc` bersih.
5. Jalankan seluruh quality gate lokal (Bagian 11.4).
6. Tinjau diff sendiri sebelum membuka PR.

### 11.3 Commit dan Branch

- Branch dinamai `<jenis>/<ringkas-kebab-case>`, dengan jenis: `feat`, `fix`, `refactor`, `test`, `bench`, `docs`, `ci`, `chore`.
- Pesan commit mengikuti Conventional Commits:
  ```
  <jenis>(<crate>): <ringkasan imperatif, maksimal 72 karakter>

  <penjelasan opsional: apa dan mengapa>

  Spec: §<nomor bagian>
  ```
- Setiap commit WAJIB dapat dibangun dan lulus `cargo test` pada crate yang disentuh. Commit setengah jadi DILARANG di branch yang akan digabung.
- Commit WAJIB atomik. Dilarang menggabungkan perubahan format dengan perubahan tampilan CLI dalam satu commit.
- History yang dipush ke branch utama WAJIB linear (rebase, bukan merge commit untuk fitur).

### 11.4 Quality Gate Lokal Sebelum PR

Agen WAJIB menjalankan dan memastikan lulus:

```
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo doc --workspace --no-deps
```

Untuk perubahan yang menyentuh format, codec, atau paralelisme, tambahkan:

```
cargo test --test golden
cargo test --test parallel_determinism
```

### 11.5 Pull Request

Setiap PR WAJIB memuat:

- [ ] Tujuan tunggal dan referensi bagian spesifikasi (`Spec: §...`).
- [ ] Daftar perubahan per crate.
- [ ] Pengujian yang ditambahkan dan cara menjalankannya.
- [ ] Konfirmasi quality gate lokal lulus.
- [ ] Dampak terhadap format, golden, dan `FORMAT_VERSION` (ya/tidak, dan alasannya).
- [ ] Dampak terhadap performa, jika ada, disertai data benchmark.
- [ ] Tidak ada `TODO`, `FIXME`, `todo!()`, `unimplemented!()`, atau kode yang dikomentari.
- [ ] Tidak ada dependensi baru tanpa label `dependency`.

PR yang tidak memenuhi daftar di atas WAJIB ditolak dan tidak ditinjau.

### 11.6 Definisi Selesai (Definition of Done)

Sebuah unit kerja dinyatakan selesai jika dan hanya jika:

1. Seluruh perilaku yang diklaim sesuai spesifikasi dan memiliki pengujian.
2. Seluruh quality gate CI lulus (Bagian 12).
3. Tidak ada placeholder, TODO, atau kode mati yang tersisa.
4. Dokumentasi publik lengkap.
5. Reviewer menyetujui.
6. Tidak ada `spec-clarification` yang terkait dan belum terjawab.

---

## 12. Quality Gate CI

Setiap perubahan WAJIB lulus seluruh gate berikut sebelum merge:

1. `cargo fmt --all -- --check`
2. `cargo clippy --workspace --all-targets -- -D warnings`
3. `cargo test --workspace` pada Linux, macOS, dan Windows.
4. Pengujian pada MSRV 1.75.
5. Pengujian golden.
6. Pengujian determinisme paralel.
7. `cargo doc --workspace --no-deps` tanpa peringatan.
8. `cargo llvm-cov` dengan ambang minimum Bagian 8.4.
9. Benchmark smoke.
10. `cargo deny check` dan `cargo audit`.
11. Pemeriksaan tidak adanya pola terlarang (lihat Bagian 13). Dijalankan sebagai skrip CI.

Fuzzing dan benchmark penuh dijalankan terjadwal (nightly dan rilis), bukan pada setiap PR.

Gate tidak boleh dilewati, dilonggarkan, atau dijadikan `allow-failure` tanpa `spec-change`.

---

## 13. Larangan Mutlak

Pola berikut DILARANG di seluruh repositori kecuali pada kode uji dan dengan alasan yang tercatat:

| Pola | Alasan |
|---|---|
| `unsafe` | Dilarang oleh `forbid(unsafe_code)`. |
| `unwrap()`, `expect()`, `panic!()` di library | Masukan tidak tepercaya. |
| `todo!()`, `unimplemented!()`, `// TODO`, `// FIXME` | Implementasi setengah jadi. |
| `#[ignore]` tanpa issue | Pengujian yang dilemahkan secara diam-diam. |
| `rand::`, `thread_rng`, `SystemTime` di jalur format | Non-determinisme. |
| `HashMap` untuk iterasi yang memengaruhi keluaran | Urutan tidak stabil. |
| `println!`, `eprintln!` di library | I/O di luar engine. |
| `static mut`, `lazy_static` mutable | State global. |
| Angka ajaib format di luar `dnaforge-core` | Duplikasi konstanta. |
| Mengubah golden tanpa perubahan format | Merusak kompatibilitas diam-diam. |
| Mengubah nilai konstanta di Bagian 6 | Perlu `spec-change`. |
| Dependensi di luar Bagian 5.2 | Perlu PR `dependency`. |
| Klaim hasil fisik dari simulasi | Pelanggaran Bagian 10. |
| Menonaktifkan gate CI | Merusak kualitas repositori. |

---

## 14. Penanganan Situasi Tidak Biasa

- **Pengujian gagal dan penyebabnya tidak jelas:** jangan ubah pengujian agar lulus. Cari penyebab di implementasi atau spesifikasi terlebih dahulu.
- **Spesifikasi terlihat salah:** jangan memperbaiki diam-diam. Buka `spec-clarification`.
- **Performa buruk:** ukur dulu dengan benchmark. Optimasi tanpa data dilarang.
- **Kegagalan CI yang tidak terkait perubahan:** laporkan dan tandai, jangan mengabaikannya.
- **Perubahan yang terasa perlu tetapi di luar lingkup PR:** buat issue baru, jangan memperluas PR.
- **Ketidakpastian tentang perilaku yang memengaruhi format:** berhenti dan tanyakan. Kesalahan format lebih mahal daripada keterlambatan.

---

## 15. Dokumentasi Repositori

Struktur minimum yang WAJIB ada:

```
docs/SPEC.md          Spesifikasi teknis v1 (sumber kebenaran)
AGENTS.md             Dokumen ini
README.md             Ringkasan, cara build, cara menjalankan, batasan klaim
CHANGELOG.md          Riwayat perubahan per versi, mengikuti Keep a Changelog
rust-toolchain.toml   Versi toolchain
deny.toml             Kebijakan lisensi dan kerentanan
tests/golden/         Berkas dan SHA-256 golden
```

- `README.md` WAJIB diperbarui setiap kali perintah CLI berubah.
- `CHANGELOG.md` WAJIB diperbarui pada setiap PR yang mengubah perilaku yang dapat diamati.
- Dokumentasi yang bertentangan dengan `docs/SPEC.md` WAJIB diperbaiki atau dihapus.

---

## 16. Keamanan dan Kebersihan Repositori

- Dilarang meng-commit berkas hasil (pool, reads, truth, laporan besar). Gunakan `.gitignore` untuk direktori keluaran.
- Dilarang meng-commit kunci, token, atau data pribadi.
- Berkas uji golden yang di-commit WAJIB kecil (maksimal 64 KiB) dan dibangkitkan secara deterministik dari seed.
- Dilarang menambahkan berkas biner besar tanpa persetujuan.

---

## 17. Ringkasan Satu Halaman

1. Baca `docs/SPEC.md`. Spesifikasi adalah hukum.
2. Ambiguitas: berhenti dan buka `spec-clarification`.
3. Tulis pengujian dulu, lalu implementasi, lalu refaktor.
4. Tidak ada setengah jadi: tidak ada TODO, stub, atau placeholder di kode yang digabung.
5. Determinisme dijaga di setiap jalur format.
6. Library tidak panik, tidak `unsafe`, tidak I/O.
7. Ubah format berarti naikkan versi, perbarui golden, dan jelaskan dampaknya.
8. Dependensi hanya dari daftar izin.
9. Klaim hasil wajib berasal dari data yang dapat direproduksi dan selalu dinyatakan sebagai simulasi.
10. Quality gate tidak boleh dilewati. Jika gagal, perbaiki, jangan longgarkan.
