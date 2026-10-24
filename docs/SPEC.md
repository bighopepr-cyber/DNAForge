# Spesifikasi Teknis DNAForge v1

## 1. Status Normatif

Dokumen ini adalah satu-satunya sumber kebenaran untuk implementasi DNAForge versi 1. Kata **WAJIB** berarti persyaratan mutlak. Kata **DILARANG** berarti tidak boleh dilakukan. Kata **BOLEH** berarti pilihan implementasi yang tidak memengaruhi format atau perilaku yang dapat diamati. Perilaku yang tidak tercantum di sini tidak diizinkan kecuali ditetapkan secara eksplisit oleh revisi spesifikasi. Jika terdapat pertentangan antara kode dan spesifikasi, spesifikasi yang berlaku.

## 2. Ruang Lingkup

**Termasuk dalam cakupan:**
- Encoding berkas biner menjadi kumpulan oligo nukleotida (pool) dan decoding kembali menjadi berkas asli.
- Simulasi kanal deterministik dengan substitusi, insersi, delesi, kehilangan oligo, dan duplikasi.
- Pemrosesan read berisik: pengelompokan (clustering), konsensus, dan decoding oligo.
- Koreksi erasure berbasis Reed-Solomon untuk pemulihan fragmen yang hilang atau rusak.
- Verifikasi integritas terhadap hash SHA-256 berkas sumber.
- Command-line interface (CLI), pengujian, benchmark, dan eksperimen ketahanan dengan metrik yang terdefinisi.

**Di luar cakupan:**
- Sintesis fisik, penyimpanan laboratorium, dan sekuensing nyata.
- Desain primer dan penambahan adapter biologis.
- Kompresi dan enkripsi. Input disimpan apa adanya.
- Kode koreksi selain Reed-Solomon erasure.
- Implementasi SIMD. Jika di kemudian hari ditambahkan, keluarannya WAJIB bit-identik dengan implementasi skalar.

## 3. Terminologi

| Istilah | Definisi |
|---|---|
| Oligo (fragmen) | Satu untai DNA tepat 192 nukleotida yang membawa satu unit data. |
| Pool | Kumpulan oligo keluaran encoder, disimpan sebagai berkas FASTA. |
| Read | Untai hasil pembacaan kanal, panjangnya boleh berbeda dari 192. |
| Klaster | Himpunan read yang diyakini berasal dari oligo yang sama. |
| Konsensus | Untai tunggal hasil voting dari satu klaster. |
| Payload | 30 byte data yang dibawa oleh satu oligo. |
| Grup | Himpunan K fragmen data dan M fragmen paritas yang dikodekan bersama. |
| Slot | Posisi fragmen dalam grup, indeks 0 sampai K+M−1. |
| Seq | Nomor urut global 32-bit sebuah oligo dalam satu berkas. |
| file_id | 6 byte pertama SHA-256 berkas sumber. Identitas berkas dalam pool. |
| Varian | Byte 0 sampai 255 yang dipilih encoder agar oligo lolos constraint. |
| Virtual fragment | Slot data kosong pada grup terakhir yang diisi nol dan tidak dipancarkan. |
| Erasure | Fragmen yang diketahui hilang atau tidak valid (CRC gagal). |

## 4. Arsitektur

### 4.1 Struktur Crate

Repositori berbentuk workspace Cargo dengan tujuh crate library dan satu crate binary.

| Crate | Tanggung jawab | Dependensi internal |
|---|---|---|
| `dnaforge-core` | Konstanta format, tipe error dasar, CRC-16, SHA-256 wrapper, FNV-1a 64, SplitMix64, Xoshiro256**, konfigurasi. | — |
| `dnaforge-codec` | Tipe nukleotida, pemetaan 2-bit, whitening, struktur oligo, parsing dan pembuatan oligo, constraint engine (GC, homopolymer). | core |
| `dnaforge-ecc` | Aritmetika GF(2^8), matriks Cauchy, encoding dan decoding erasure Reed-Solomon. | core |
| `dnaforge-fragment` | Manifest, tata letak seq, pembagian berkas menjadi blok, perencanaan grup. | core, ecc |
| `dnaforge-channel` | Simulator kanal deterministik. | core |
| `dnaforge-recovery` | Parsing read, clustering, konsensus, decoding read menjadi oligo tervalidasi. | core, codec |
| `dnaforge-engine` | Orkestrasi encode dan decode, I/O streaming, verifikasi, pelaporan. | semua crate di atas |
| `dnaforge-bench` | Harness benchmark, runner eksperimen, pengumpulan metrik. | engine, channel |
| `dnaforge-cli` | Binary `dnaforge`. | engine, channel, bench |

### 4.2 Aturan Arsitektur

1. Graf dependensi WAJIB asiklik. Crate yang lebih rendah DILARANG mengimpor crate yang lebih tinggi.
2. Crate library selain `dnaforge-engine` DILARANG melakukan I/O berkas atau jaringan. Fungsi bersifat murni terhadap masukannya.
3. Tidak boleh ada state global yang dapat berubah. Konstanta boleh berupa `const` atau `static` yang tidak dapat berubah.
4. Setiap crate WAJIB memiliki `#![forbid(unsafe_code)]`.
5. Setiap crate WAJIB mendefinisikan enum error sendiri dengan `thiserror`. Error antarcrate dikonversi secara eksplisit.
6. Fungsi publik WAJIB didokumentasikan dengan `///` yang menyebut prasyarat dan kondisi error.
7. Kode library DILARANG memakai `unwrap()`, `expect()`, atau `panic!()` pada data yang berasal dari masukan pengguna atau kanal.

## 5. Lingkungan dan Dependensi

### 5.1 Toolchain

- Rust stable, versi dikunci melalui `rust-toolchain.toml`.
- MSRV: Rust 1.75.
- Edition: 2021.
- Target wajib: Linux x86_64, macOS (arm64 dan x86_64), Windows x86_64.
- Profil `release`: `opt-level = 3`, `lto = "thin"`, `codegen-units = 1`.

### 5.2 Dependensi Eksternal yang Diizinkan

