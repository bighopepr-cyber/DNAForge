```markdown
# AGENTS.md — Aturan Pengembangan DNAForge

Dokumen ini mengatur perilaku setiap agen (manusia maupun AI) yang mengubah repositori DNAForge. Dokumen ini berlaku bersama `SPEC.md` (spesifikasi teknis v1). Jika ada pertentangan, urutan prioritas adalah: spesifikasi, lalu dokumen ini, lalu preferensi pribadi.

## 1. Prinsip Utama

1. **Spesifikasi adalah sumber kebenaran.** Setiap perubahan perilaku harus dapat ditelusuri ke bagian spesifikasi. Jika spesifikasi tidak menjelaskan suatu hal, berhenti dan ajukan pertanyaan. Jangan menebak dan jangan mengisi celah dengan keputusan sendiri.
2. **Tidak ada implementasi setengah jadi.** Setiap perubahan yang di-merge harus lengkap, teruji, dan konsisten dengan seluruh repositori.
3. **Kebenaran lebih penting daripada kecepatan.** Output yang salah dan tampak benar lebih berbahaya daripada output yang gagal dengan jelas.
4. **Determinisme adalah fitur inti.** Keluaran encode dan decode harus byte-identik untuk masukan dan parameter yang sama.
5. **Transparansi.** Setiap klaim tentang performa, ketahanan, atau akurasi harus didukung data yang dapat direproduksi.

## 2. Larangan Mutlak

Agen DILARANG melakukan hal berikut, tanpa pengecualian:

- Meninggalkan kode dalam keadaan setengah jadi: `todo!()`, `unimplemented!()`, `unreachable!()` sebagai pengganti logika nyata, `// TODO`, `// FIXME`, `// XXX`, fungsi stub yang mengembalikan nilai palsu, atau placeholder seperti `Ok(Vec::new())` tanpa logika.
- Mengomentari kode yang tidak dipakai sebagai cara menyimpannya. Hapus, atau pertahankan dalam riwayat git.
- Menonaktifkan, melewati, atau melemahkan pengujian: `#[ignore]`, `#[should_panic]` tanpa alasan spesifikasi, penghapusan assertion, atau pengurangan jumlah kasus tanpa persetujuan.
- Memperbarui nilai golden (Bagian 23.3 spesifikasi) tanpa perubahan `format_version` atau revisi spesifikasi yang sah.
- Menambahkan `#[allow(...)]` untuk membungkam lint tanpa komentar pembenaran satu baris di baris yang sama.
- Menambahkan `unsafe`, `unwrap()`, `expect()`, atau `panic!()` pada jalur library yang memproses masukan pengguna atau data kanal.
- Menambahkan dependensi di luar daftar Bagian 5.2 spesifikasi tanpa revisi spesifikasi.
- Menggunakan `rand`, `SystemTime`, `Instant`, atau sumber nondeterministik lain pada jalur format encode dan decode.
- Menggunakan `HashMap` atau `HashSet` dengan iterasi yang memengaruhi keluaran tanpa pengurutan eksplisit.
- Melakukan I/O berkas atau jaringan di crate selain `dnaforge-engine`, `dnaforge-bench`, dan `dnaforge-cli`.
- Mengubah perilaku yang sudah ada tanpa menyebutkannya secara eksplisit di deskripsi perubahan.
- Melakukan force push ke cabang utama, menghapus riwayat, atau me-merge dengan CI merah.
- Menyembunyikan kegagalan. Kegagalan decode, konsensus, atau verifikasi WAJIB tercatat di laporan sesuai Bagian 26.

## 3. Alur Kerja Wajib

Setiap tugas WAJIB mengikuti urutan berikut:

### 3.1 Sebelum menulis kode

1. Baca bagian spesifikasi yang relevan dengan tugas. Sebutkan nomor bagiannya.
2. Periksa konstanta di Bagian 6 dan aturan di Bagian 4.2 yang berlaku.
3. Cek kode yang sudah ada di crate terkait. Jangan menduplikasi fungsi yang sudah tersedia.
4. Tentukan crate yang terdampak dan pastikan arah dependensi tetap asiklik.
5. Jika tugas tidak jelas, atau spesifikasi ambigu atau bertentangan, berhenti dan ajukan pertanyaan spesifik. Jangan lanjut dengan asumsi tersembunyi.

### 3.2 Saat menulis kode