| Crate | Dipakai oleh | Keterangan |
|---|---|---|
| `sha2` | core | SHA-256. |
| `thiserror` | semua library | Enum error. |
| `clap` (derive) | cli | Parsing argumen. |
| `rayon` | engine, recovery | Paralelisme deterministik, di belakang fitur `parallel` (aktif secara default). |
| `serde`, `serde_json` | bench | Penulisan laporan JSON. |
| `criterion` | bench (dev) | Benchmark mikro. |
| `proptest` | semua (dev) | Property-based testing. |
| `tempfile` | engine (dev) | Berkas sementara dalam pengujian. |
| `cargo-llvm-cov` | CI | Pengukuran cakupan, alat, bukan dependensi. |

Jalur format (encode dan decode) DILARANG menggunakan `rand` atau RNG eksternal lain. Semua pembangkit acak berasal dari `dnaforge-core`.

### 5.3 Pemrosesan Paralel

Fitur `parallel` mengaktifkan Rayon untuk encoding per grup, konsensus per klaster, dan decoding RS per grup. Urutan hasil WAJIB dikumpulkan berdasarkan indeks asal sehingga keluaran identik untuk jumlah thread apa pun.

## 6. Konstanta Format v1

| Konstanta | Nilai | Keterangan |
|---|---|---|
| `OLIGO_BYTES` | 48 | Ukuran biner satu oligo sebelum pemetaan nukleotida. |
| `OLIGO_NT` | 192 | Panjang oligo dalam nukleotida (48 × 4). |
| `HEADER_BYTES` | 16 | Ukuran header. |
| `PAYLOAD_BYTES` | 30 | Kapasitas payload per oligo. |
| `CRC_BYTES` | 2 | CRC-16 di akhir oligo. |
| `MAGIC` | `0x44 0x46` (ASCII "DF") | Penanda awal header. |
| `FORMAT_VERSION` | 1 | Versi format. |
| `KIND_DATA` | 0 | Fragmen data. |
| `KIND_PARITY` | 1 | Fragmen paritas. |
| `KIND_MANIFEST` | 2 | Manifest. |
| `MANIFEST_COPIES` | 3 | Jumlah salinan manifest. |
| `SCHEME_ID` | 1 | Identifikasi skema encoding. |
| `GC_MIN` / `GC_MAX` | 77 / 115 | Jumlah G+C yang diizinkan (40%–60% dari 192, dibulatkan ke dalam). |
| `MAX_HOMOPOLYMER` | 3 | Run identik terpanjang yang diizinkan. |
| `MAX_VARIANTS` | 256 | Varian 0 sampai 255. |
| `DEFAULT_K` | 10 | Fragmen data per grup. |
| `DEFAULT_M` | 4 | Fragmen paritas per grup. |
| `KMER_LEN` | 12 | Panjang k-mer untuk indeks klaster. |
| `BAND_WIDTH` | 24 | Lebar pita DP untuk penyelarasan. |
| `CLUSTER_RATIO` | 0.30 | Ambang jarak edit relatif terhadap panjang representatif. |
| `MIN_SHARED_KMERS` | 8 | Minimum k-mer bersama untuk kandidat klaster. |
| `MAX_CANDIDATES` | 16 | Kandidat klaster maksimum yang diverifikasi per read. |
| `CONSENSUS_ROUNDS` | 2 | Jumlah putaran konsensus. |
| `FILE_ID_BYTES` | 6 | Panjang file_id. |
| `MANIFEST_HASH_BYTES` | 16 | Panjang prefiks SHA-256 di manifest. |
| `MAX_SEQ` | 2^32 − 1 | Nomor seq maksimum. |

Batas ukuran berkas adalah jumlah fragmen ≤ 2^32 − 1, sehingga ukuran maksimum sekitar 128 GiB. Encoder WAJIB menolak berkas yang melampaui batas ini dengan error `FileTooLarge`.

## 7. Pemetaan Nukleotida dan Pengemasan

Pemetaan 2-bit tetap:

| Bit | Nukleotida |
|---|---|
| `00` | A |
| `01` | C |
| `10` | G |
| `11` | T |

Pengemasan: setiap byte diproses dari bit tertinggi ke terendah. Setiap pasangan bit menghasilkan satu nukleotida, dengan pasangan pertama (bit 7 dan 6) menjadi nukleotida pertama. Satu byte menghasilkan empat nukleotida. Oligo 48 byte menghasilkan tepat 192 nukleotida.

Operasi `pack(bytes) -> String` dan `unpack(String) -> Result<Vec<u8>>` WAJIB bersifat bijektif untuk input yang valid. Unpack WAJIB menolak karakter selain `ACGT` (huruf besar) dan panjang yang tidak kelipatan 4.

## 8. Struktur Oligo

### 8.1 Layout Biner

Oligo terdiri dari 48 byte dengan urutan berikut (indeks berbasis nol, big-endian untuk field multi-byte):

| Offset | Panjang | Field | Ditulis dalam | Keterangan |
|---|---|---|---|---|
| 0 | 2 | `magic` | Plain | `0x44 0x46` |
| 2 | 1 | `version` | Plain | Selalu 1 |
| 3 | 1 | `kind` | Plain | 0, 1, atau 2 |
| 4 | 6 | `file_id` | Plain | 6 byte pertama SHA-256 berkas sumber |
| 10 | 4 | `seq` | Plain | u32 BE |
| 14 | 1 | `payload_len` | Plain | 1–30 untuk data; 30 untuk paritas dan manifest |
| 15 | 1 | `variant` | Plain | Varian 0–255 |
| 16 | 30 | `payload` | Whitened | Data, zero-padded |
| 46 | 2 | `crc16` | Whitened | CRC-16/CCITT-FALSE atas byte 0–45 (plain) |

Header (byte 0–15) disimpan tanpa whitening agar dapat dibaca sebelum keystream diketahui.

### 8.2 CRC-16

Algoritma: CRC-16/CCITT-FALSE. Polinom 0x1021, init 0xFFFF, tanpa refleksi input maupun output, xorout 0. Nilai uji untuk ASCII `"123456789"` adalah `0x29B1`. Nilai CRC disimpan big-endian.

CRC dihitung atas 46 byte plaintext (header plus payload) sebelum whitening. Penerima WAJIB menghitung ulang CRC setelah unwhitening dan membandingkannya.

### 8.3 Whitening

Whitening memakai XOR dengan keystream yang diturunkan dari parameter oligo.

Seed:
```
state0 = FNV1a64( file_id[0..6] ‖ seq_be[0..4] ‖ [variant] )   // 11 byte masukan
```

Keystream: SplitMix64 dengan state awal `state0`. Setiap pemanggilan menghasilkan u64 `w_k`. Keystream diambil sebagai `w_0.to_le_bytes() ‖ w_1.to_le_bytes() ‖ w_2.to_le_bytes() ‖ w_3.to_le_bytes()` dan hanya 32 byte pertama yang dipakai untuk byte 16 sampai 47.

Whitening bersifat involutif: menerapkannya dua kali menghasilkan data asli. Whitening WAJIB diterapkan pada byte 16 sampai 47 setelah CRC dihitung.

### 8.4 Algoritma Dasar Generator Acak

**FNV-1a 64-bit:** offset basis `0xcbf29ce484222325`, prima `0x100000001b3`. Untuk setiap byte: XOR dengan byte, lalu kalikan dengan prima (mod 2^64).

**SplitMix64:**
```
state = state + 0x9E3779B97F4A7C15
z = state
z = (z ^ (z >> 30)) * 0xBF58476D1CE4E5B9
z = (z ^ (z >> 27)) * 0x94D049BB133111EB
return z ^ (z >> 31)
```

**Xoshiro256\*\*:** state 256-bit diinisialisasi dengan empat keluaran SplitMix64 berturut-turut dari seed 64-bit.

**Bilangan real uniform [0,1):** `(next_u64() >> 11) as f64 * 2^-53`.

**Indeks uniform [0, n):** `((next_u64() as u128 * n as u128) >> 64) as usize`. Metode ini DILARANG diganti dengan modulo.

## 9. Constraint Engine

Sebuah oligo 192 nukleotida dinyatakan **valid secara constraint** jika dan hanya jika seluruh kondisi berikut terpenuhi:

1. Panjang tepat 192 nukleotida.
2. Jumlah G dan C berada dalam rentang `[77, 115]` (inklusif).
3. Tidak ada run nukleotida identik dengan panjang lebih dari 3 (tidak ada empat huruf sama berurutan).

Constraint engine menyediakan fungsi `check(oligo) -> Result<(), ConstraintViolation>` dengan varian error `GcOutOfRange { count }`, `Homopolymer { run, position }`, dan `BadLength`. Constraint TIDAK diterapkan pada read hasil kanal, hanya pada oligo hasil encoding dan konsensus yang diterima.

## 10. Pembuatan Oligo

Fungsi `build_oligo(fields: OligoFields) -> Result<String, EncodeError>` dengan `OligoFields { file_id, kind, seq, payload_len, payload: [u8; 30] }`.

Algoritma:
1. Untuk `v` dari 0 sampai 255 (inklusif), secara berurutan:
   a. Susun header 16 byte dengan `variant = v`.
   b. Hitung CRC-16 atas 46 byte pertama.
   c. Terapkan whitening pada byte 16–47 dengan keystream yang bergantung pada `v`.
   d. Petakan 48 byte menjadi 192 nukleotida.
   e. Jalankan constraint engine. Jika lolos, kembalikan hasil.
2. Jika seluruh 256 varian gagal, kembalikan `EncodeError::ConstraintUnsatisfiable { seq }`.

Varian pertama yang lolos WAJIB dipilih (urutan menaik). Dengan demikian hasil bersifat deterministik.

## 11. Parsing dan Validasi Oligo

Fungsi `parse_oligo(nt: &[u8]) -> Result<OligoRecord, ParseError>` WAJIB melakukan validasi berikut secara berurutan. Kegagalan pada langkah mana pun menghasilkan error dan oligo dibuang.

1. Panjang tepat 192 dan alfabet `ACGT`.
2. Unpack menjadi 48 byte.
3. `magic == [0x44, 0x46]`.
4. `version == 1`.
5. `kind ∈ {0, 1, 2}`.
6. `payload_len`: 1–30 untuk kind 0; tepat 30 untuk kind 1 dan 2.
7. Unwhitening byte 16–47 dengan keystream yang diturunkan dari `file_id`, `seq`, dan `variant` pada header.
8. CRC-16 atas byte 0–45 harus sama dengan byte 46–47.
9. Untuk kind 2: `payload` byte 27–29 harus nol.

`OligoRecord` berisi `file_id`, `kind`, `seq`, `payload_len`, dan `payload[30]`. Hasil parsing WAJIB tidak memuat byte padding sebagai data.

## 12. Manifest dan Tata Letak

### 12.1 Isi Manifest (payload kind 2)

| Offset payload | Panjang | Field |
|---|---|---|
| 0 | 8 | `file_len` (u64 BE, jumlah byte berkas sumber) |
| 8 | 16 | `sha256_prefix` (16 byte pertama SHA-256 berkas) |
| 24 | 1 | `K` |
| 25 | 1 | `M` |
| 26 | 1 | `scheme_id` = 1 |
| 27 | 3 | Reserved = 0 |

### 12.2 Tiga Salinan Manifest

Encoder WAJIB memancarkan tiga oligo manifest dengan `seq = 0`, `1`, dan `2`. Keempat field payload identik pada ketiganya. Setiap salinan memakai whitening dan varian sendiri sesuai Bagian 10.

### 12.3 Jumlah Fragmen

Diberikan `S = file_len`:
- `N = ceil(S / 30)` adalah jumlah fragmen data nyata. Jika `S = 0`, maka `N = 0`.
- `G = ceil(N / K)` adalah jumlah grup. Jika `N = 0`, maka `G = 0`.

### 12.4 Pemetaan Seq

Data indeks `i` (0 ≤ i < N):
```
g = i / K
j = i % K
seq = 3 + g * (K + M) + j
```