1. Implementasikan satu unit logis per perubahan. Hindari perubahan yang mencampur refactor, fitur baru, dan perbaikan bug.
2. Tulis pengujian bersamaan dengan kode, bukan sesudahnya. Untuk perbaikan bug, tulis pengujian yang gagal terlebih dahulu.
3. Setiap fungsi publik mendapat dokumentasi `///` yang menyebut prasyarat, nilai kembali, dan kondisi error (Bagian 4.2 poin 6).
4. Error ditangani dengan enum `thiserror` di crate masing-masing. Konversi antarcrate dilakukan secara eksplisit.
5. Tidak ada angka ajaib. Setiap konstanta format didefinisikan sekali di `dnaforge-core` dengan nama yang sesuai spesifikasi.

### 3.3 Sebelum menyerahkan perubahan

Jalankan seluruh pemeriksaan berikut secara lokal. Perubahan yang gagal pada salah satu langkah tidak boleh diserahkan:

- `cargo fmt --all -- --check`
- `cargo clippy --workspace --all-targets -- -D warnings`
- `cargo test --workspace`
- `cargo doc --workspace --no-deps` tanpa peringatan
- Pengujian golden dan determinisme paralel jika perubahan menyentuh codec, ecc, fragment, atau engine

### 3.4 Setelah perubahan

1. Perbarui dokumentasi yang terdampak dalam perubahan yang sama, bukan di perubahan terpisah.
2. Jika perubahan memengaruhi metrik atau benchmark, jalankan benchmark terkait dan laporkan hasil dengan kondisi lingkungannya.
3. Isi deskripsi perubahan sesuai Bagian 5.

## 4. Definisi Selesai (Definition of Done)

Sebuah tugas dinyatakan selesai jika dan hanya jika:

- [ ] Perilaku sesuai dengan bagian spesifikasi yang dirujuk.
- [ ] Tidak ada `TODO`, `FIXME`, `todo!()`, `unimplemented!()`, atau stub.
- [ ] Pengujian unit untuk kasus normal, kasus batas, dan kasus error sudah ada dan lulus.
- [ ] Untuk logika format, ada property test atau pengujian round-trip.
- [ ] Seluruh quality gate lokal (Bagian 3.3) lulus.
- [ ] Cakupan kode pada crate terkait tidak turun di bawah ambang Bagian 27 spesifikasi.
- [ ] Tidak ada perubahan perilaku tersembunyi.
- [ ] Dokumentasi publik dan `README` yang terdampak sudah diperbarui.
- [ ] Tidak ada dependensi baru tanpa revisi spesifikasi.
- [ ] Tidak ada peningkatan peak memori atau penurunan throughput lebih dari 10% tanpa penjelasan.

## 5. Konvensi Git dan Pull Request

### 5.1 Cabang

- Cabang utama `main` selalu dalam keadaan hijau (semua CI lulus).
- Pengembangan dilakukan di cabang `feat/<ringkas>`, `fix/<ringkas>`, `refactor/<ringkas>`, `test/<ringkas>`, `docs/<ringkas>`, atau `chore/<ringkas>`.
- Cabang tidak boleh hidup lebih dari beberapa hari. Pecah pekerjaan besar menjadi perubahan kecil yang masing-masing lengkap.

### 5.2 Commit

Format commit mengikuti Conventional Commits:

```

<tipe>(<crate>): <ringkasan dalam bentuk perintah, maksimal 72 karakter>

<badan: apa dan mengapa, merujuk bagian spesifikasi>

Spec: Bagian <nomor>
```

Tipe yang diizinkan: `feat`, `fix`, `refactor`, `test`, `bench`, `docs`, `perf`, `chore`, `ci`.

Contoh: `fix(recovery): samakan batas ambang jarak edit dengan Bagian 16.2`

Setiap commit harus membangun dan lulus pengujian sendiri. Commit yang merusak build di tengah riwayat tidak diizinkan.

### 5.3 Pull Request

Setiap PR WAJIB memuat:

1. Rujukan bagian spesifikasi yang diimplementasikan.
2. Ringkasan perubahan perilaku yang dapat diamati.
3. Daftar pengujian yang ditambahkan atau diubah beserta alasannya.
4. Dampak terhadap format, nilai golden, dan metrik. Tulis "tidak ada" jika memang tidak ada.
5. Hasil benchmark jika menyentuh jalur panas (GF, CRC, keystream, edit distance, konsensus).
6. Konfirmasi bahwa seluruh quality gate lulus.

PR yang mengubah `format_version`, konstanta di Bagian 6, atau nilai golden WAJIB disertai revisi spesifikasi yang sudah di-merge lebih dulu.

### 5.4 Review

- Setiap PR membutuhkan minimal satu review yang menyetujui.
- Reviewer memeriksa kepatuhan terhadap spesifikasi, bukan hanya kompilasi.
- Komentar review yang belum diselesaikan harus diselesaikan atau dijawab sebelum merge.