Paritas grup `g`, indeks `p` (0 ≤ p < M):
```
seq = 3 + g * (K + M) + K + p
```

Panjang payload fragmen data indeks `i`: 30 untuk semua `i < N−1`, dan `S − (N−1)·30` untuk `i = N−1`. Panjang payload paritas selalu 30.

Slot virtual (indeks data `i ≥ N` pada grup terakhir) TIDAK dipancarkan dan seq-nya dilewati. Penerima menentukan slot virtual dari `S`, `K`, dan `N`.

Validasi: `3 + G·(K+M) − 1 ≤ MAX_SEQ`. Jika tidak, encoder WAJIB menolak dengan `FileTooLarge`.

### 12.5 Parameter K dan M

- `1 ≤ K ≤ 128`.
- `0 ≤ M ≤ 64`.
- `K + M ≤ 256`.

Nilai default `K = 10`, `M = 4`. Nilai K dan M disimpan di manifest sehingga decoder tidak memerlukan konfigurasi eksternal.

## 13. Error Correction: Reed-Solomon Erasure

### 13.1 Lapangan GF(2^8)

- Polinom primitif: `x^8 + x^4 + x^3 + x^2 + 1` (0x11D).
- Penjumlahan dan pengurangan: XOR.
- Perkalian: melalui tabel `exp` dan `log` dengan generator `α = 2`. Tabel dihitung saat kompilasi dengan `const fn`.
- Invers: `inv(a) = exp[255 − log[a]]` untuk `a ≠ 0`. `inv(0)` WAJIB menghasilkan error.

### 13.2 Matriks Cauchy

Untuk paritas `p` (0 ≤ p < M) dan data `j` (0 ≤ j < K):
```
P[p][j] = inv( (K + p) XOR j )
```

Karena `K + p ≠ j` untuk semua pasangan, setiap entri bernilai tak-nol. Setiap submatriks persegi dari matriks generator sistematik `G = [I_K ; P]` bersifat invertibel. Oleh karena itu kode bersifat MDS: K fragmen mana pun dari K + M cukup untuk rekonstruksi.

### 13.3 Encoding Grup

Untuk setiap grup, untuk setiap kolom byte `c` dari 0 sampai 29:

```
p_c[p] = XOR_{j=0}^{K−1} ( P[p][j] · d_c[j] )
```
`d_c[j]` adalah byte kolom `c` dari payload data slot `j` (nol untuk slot virtual).

### 13.4 Decoding Grup

Input: himpunan indeks slot yang diketahui `Known ⊆ {0..K+M−1}` beserta nilai payload-nya. Slot virtual selalu termasuk dalam `Known` dengan nilai nol.

Langkah:
1. Jika `|Known| < K`, grup dinyatakan **tidak dapat dipulihkan**.
2. Pilih tepat K indeks dari `Known`. Urutan pilihan: slot data terlebih dahulu (menaik), lalu slot paritas (menaik).
3. Bentuk matriks `A` berukuran K×K, dengan baris `i` adalah baris `G[slot_i]` (baris identitas untuk data, baris `P` untuk paritas).
4. Hitung `A^{-1}` dengan eliminasi Gauss-Jordan atas GF(256). Jika singular (tidak seharusnya terjadi), kembalikan error internal.
5. Untuk setiap kolom `c`: `d_c = A^{-1} · y_c`, dengan `y_c` nilai kolom `c` dari K slot terpilih.

Algoritma pemilihan dan eliminasi WAJIB deterministik sehingga keluaran tidak bergantung pada urutan pembacaan read.

## 14. Pipeline Encode

Input: berkas sumber, K, M.

1. **Pass pertama:** baca berkas secara streaming, hitung `file_len` dan SHA-256 penuh. Tetapkan `file_id = SHA256[0..6]` dan `sha256_prefix = SHA256[0..16]`.
2. Validasi K, M, dan batas `MAX_SEQ`.
3. Bangun manifest, lalu keluarkan tiga oligo manifest (seq 0–2) dengan `build_oligo`.
4. **Pass kedua:** baca berkas secara streaming dalam blok per grup (K × 30 byte). Untuk setiap grup:
   a. Bagi menjadi K slot payload. Byte terakhir grup terakhir diisi nol.
   b. Bangun oligo data untuk slot nyata (`payload_len` sesuai Bagian 12.3).
   c. Hitung paritas Bagian 13.3 dan bangun M oligo paritas dengan `payload_len = 30`.
   d. Keluarkan oligo berurutan menurut seq.
5. Tulis pool dalam format FASTA (Bagian 19.1).
6. Pool yang dihasilkan WAJIB dapat didecode kembali menjadi berkas identik tanpa kanal (uji identitas, Bagian 23).

Paralelisme (jika aktif): grup diproses paralel, lalu oligo dikumpulkan dan ditulis sesuai urutan grup.

## 15. Channel Simulator

### 15.1 Parameter

| Parameter | Simbol | Rentang | Default (profil low) |
|---|---|---|---|
| Substitusi per basa | `p_s` | [0, 0.2] | 0.001 |
| Delesi per basa | `p_d` | [0, 0.2] | 0.0005 |
| Insersi per posisi | `p_i` | [0, 0.2] | 0.0005 |
| Kehilangan read | `p_l` | [0, 1) | 0.01 |
| Duplikasi read | `p_dup` | [0, 1) | 0.005 |
| Coverage | `C` | ≥ 1 | 10 |
| Seed | `seed` | u64 | wajib |

Syarat: `p_s + p_d ≤ 1`. Profil `medium` dan `high` didefinisikan dalam konfigurasi dan dipakai dalam eksperimen (Bagian 24). Nilai profil adalah parameter eksperimen, bukan klaim hasil.

### 15.2 Seed Turunan

Setiap salinan read diberi RNG tersendiri agar hasil tidak bergantung pada jumlah thread atau urutan proses:

```
s1 = SplitMix64_next( seed ^ (seq as u64) * 0x9E3779B97F4A7C15 )
s2 = SplitMix64_next( s1 ^ (copy as u64) * 0xC2B2AE3D27D4EB4F )
rng = Xoshiro256**( seed = s2 )
```
`copy` adalah indeks salinan `0..C−1`.

### 15.3 Algoritma per Salinan

Input: oligo `x` (192 nt).

1. Undi `u_loss`. Jika `u_loss < p_l`, salinan ini hilang. Tidak ada read yang dipancarkan. Lanjut ke salinan berikutnya (tetap menyelesaikan seluruh undian dari RNG untuk salinan ini agar deterministik; cukup dengan mengonsumsi satu undian).
2. Untuk setiap posisi `i` dari 0 sampai 191:
   a. Undi `u1`. Jika `u1 < p_d`, basa dibuang.
   b. Jika `p_d ≤ u1 < p_d + p_s`, emit basa pengganti dipilih secara uniform dari tiga basa lain.
   c. Jika tidak, emit `x[i]`.
   d. Undi `u2`. Jika `u2 < p_i`, emit satu basa acak uniform dari empat basa.
3. Undi `u3`. Jika `u3 < p_dup`, emit satu salinan duplikat dari read ini sebagai record tambahan dengan penanda `dup=1`.

Setiap posisi WAJIB mengonsumsi tepat tiga undian `u1`, `u2`, dan (hanya bila diperlukan) indeks basa acak. Hal ini menjamin determinisme.

### 15.4 Pengacakan Urutan Read

Setelah seluruh read dihasilkan, daftar read diacak dengan Fisher-Yates menggunakan RNG dengan seed `seed ^ 0xD1B54A32D192ED03`. Pengacakan ini WAJIB dilakukan agar decoder tidak bergantung pada urutan.

### 15.5 Keluaran

Keluaran berupa berkas FASTA reads dan berkas truth TSV dengan kolom: `read_index`, `seq`, `copy`, `dup`, `truth_nt` (oligo asal, hanya untuk benchmark). Truth tidak boleh dibaca oleh jalur decode.

## 16. Pemrosesan Read

### 16.1 Parsing Input

Format FASTA: baris yang diawali `>` adalah header dan diabaikan isinya. Baris lain disambung menjadi satu urutan per record. Karakter huruf kecil dikonversi ke huruf besar. Karakter selain `ACGT` menyebabkan record dibuang dan dihitung di laporan sebagai `invalid_alphabet`.

### 16.2 Clustering

Algoritma greedy urutan masukan:

1. Untuk setiap read `r` dalam urutan masukan:
   a. Hitung himpunan 12-mer berbeda dari `r`. Jika panjang `r` kurang dari 12, read dibuang.
   b. Untuk setiap 12-mer, ambil daftar klaster yang representatifnya memiliki k-mer tersebut dari indeks terbalik (`HashMap<u32, Vec<u32>>`).
   c. Hitung jumlah k-mer bersama per klaster kandidat. Ambil kandidat dengan jumlah ≥ `MIN_SHARED_KMERS`.
   d. Urutkan kandidat berdasarkan jumlah k-mer bersama menurun, lalu ID klaster menaik. Ambil paling banyak `MAX_CANDIDATES` teratas.
   e. Untuk setiap kandidat, hitung jarak edit berpita (Bagian 16.3) terhadap representatif. Jika jarak ≤ `floor(0.30 × panjang_representatif)`, read masuk ke klaster itu dan iterasi berhenti.
   f. Jika tidak ada kandidat yang cocok, buat klaster baru dengan `r` sebagai representatif dan indeks k-mer diperbarui.

### 16.3 Jarak Edit Berpita

Levenshtein dengan biaya substitusi, insersi, dan delesi masing-masing 1. Matriks DP hanya mengisi sel `(i, j)` dengan `|i − j| ≤ BAND_WIDTH`. Jika `|len(a) − len(b)| > BAND_WIDTH`, hasilnya dinyatakan melebihi ambang tanpa perhitungan. Perhitungan dihentikan lebih awal jika nilai minimum satu baris DP melebihi ambang.

### 16.4 Konsensus

Untuk klaster dengan read `R_1..R_n` dan representatif `T`:

Untuk setiap putaran `t` dari 1 sampai `CONSENSUS_ROUNDS`:
1. Setiap read diselaraskan terhadap template `T` dengan penyelarasan global berpita (biaya sama dengan Bagian 16.3). Traceback mengutamakan diagonal, lalu delesi (konsumsi template), lalu insersi, saat terjadi seri.
2. Setiap posisi template `j` menerima suara dari read yang terselaraskan ke `j`: basa `A, C, G, T` atau gap `-`. Basa hasil insersi relatif terhadap template DIABAIKAN.
3. Konsensus posisi `j` adalah simbol dengan suara terbanyak. Seri diselesaikan dengan urutan prioritas `A < C < G < T < -`, dipilih yang indeks lebih kecil.
4. Posisi dengan konsensus `-` dihapus. Hasilnya adalah template baru `T := konsensus_t`.

Hasil akhir adalah `T` setelah putaran terakhir. Jika panjangnya bukan 192, klaster dinyatakan **gagal konsensus** dan dicatat di laporan. Klaster dengan satu read menghasilkan konsensus berupa read itu sendiri, lalu tetap menjalani validasi panjang.

## 17. Pipeline Decode

Input: berkas read, opsional `file_id` yang dipilih, opsi `allow_partial`.