## 6. Perubahan Format dan Spesifikasi

Format v1 bersifat tetap. Aturan perubahan:

1. Perubahan yang memengaruhi byte pool, oligo, manifest, pemetaan nukleotida, whitening, CRC, GF, matriks Cauchy, atau seed RNG adalah **perubahan format**.
2. Perubahan format WAJIB:
   - Diajukan sebagai revisi spesifikasi tersendiri dengan alasan dan dampaknya.
   - Menaikkan `FORMAT_VERSION` jika format keluaran berubah.
   - Memperbarui nilai golden dengan catatan di riwayat commit.
   - Menjaga kemampuan membaca format sebelumnya, atau secara eksplisit menolaknya dengan error yang jelas.
3. Perubahan yang tidak memengaruhi byte keluaran (refactor, optimasi, logging) WAJIB menjaga seluruh nilai golden tetap sama. Nilai golden yang berubah pada perubahan semacam ini menandakan bug.
4. Optimasi (termasuk SIMD) hanya boleh diterima jika keluarannya bit-identik dengan implementasi skalar, dan dibuktikan dengan pengujian perbandingan.

## 7. Standar Kualitas Kode

### 7.1 Umum

- `rustfmt` dengan konfigurasi repositori. Tidak ada format manual yang menyimpang.
- Clippy dengan `-D warnings`. Peringatan tidak boleh ditekan secara massal.
- Fungsi lebih dari sekitar 60 baris atau dengan lebih dari empat tingkat nesting perlu dipecah atau dijustifikasi.
- Nama harus deskriptif dan konsisten dengan istilah di Bagian 3 spesifikasi (misalnya `oligo`, `pool`, `slot`, `seq`, `variant`).
- Komentar menjelaskan mengapa, bukan apa. Komentar yang hanya mengulang kode dilarang.

### 7.2 Keamanan dan keandalan

- `#![forbid(unsafe_code)]` pada setiap crate.
- Aritmetika pada data masukan memakai operasi yang eksplisit: `checked_*`, `wrapping_*`, atau `saturating_*` sesuai maksud, bukan operasi biasa yang dapat overflow.
- Alokasi berdasarkan nilai dari masukan (misalnya ukuran berkas di manifest) WAJIB divalidasi dengan batas sebelum dialokasikan.
- Semua indeks array yang berasal dari masukan diperiksa batasnya.

### 7.3 Struktur modul

- Satu tanggung jawab per modul. Modul yang melebihi sekitar 800 baris perlu dipecah.
- Visibilitas seminimal mungkin. Gunakan `pub(crate)` sebelum `pub`.
- Tipe data format (oligo, header, manifest) memakai tipe kuat, bukan `Vec<u8>` tanpa makna.

## 8. Strategi Pengujian

### 8.1 Hierarki pengujian

| Lapisan | Lokasi | Wajib untuk |
|---|---|---|
| Unit | `#[cfg(test)]` di modul | Semua fungsi publik dan internal kritis |
| Property | `proptest` | Round-trip, invarian aljabar, codec |
| Integrasi | `tests/` tiap crate dan `dnaforge-engine` | Pipeline encode dan decode |
| Golden | `tests/golden/` | Stabilitas format |
| Determinisme paralel | `tests/` engine | Keluaran sama untuk thread berbeda |
| Fuzz | `fuzz/` | Parser dan decoder |
| Benchmark | `benches/` | Performa, bukan kebenaran |

### 8.2 Aturan pengujian

- Pengujian harus deterministik. Gunakan seed tetap. Pengujian yang kadang gagal dianggap bug, bukan noise.
- Pengujian tidak boleh bergantung pada urutan eksekusi, waktu, atau jaringan.
- Setiap bug yang diperbaiki WAJIB memiliki pengujian regresi yang gagal sebelum perbaikan.
- Pengujian harus memeriksa nilai spesifik, bukan hanya `is_ok()`. Error diperiksa variannya.
- Assertion pada data biner menampilkan perbedaan yang berguna jika gagal (gunakan pesan atau fungsi pembanding yang jelas).
- Pengujian yang lambat (lebih dari beberapa detik) ditandai dan dijalankan di tier yang sesuai, bukan dilewati.

### 8.3 Ambang cakupan

Sesuai Bagian 27 spesifikasi: minimal 90% garis untuk core, codec, ecc, dan fragment; minimal 80% untuk recovery dan engine. Penurunan cakupan di bawah ambang memblokir merge. Cakupan tinggi dengan pengujian lemah tidak dianggap memenuhi syarat. Pengujian harus memverifikasi perilaku.

## 9. Disiplin Benchmark dan Metrik