1. Parse read (Bagian 16.1).
2. Clustering (Bagian 16.2).
3. Konsensus (Bagian 16.4) untuk setiap klaster.
4. Parse oligo (Bagian 11) untuk setiap konsensus berpanjang 192. Oligo yang gagal dicatat dengan alasan.
5. Kelompokkan oligo valid menurut `file_id`. Jika terdapat lebih dari satu `file_id` dan `--file-id` tidak diberikan, decode WAJIB gagal dengan `AmbiguousFileId`.
6. Dari oligo bertipe manifest yang valid, ambil nilai mayoritas (berdasarkan isi payload 30 byte). Jika terdapat dua isi berbeda dengan jumlah sama, decode gagal dengan `ManifestConflict`. Jika tidak ada manifest valid, decode gagal dengan `ManifestUnavailable`.
7. Validasi manifest: `K` dan `M` dalam rentang Bagian 12.5, `scheme_id == 1`, dan reserved nol.
8. Hitung `N` dan `G` dari `file_len`.
9. Untuk setiap grup `g` (berurutan):
   a. Kumpulkan slot yang diketahui: slot data dan paritas yang memiliki oligo valid dengan `kind`, `seq`, dan `payload_len` yang sesuai. Jika terdapat lebih dari satu oligo untuk satu seq dengan isi berbeda, oligo pertama menurut ID klaster dipakai dan konflik dicatat.
   b. Tambahkan slot virtual dengan nilai nol.
   c. Jalankan decoding grup (Bagian 13.4). Jika tidak dapat dipulihkan, catat rentang seq yang hilang.
   d. Tulis byte data hasil pemulihan ke keluaran sesuai panjang yang diharapkan.
10. Hitung SHA-256 secara bertahap dari byte yang ditulis dan bandingkan 16 byte pertama dengan `sha256_prefix`.

Jika ada grup tidak dapat dipulihkan dan `allow_partial = false`, keluaran sementara dihapus dan decode gagal dengan `UnrecoverableData` yang memuat daftar grup. Jika `allow_partial = true`, grup yang gagal diisi nol, daftar rentang byte rusak dicatat, dan status verifikasi dinyatakan gagal.

## 18. Verifikasi Integritas

- Hash dibandingkan dalam waktu konstan (`subtle`-style, atau XOR akumulatif), meski tidak ada kebutuhan keamanan.
- Keluaran ditulis ke `<output>.part`. Berkas hanya dipindahkan ke `<output>` setelah verifikasi berhasil.
- Jika verifikasi gagal dan tanpa `--keep-unverified`, berkas `.part` dihapus dan decode gagal dengan exit code 5.

## 19. Format Berkas Eksternal

### 19.1 Pool FASTA

Header setiap record:
```
>df|fid=<12 hex huruf kecil>|seq=<desimal>|kind=<data|parity|manifest>
<192 nukleotida dalam satu baris>
```

Urutan record WAJIB menaik menurut `seq`.

### 19.2 Reads FASTA

Format FASTA standar. Satu read dapat dipisah menjadi beberapa baris, tetapi decoder WAJIB menyambungkannya.

### 19.3 Truth TSV

Kolom tab-separated: `read_index`, `seq`, `copy`, `dup`, `truth_nt`. Baris pertama adalah header. File ini hanya dibuat oleh `simulate` dan dibaca oleh benchmark.

### 19.4 Laporan JSON

Setiap eksekusi benchmark atau decode menghasilkan objek JSON dengan field minimal: `tool_version`, `format_version`, `git_commit`, `rustc_version`, `host` (CPU, jumlah core, RAM), `params` (K, M, seed, profil kanal), `input` (ukuran byte, SHA-256 lengkap), `counts` (oligo dipancarkan, read masuk, read invalid, klaster, klaster gagal konsensus, oligo valid, grup gagal), `timing` (ms per tahap), `memory` (peak RSS byte), `verified` (boolean), dan `error` (null atau kode).

## 20. Command-Line Interface

Binary: `dnaforge`.

```
dnaforge encode   --input <file> --output <pool.fasta> [--k <K>] [--m <M>] [--threads <n>]
dnaforge simulate --input <pool.fasta> --output <reads.fasta> --truth <truth.tsv>
                  --seed <u64> --sub <f64> --del <f64> --ins <f64>
                  --loss <f64> --dup <f64> --coverage <u32> [--no-shuffle]
dnaforge decode   --input <reads.fasta> --output <file> [--file-id <hex12>]
                  [--allow-partial] [--keep-unverified] [--threads <n>]
dnaforge inspect  --input <pool.fasta>
dnaforge bench    --suite <micro|robustness|pipeline> --seeds <n>
                  --output <results.jsonl> [--size <bytes>]
```

Perintah `inspect` menampilkan ringkasan: jumlah oligo per kind, file_id, nilai K dan M dari manifest, serta jumlah oligo yang gagal parsing. Output dalam format teks yang stabil.

Aturan umum: `--threads 0` berarti menggunakan jumlah core logis. Setiap argumen numerik WAJIB divalidasi dengan batas yang disebutkan pada Bagian 6 dan 15.

## 21. Penanganan Error dan Exit Code

| Exit code | Arti |
|---|---|
| 0 | Berhasil. Untuk decode, berkas terverifikasi. |
| 1 | Error internal (bug). |
| 2 | Argumen CLI tidak valid. |
| 3 | Masukan tidak valid (format FASTA, alfabet, ukuran). |
| 4 | Data tidak dapat dipulihkan (grup tidak cukup, manifest hilang). |
| 5 | Verifikasi hash gagal. |
| 6 | Encoding gagal karena constraint tidak terpenuhi. |

Setiap error ditampilkan dalam satu baris yang memuat kode error dan konteks (misalnya seq atau indeks grup). Stack trace hanya ditampilkan pada mode debug.

## 22. Determinisme, Konkurensi, dan Memori

### 22.1 Determinisme

- Untuk masukan dan parameter yang sama, `encode` WAJIB menghasilkan pool byte-identik, terlepas dari jumlah thread.
- `simulate` dengan seed yang sama WAJIB menghasilkan read dan truth yang identik.
- `decode` WAJIB menghasilkan keluaran identik untuk urutan read yang diacak.
- Struktur data yang urutannya memengaruhi hasil WAJIB `BTreeMap`, `Vec`, atau urutan eksplisit. `HashMap` hanya boleh untuk pencarian yang hasilnya diurutkan sebelum dipakai.

### 22.2 Konkurensi

Paralelisme hanya diizinkan pada: encoding grup, konsensus klaster, dan decoding RS grup. Clustering bersifat sekuensial karena bergantung pada urutan. Pengumpulan hasil paralel WAJIB mempertahankan urutan indeks asal.

### 22.3 Memori

- Encode: memori puncak independen dari ukuran berkas, dengan O(K × 30 byte) untuk buffer grup, ditambah buffer I/O sebesar 1 MiB.
- Decode: memori sebanding dengan jumlah oligo valid yang disimpan (sekitar 48 byte per oligo ditambah overhead struktur), ditambah indeks klaster yang sebanding dengan jumlah read.
- Buffer I/O WAJIB `BufReader` dan `BufWriter` dengan kapasitas 1 MiB.

## 23. Pengujian

### 23.1 Pengujian Unit

| Modul | Pengujian wajib |
|---|---|
| core | CRC-16 nilai uji `"123456789"` = `0x29B1`. FNV-1a 64 untuk input kosong = `0xcbf29ce484222325`. SplitMix64 dan Xoshiro256** deterministik untuk seed sama. SHA-256 memenuhi vektor NIST `"abc"`. |
| codec | Pack/unpack bijektif untuk semua byte 0–255 dan panjang acak. Penolakan alfabet tidak valid. Constraint engine untuk setiap kasus batas (GC 76 dan 77, 115 dan 116, run 3 dan 4). Whitening involutif. |
| codec | Round-trip `build_oligo` lalu `parse_oligo` untuk field acak (property test, minimal 10.000 kasus). |
| codec | Parsing menolak: magic salah, versi salah, kind tidak dikenal, payload_len di luar rentang, CRC salah, reserved manifest tidak nol. |
| ecc | Tabel GF: `a · inv(a) = 1` untuk semua `a ≠ 0`. Distributivitas dan asosiativitas perkalian (uji acak). Setiap submatriks persegi Cauchy untuk K ≤ 4 dan M ≤ 3 diuji invertibel secara eksaustif. |
| ecc | Encode lalu decode untuk K = 4, M = 2 dengan setiap pola erasure hingga M, dan K = 10, M = 4 dengan 10.000 pola acak. |
| fragment | Pemetaan seq sesuai rumus Bagian 12.4. Slot virtual dan panjang payload akhir benar untuk N = 0, 1, K−1, K, K+1, dan kelipatan 30 ± 1. |
| channel | Determinisme. Frekuensi empiris tiap kejadian dalam 10^6 undian berada dalam 5 simpangan baku dari probabilitas teoritis. Dengan semua parameter nol, read identik dengan oligo. |
| recovery | Jarak edit berpita sama dengan Levenshtein penuh untuk string dengan selisih panjang ≤ 24 (uji acak 10.000 pasangan). Konsensus memulihkan oligo sempurna untuk coverage ≥ 5 dengan substitusi 1%. |

### 23.2 Pengujian Integrasi

Untuk setiap ukuran berkas berikut, dilakukan encode, lalu decode tanpa kanal, dan hasilnya WAJIB identik byte demi byte: 0, 1, 29, 30, 31, 299, 300, 301, 1024 byte, 1 MiB acak, 1 MiB nol, 1 MiB pola berulang.

Untuk setiap skenario kanal berikut, decode WAJIB memulihkan file identik jika jumlah erasure dalam setiap grup ≤ M:
- Kehilangan oligo tepat M per grup pada posisi acak.
- Kehilangan oligo tepat M per grup pada posisi paritas.
- Kehilangan oligo lebih dari M pada satu grup, dengan `--allow-partial`. Harus menghasilkan exit code 4 tanpa `--allow-partial` dan exit code 5 dengan `--allow-partial`.

### 23.3 Pengujian Golden

Satu berkas uji tetap (64 KiB, isi dibangkitkan dari seed 42) di-encode. SHA-256 pool keluaran WAJIB sama dengan nilai yang tercatat di repositori. Perubahan nilai ini hanya sah jika bersamaan dengan perubahan `format_version` atau revisi spesifikasi.

### 23.4 Fuzzing

Target fuzz untuk `parse_oligo`, `unpack`, parser FASTA, dan decode lengkap pada masukan acak. Syarat: tidak ada panic, tidak ada overflow aritmetika yang tidak tertangani, dan tidak ada alokasi di luar batas wajar. Target dijalankan minimal 10 menit di CI nightly.

### 23.5 Pengujian Memori

Encode berkas 1 MiB dan 16 MiB dengan `K = 10`. Selisih peak RSS (VmHWM di Linux) WAJIB kurang dari 8 MiB.

### 23.6 Pengujian Determinisme Paralel

Encode dan decode yang sama dijalankan dengan `--threads 1` dan `--threads 8`. Keluaran WAJIB identik byte demi byte.

## 24. Benchmark dan Eksperimen Ketahanan

### 24.1 Benchmark Mikro (Criterion.rs)

| Nama | Yang diukur | Masukan |
|---|---|---|
| `gf_mul` | Perkalian GF(256) | 10^6 pasang acak |
| `gf_inv` | Invers GF(256) | 10^6 nilai acak ≠ 0 |
| `cauchy_encode_group` | Encode paritas satu grup | K = 10, M = 4 |
| `rs_decode_group` | Decode grup dengan 4 erasure | K = 10, M = 4 |
| `crc16_46b` | CRC-16 atas 46 byte | Acak |
| `keystream_32b` | Pembangkitan keystream | Acak |
| `pack_unpack` | Pack dan unpack 48 byte | Acak |
| `constraint_check` | Pemeriksaan constraint | Oligo acak |
| `oligo_build` | Build dengan variant search | Payload acak |
| `oligo_parse` | Parse oligo valid | Oligo acak |
| `edit_banded` | Jarak edit berpita | Pasangan read 192 nt, error 5% |
| `consensus_50` | Konsensus 50 read | Coverage 50 |
| `encode_1mib` / `encode_16mib` | Encode lengkap | Acak |
| `decode_1mib_medium` | Decode lengkap | Profil medium, C = 10 |
| `simulate_channel` | Simulasi kanal | 10.000 oligo |

Setiap benchmark WAJIB menggunakan `black_box` untuk mencegah optimasi berlebihan. Pengukuran WAJIB mencakup minimal 100 sampel per benchmark sesuai default Criterion.