- Benchmark mikro menggunakan Criterion dengan `black_box` dan minimal 100 sampel.
- Perbandingan performa hanya valid pada lingkungan yang sama. Laporkan CPU, jumlah core, RAM, OS, dan versi rustc.
- Jangan mengklaim peningkatan performa dari satu kali pengukuran. Gunakan median dan rentang interkuartil dari minimal 10 pengulangan.
- Proporsi dilaporkan dengan interval kepercayaan Wilson 95%.
- Angka di README, laporan, dan PR harus berasal dari eksekusi yang tercatat, bukan perkiraan.
- Regresi lebih dari 10% pada throughput atau memori memblokir merge (Bagian 24.4 spesifikasi).
- Jangan menulis klaim tentang sintesis, penyimpanan fisik, atau sekuensing nyata. Semua hasil berlaku untuk model kanal simulasi.

## 10. Pengelolaan Dependensi

- Hanya dependensi yang terdaftar di Bagian 5.2 spesifikasi yang boleh ditambahkan.
- Versi dependensi dikunci melalui `Cargo.lock`. Pembaruan dilakukan dalam PR tersendiri dengan catatan perubahan.
- `cargo deny` dan `cargo audit` wajib lulus. Kerentanan tingkat tinggi memblokir rilis.
- Dependensi transitif yang tidak diperlukan harus diminimalkan. Fitur yang tidak dipakai harus dinonaktifkan.

## 11. Dokumentasi

- Spesifikasi (`SPEC.md`) adalah dokumen normatif. Perubahannya hanya melalui PR revisi yang disetujui.
- `README.md` berisi cara membangun, menjalankan, dan contoh CLI yang sudah diverifikasi. Contoh di README harus dapat dijalankan dan diuji.
- Setiap bagian publik crate memiliki dokumentasi `///`. `cargo doc` tanpa peringatan adalah syarat.
- Hasil eksperimen disimpan dalam format JSONL dengan metadata lengkap (Bagian 19.4 spesifikasi).
- Dokumen baru tidak dibuat untuk menggantikan spesifikasi. Jika ada informasi yang perlu dicatat, tambahkan ke bagian yang relevan dalam spesifikasi atau dokumentasi kode.

## 12. Penanganan Ketidakpastian dan Konflik

Agen WAJIB berhenti dan bertanya, bukan menebak, dalam situasi berikut:

- Spesifikasi tidak mendefinisikan perilaku yang dibutuhkan.
- Dua bagian spesifikasi saling bertentangan.
- Implementasi yang benar membutuhkan perubahan format.
- Pengujian yang sudah ada gagal dan penyebabnya tidak jelas. Jangan ubah pengujian agar lulus sebelum memahami penyebabnya.
- Perubahan akan melampaui batas dependensi, crate, atau ruang lingkup tugas.

Format pertanyaan: sebutkan bagian spesifikasi yang relevan, jelaskan ambiguitasnya, dan usulkan minimal dua opsi dengan konsekuensinya.

## 13. Format Pelaporan Agen

Setiap penyelesaian tugas WAJIB dilaporkan dengan struktur:

1. **Yang diubah:** daftar file dan crate terdampak.
2. **Rujukan spesifikasi:** nomor bagian yang diimplementasikan.
3. **Hasil verifikasi:** status setiap quality gate lokal (lulus atau gagal, dengan output ringkas jika gagal).
4. **Dampak:** perubahan format, golden, metrik, atau memori, atau "tidak ada".
5. **Batasan dan asumsi:** hanya yang benar-benar ada. Jika tidak ada, tulis "tidak ada".

Laporan dilarang mengklaim selesai jika ada pemeriksaan yang belum dijalankan. Tulis secara eksplisit bagian mana yang belum diverifikasi.

## 14. Keputusan Arsitektur

- Perubahan arsitektur (menambah, menghapus, atau memindahkan crate; mengubah arah dependensi; mengubah tipe publik lintas crate) WAJIB melalui revisi spesifikasi Bagian 4.
- Perubahan internal yang tidak terlihat dari luar crate boleh dilakukan melalui PR biasa, selama aturan Bagian 4.2 tetap terpenuhi.
- Abstraksi baru hanya ditambahkan jika sudah ada minimal dua penggunaan nyata. Jangan membuat abstraksi untuk kebutuhan yang belum ada.

## 15. Larangan Akhir

Agen tidak boleh menyelesaikan tugas dengan cara yang membuat tes lulus tetapi perilaku tidak benar. Jika pengujian dan spesifikasi bertentangan, spesifikasi yang menang, dan pengujian diperbaiki melalui PR yang menjelaskan alasannya.
```