### 24.2 Eksperimen Ketahanan

Protokol:
- Berkas uji: 64 KiB, dibangkitkan secara deterministik dari seed `0..S−1`.
- Seed eksperimen: `0..19` (20 seed). Setiap seed menghasilkan berkas uji dan pola kanal yang berbeda.
- Grid parameter:
  - Substitusi `p_s` ∈ {0, 0.001, 0.005, 0.01, 0.02}.
  - Delesi dan insersi `p_d = p_i` ∈ {0, 0.0005, 0.002, 0.005}.
  - Kehilangan `p_l` ∈ {0, 0.05, 0.1, 0.2}.
  - Coverage `C` ∈ {1, 3, 5, 10, 20}.
  - Duplikasi `p_dup` tetap 0.01.
- Setiap kombinasi dijalankan untuk seluruh seed.
- Untuk setiap kombinasi dicatat seluruh metrik Bagian 25.

Catatan: coverage 1 dengan indel menghasilkan konsensus yang tidak sama panjang dengan 192, sehingga kegagalan tercatat secara sah dan tidak boleh disembunyikan dari laporan.

### 24.3 Suite Pipeline

Satu eksekusi penuh encode, simulate, dan decode pada profil `low`, `medium`, dan `high`, dengan 10 pengulangan per profil, untuk mengukur waktu end-to-end dan ukuran pool.

### 24.4 Regresi

Hasil benchmark disimpan sebagai baseline per versi. Penurunan throughput lebih dari 10% pada benchmark dengan median yang sama, atau kenaikan peak memori lebih dari 10%, WAJIB ditandai sebagai regresi dan memblokir merge.

## 25. Metrik

| Metrik | Rumus | Satuan |
|---|---|---|
| Kepadatan bersih | `D_net = 8·S / N_nt` | bit/nt |
| Overhead payload | `O = (30 · N_emitted_payload) / S − 1` | rasio |
| Rasio data efektif | `R = S / (N_nt / 4)` | byte/byte |
| Tingkat oligo valid | `R_oligo = valid / dipancarkan` | proporsi |
| Coverage efektif | `C_eff = read_valid / oligo_dipancarkan` | kali |
| Tingkat pemulihan grup | `R_group = grup_pulih / G` | proporsi |
| Keberhasilan file | `P_success = trial_identik / trial` | proporsi |
| Akurasi konsensus | `A_cons = klaster_konsensus_benar / klaster` | proporsi |
| Purity klaster | `Purity = Σ_k max_s |k ∩ s| / read_valid` | proporsi |
| Throughput encode | `T_enc = S / t_enc` | MiB/s |
| Throughput decode | `T_dec = S / t_dec` | MiB/s |
| Peak memori | VmHWM atau setara | byte |
| Jarak edit ternormalisasi | `d_norm = lev(konsensus, truth) / 192` | proporsi |

Di mana `S` adalah ukuran berkas sumber dalam byte, `N_nt` adalah total nukleotida pool, dan `N_emitted_payload` adalah jumlah oligo (data nyata, paritas, dan manifest) dikali 30.

Untuk proporsi, WAJIB dilaporkan interval kepercayaan 95% Wilson. Untuk waktu, WAJIB dilaporkan median dan rentang interkuartil dari minimal 10 pengulangan.

## 26. Pelaporan Hasil

Setiap laporan eksperimen WAJIB memuat:
1. Lingkungan: model CPU, jumlah core, RAM, OS, versi rustc, dan commit git.
2. Parameter lengkap: K, M, profil kanal, grid, seed, dan coverage.
3. Tabel metrik Bagian 25 per kombinasi parameter, dengan interval kepercayaan.
4. Daftar kegagalan dengan kategori: oligo tidak valid, klaster gagal konsensus, grup tidak dapat dipulihkan, verifikasi gagal.
5. Pernyataan batasan: hasil berlaku untuk model kanal yang disimulasikan dan tidak membuktikan keberhasilan sintesis, penyimpanan, atau sekuensing fisik.

## 27. CI dan Quality Gates

Pipeline CI WAJIB menjalankan, untuk setiap perubahan:

1. `cargo fmt --all -- --check`
2. `cargo clippy --workspace --all-targets -- -D warnings`
3. `cargo test --workspace` pada Linux, macOS, dan Windows.
4. Pengujian pada MSRV 1.75.
5. Pengujian golden (Bagian 23.3).
6. Pengujian determinisme paralel (Bagian 23.6).
7. `cargo doc --workspace --no-deps` tanpa peringatan.
8. `cargo llvm-cov` dengan ambang minimum garis: 90% untuk core, codec, ecc, dan fragment; 80% untuk recovery dan engine.
9. Benchmark smoke (subset kecil) untuk memastikan benchmark tetap berjalan.
10. Pemeriksaan lisensi dan kerentanan dependensi (`cargo deny`, `cargo audit`).

Fuzzing dan benchmark penuh dijalankan terjadwal, bukan pada setiap perubahan.

## 28. Kriteria Kesesuaian

Implementasi dinyatakan sesuai spesifikasi v1 jika dan hanya jika seluruh kondisi berikut terpenuhi:

1. Setiap konstanta pada Bagian 6 memiliki nilai yang sama dalam kode.
2. Setiap oligo yang dihasilkan encoder berukuran tepat 192 nukleotida dan lolos constraint engine.
3. Pengujian integrasi Bagian 23.2 lulus seluruhnya.
4. Pengujian golden Bagian 23.3 menghasilkan SHA-256 yang tercatat.
5. Decode dapat memulihkan berkas identik pada setiap skenario erasure dengan jumlah erasure per grup ≤ M.
6. Keluaran encode dan decode deterministik terhadap jumlah thread.
7. Seluruh quality gate Bagian 27 lulus tanpa pengecualian.
8. Setiap metrik pada laporan dihitung sesuai definisi Bagian 25.
9. Tidak ada fitur di luar spesifikasi yang aktif secara default pada jalur format.
10. Seluruh batasan dan klaim yang dilaporkan menyatakan secara eksplisit bahwa hasil berasal dari simulasi komputasi.
