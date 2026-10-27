# Product Requirements Document (PRD) & Blueprint Arsitektur MSBD
## Sistem Informasi Manajemen Sekolah 1 Atap Terdistribusi (Edisi Revisi v2.0 - PostgreSQL)

**Status Dokumen:** Architecture Decision Locked (Siap Tahap Pemodelan & DDL)  
**RDBMS Utama:** **PostgreSQL 16+** (Engine Terkunci)  
**Cakupan Entitas:** Satu Yayasan menaungi **5 Jenjang (SD, SMP, SMA, MTs, SMK)** dengan *Codebase* Seragam & Isolasi Data per Jenjang  
**Skala Target:** $\ge 1400$ Siswa Aktif Lintas Jenjang, $\pm 120$ Guru & Staf Administrasi  
**Paradigma Arsitektur:** Modular Monolith (API-First) + Multi-Jenjang Isolation + Distributed E-Learning + Cloudflare R2 Stream  

---

## 1. Ringkasan Eksekutif & Topologi Infrastruktur Terbaru

Sistem ini dirancang untuk mengelola operasional **Yayasan Pendidikan 1 Atap** yang menaungi **5 jenjang pendidikan berbeda (SD, SMP, SMA, MTs, dan SMK)**. Setiap jenjang memiliki karakteristik operasional dan penyesuaian kurikulum tersendiri (misalnya struktur mata pelajaran kejuruan/PKL pada SMK dan muatan keagamaan Kemenag pada MTs), sehingga sistem di-deploy dalam **versi masing-masing jenjang** menggunakan satu arsitektur kode inti (*single unified core architecture*).

### Topologi Infrastruktur Terdistribusi (Konfigurasi Baru)

1. **Server 2 — Core Backend, PostgreSQL & Redis Node ($6\text{ vCPU} / 6\text{ GB RAM}$):**
   * Merupakan jantung komputasi sistem yang menjalankan **Backend Laravel (Modular Monolith, API-First)**.
   * Menampung **PostgreSQL Primary Cluster** sebagai basis data relasional utama yang memanfaatkan fitur *Schema Isolation* (`schema_sd`, `schema_smp`, `schema_sma`, `schema_mts`, `schema_smk`, serta `schema_yayasan_shared`), tipe data `JSONB`, *Declarative Partitioning*, dan fungsi prosedural `PL/pgSQL`.
   * Menjalankan **Redis** sebagai *in-memory buffer* untuk menangani konkurensi tinggi pada CBT Custom (menyimpan jawaban setiap kali siswa berpindah soal) dan manajemen *Queue Worker*.
   * Menangani aliran unggah/unduh berkas melalui **Laravel Backend Stream** langsung ke Cloudflare R2.

2. **Server 1 — E-Learning (Moodle) & Edge Services Node ($4\text{ vCPU} / 4\text{ GB RAM}$):**
   * Menjalankan **Moodle LMS** untuk kegiatan pembelajaran asinkron, distribusi materi statis, dan forum diskusi.
   * Kapasitas $4\text{C}/4\text{G}$ memberikan ruang memori yang jauh lebih lega untuk *PHP-FPM workers* Moodle serta layanan pendukung seperti *Reverse Proxy* atau *worker* perenderan PDF statis.
   * Terhubung ke PostgreSQL di Server 2 melalui koneksi terbatas (*Read-Only Foreign Data Wrapper* / *Database View*) untuk sinkronisasi autentikasi pengguna dan struktur kelas.

3. **Distributed Object Storage (Cloudflare R2 via Laravel Backend Stream):**
   * Seluruh berkas berukuran besar (dokumen pendaftaran PPDB, hasil pindai arsip fisik, materi Moodle, serta **cetakan PDF permanen Rapor & Surat Digital ber-Barcode TTD**) dialirkan melalui *stream* (`Flysystem S3 Stream`) dari Backend Laravel di Server 2 menuju **Cloudflare R2**.
   * PostgreSQL hanya menyimpan *metadata* objek, *checksum* $\text{SHA-256}$, dan *status lifecycle* berkas (`PENDING`, `ATTACHED`, `ORPHANED`).

---

## 2. Matriks Pembagian Tugas Tim & Struktur Laporan (WBS)

| Anggota Tim | Peran Spesialis | Tanggung Jawab Bab Laporan MSBD | Fokus Teknis & Deliverables Utama (Spesifik PostgreSQL) |
| :--- | :--- | :--- | :--- |
| **Yosua** | *Backend Architect & Core DB Logic* | **BAB III.4, BAB III.5, BAB III.6** (Serta supervisi arsitektur API-First) | Desain ERD Multi-Jenjang (SD/SMP/SMA/MTs/SMK), Normalisasi Rapor Kurikulum Merdeka (dengan kolom deskripsi *custom*), Skema *High-Concurrency* CBT (persistensi per pindah soal), *Trigger* `PL/pgSQL` Kartu Ujian Sementara Berbatas Waktu, *Stored Procedure* Penilaian, & *Partition Archiving*. |
| **[Nama Anggota 2]** | *Systems Analyst & Modeler* | **BAB I, BAB II, BAB III.1, BAB III.2, BAB III.3** | Perencanaan proyek, studi kelayakan infrastruktur baru ($4\text{C}/4\text{G}$ & $6\text{C}/6\text{G}$), analisis sistem berjalan di 5 jenjang yayasan, elisitasi *User Requirement* 15 fitur, serta pemodelan DFD Level Konteks s.d. Level 2. |
| **[Nama Anggota 3]** | *Infrastructure & Distributed DB Specialist* | **BAB III.7 (Bagian Transaksi), BAB IV.2** (Serta topologi terdistribusi) | Perancangan *ACID Transaction* & *Pessimistic Locking* (`SELECT ... FOR UPDATE`) Kasir Offline di PostgreSQL, topologi jaringan Server 1 ($4\text{C}/4\text{G}$) $\leftrightarrow$ Server 2 ($6\text{C}/6\text{G}$), pipa *Laravel Backend Stream* ke Cloudflare R2, & *Garbage Collector* berkas `ORPHANED`. |
| **[Nama Anggota 4]** | *Security & API Architect* | **BAB III.7 (Bagian Keamanan & Hak Akses)** | Desain *Multi-Role & Multi-Jenjang RBAC*, matriks hak akses tingkat skema PostgreSQL (DCL: `GRANT`/`REVOKE`), verifikasi kriptografis PDF Barcode TTD, serta *Stateless Authentication* (JWT) untuk kesiapan aplikasi Mobile. |
| **[Nama Anggota 5]** | *UI/UX & Implementation Lead* | **BAB III.8, BAB IV.1, BAB IV.3, BAB V** | Desain *wireframe* (termasuk pembedaan visual **Kartu Ujian Reguler vs. Kartu Ujian Sementara**), kompilasi skrip DDL/DML PostgreSQL, pengujian *stress test* CBT per perpindahan soal, & analisis kinerja kueri (`EXPLAIN ANALYZE`). |

---

## 3. Arsitektur Multi-Jenjang "Satu Yayasan Lima Unit" di PostgreSQL

Karena definisi **Sekolah 1 Atap** pada proyek ini adalah satu Yayasan yang membawahi **SD, SMP, SMA, MTs, dan SMK** di mana setiap jenjang memiliki versi sistem sendiri namun dengan arsitektur inti yang sama, PostgreSQL memberikan solusi elegan melalui **Schema-Based Multi-Tenancy** yang dikombinasikan dengan tabel atribut spesifik jenjang:

1. **Skema Bersama Yayasan (`schema_yayasan`):**
   * Menyimpan entitas lintas unit seperti `master_yayasan`, `unit_jenjang` (`SD`, `SMP`, `SMA`, `MTs`, `SMK`), `pegawai_lintas_jenjang` (jika ada guru/staf yang mengajar di SMP sekaligus SMA), `rekening_kas_yayasan`, dan `ppdb_pendaftar` (memudahkan pelacakan alumni SD/SMP/MTs yang mendaftar lanjut ke SMA/SMK di yayasan yang sama).
2. **Skema Operasional per Jenjang (`schema_sd`, `schema_smp`, `schema_sma`, `schema_mts`, `schema_smk`):**
   * Memiliki struktur tabel 15 fitur yang identik (seragam secara DDL), namun terisolasi secara fisik di dalam klaster PostgreSQL Server 2.
   * Saat pengguna mengakses aplikasi (misal versi SMA di `sma.sekolah.sch.id` atau memilih *scope* SMA di aplikasi Mobile), *middleware* Laravel secara otomatis mengatur `SET search_path TO schema_sma, schema_yayasan;` pada koneksi PostgreSQL.
3. **Fleksibilitas Spesifik Jenjang dengan `JSONB`:**
   * Perbedaan kecil antarjenjang (misalnya data Praktik Kerja Lapangan/PKL dan Kopetensi Keahlian pada **SMK**, atau atribut nomor statistik madrasah/EMIS pada **MTs**) disimpan pada kolom ekstensi bertipe `atribut_spesifik_jenjang JSONB` yang diindeks dengan **GIN Index** di PostgreSQL, sehingga tidak merusak keseragaman tabel utama.

---

## 4. Rancangan Detail Berdasarkan Sistematika Laporan MSBD

### BAB I & BAB II: Perencanaan dan Studi Kelayakan Sistem
**Penanggung Jawab Utama:** `[Nama Anggota 2]`

* **Analisis Beban Kerja Server 2 ($6\text{C}/6\text{G}$) & Server 1 ($4\text{C}/4\text{G}$):**
  * **Beban Transaksi CBT (Perpindahan Soal):** Karena sistem menyimpan jawaban setiap kali siswa menyelesaikan 1 soal/berpindah soal, maka jika terdapat $1000$ siswa ujian serentak yang berpindah soal rata-rata setiap $90\text{ detik}$, sistem akan menerima sekitar:
    $$\lambda = \frac{1000\text{ permintaan}}{90\text{ detik}} \approx 11,11\text{ operasi tulis/detik (rata-rata)}$$
    Namun, pada detik-detik awal ujian atau menjelang waktu habis, lonjakan (*burst traffic*) dapat mencapai $\ge 250\text{ permintaan/detik}$. Penempatan Redis dan PostgreSQL di Server 2 ($6\text{C}/6\text{G}$) menjamin proses tangkap-tulis ini berjalan tanpa jeda jaringan antar-VPS.
  * **Beban *Laravel Backend Stream* ke R2:** Dengan menggunakan *chunked streaming* (misal ukuran *buffer* $2\text{ MB}$ per aliran unggah), 50 pengguna yang mengunggah berkas PPDB atau arsip secara bersamaan hanya akan mengonsumsi memori sekitar $50 \times 2\text{ MB} = 100\text{ MB}$ RAM di Server 2, sangat aman bagi kapasitas $6\text{ GB}$ RAM.

---

### BAB III.1 – III.3: Analisis Sistem, Kebutuhan Pengguna, dan DFD
**Penanggung Jawab Utama:** `[Nama Anggota 2]`

* **Penyesuaian DFD Level 1 & Level 2 terhadap Keputusan Baru:**
  * **Aliran Data Unggah Berkas & R2 Garbage Collection:** DFD harus menggambarkan bahwa setiap berkas dari klien melewati proses *Backend Stream Validator* di Server 2 sebelum diteruskan ke *External Storage* Cloudflare R2.
  * **Aliran Data Cetak Kartu Ujian (Reguler vs. Sementara):** DFD Level 2 pada modul Keuangan & Akademik wajib memisahkan *output* antara **Kartu Ujian Reguler** (bagi siswa lunas SPP) dan **Kartu Ujian Sementara Berbatas Waktu** (bagi siswa yang mendapat *override* kasir).
  * **Aliran Data Pengesahan Dokumen (Barcode TTD):** Menggambarkan proses pembangkitan PDF statis saat pejabat sekolah menandatangani Rapor atau Surat Digital, diikuti aliran penyimpanan PDF permanen ke Cloudflare R2.

---

### BAB III.4 & III.5: Perancangan Basis Data di PostgreSQL (ERD & Normalisasi)
**Penanggung Jawab Utama:** `Yosua`

#### A. Normalisasi Rapor Kurikulum Merdeka & Dukungan Kolom Deskripsi Custom
Mengakomodasi keputusan bahwa guru/wali kelas diberikan hak menyunting deskripsi capaian secara manual apabila deskripsi otomatis dirasa kurang pas, struktur tabel hasil normalisasi ($3\text{NF}/\text{BCNF}$) pada tabel `rapor_intrakurikuler` dirancang sebagai berikut:

* ** Pemisahan Deskripsi Otomatis dan Deskripsi Kustom:**
  * `nilai_akhir_angka NUMERIC(5,2)`: Hasil kalkulasi bobot sumatif lingkup materi dan sumatif akhir semester.
  * `id_tp_tertinggi UUID` & `id_tp_terendah UUID`: Referensi ke butir Tujuan Pembelajaran dengan nilai formatif tertinggi dan terendah.
  * `deskripsi_capaian_otomatis TEXT`: Kalimat hasil rakitan *Stored Procedure* PostgreSQL berdasarkan capaian TP.
  * `deskripsi_capaian_custom TEXT NULL`: **Kolom khusus sesuai keputusan arsitektur**. Jika guru/wali kelas mengisi kolom ini, maka nilai pada kolom ini yang akan tampil saat perenderan PDF Rapor (menggunakan fungsi `COALESCE(deskripsi_capaian_custom, deskripsi_capaian_otomatis)`).
  * `is_locked BOOLEAN DEFAULT FALSE`: Dikunci menjadi `TRUE` saat rapor disahkan dengan Barcode TTD dan dirender menjadi PDF statis ke R2.

* **Strategi Adaptif untuk Pertanyaan 1 (TP) & Pertanyaan 3 (P5) — Menunggu File Excel Sekolah:**
  Agar Yosua dapat mulai membangun kerangka ERD sambil menunggu file aplikasi rapor lama dari pihak sekolah, skema tabel `tujuan_pembelajaran` dan `projek_p5` menggunakan **Pola Relasi Opsional (Nullable Foreign Key Scope)**:
  * Pada tabel `tujuan_pembelajaran`, sertakan kolom `id_mapel UUID NOT NULL`, `tingkat_kelas VARCHAR(10) NOT NULL`, serta kolom opsional `id_rombel UUID NULL` dan `id_guru_pembuat UUID NULL`. Jika nantinya file Excel sekolah menunjukkan TP bersifat terpusat per tingkat, maka `id_rombel` cukup dikosongkan (`NULL`). Jika ternyata per guru/kelas berbeda, kolom `id_rombel` langsung dapat digunakan tanpa perlu mengubah ulang struktur tabel (tanpa *breaking migration*).

#### B. Skema Manajemen Berkas & Siklus Hidup Objek Cloudflare R2 (`storage_objects`)
Mendukung keputusan **Laravel Backend Stream**, **Garbage Collection**, dan **PDF Statis Barcode TTD**:
* Tabel `storage_objects`:
  * `id_object UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  * `jenjang_unit VARCHAR(10)` (`SD`, `SMP`, `SMA`, `MTS`, `SMK`, `YAYASAN`)
  * `r2_bucket VARCHAR(100) NOT NULL`
  * `r2_object_key VARCHAR(512) UNIQUE NOT NULL`
  * `original_filename VARCHAR(255) NOT NULL`
  * `mime_type VARCHAR(100) NOT NULL`
  * `size_bytes BIGINT NOT NULL`
  * `sha256_hash CHAR(64) NOT NULL` (Digunakan untuk memverifikasi keaslian PDF Rapor/Surat saat Barcode TTD dipindai)
  * `status_berkas VARCHAR(20) CHECK (status_berkas IN ('PENDING', 'ATTACHED', 'ORPHANED')) DEFAULT 'PENDING'`
  * `is_immutable_signed_pdf BOOLEAN DEFAULT FALSE` (Bernilai `TRUE` untuk PDF Rapor & Surat Digital yang telah disahkan Barcode TTD)
  * `created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP`

#### C. Skema Kartu Ujian Reguler vs. Kartu Ujian Sementara (`dispensasi_kartu_ujian`)
Mendukung keputusan bahwa izin cetak (*override*) dari kasir hanya berlaku untuk **1 periode ujian spesifik**, memiliki **batas waktu beberapa hari**, menghasilkan **Kartu Ujian Sementara dengan desain berbeda**, dan **wajib mengajukan ulang** jika batas waktu terlewati:
* Tabel `periode_ujian`:
  * `id_periode_ujian UUID PRIMARY KEY`, `id_semester UUID`, `jenis_ujian VARCHAR(20)` (`UTS`, `UAS`, `US`), `tgl_mulai DATE`, `tgl_selesai DATE`.
* Tabel `hak_akses_kartu_ujian`:
  * `id_akses UUID PRIMARY KEY`
  * `id_siswa UUID NOT NULL`
  * `id_periode_ujian UUID NOT NULL`
  * `status_akses VARCHAR(25) CHECK (status_akses IN ('BLOCKED', 'REGULAR_UNLOCKED', 'TEMPORARY_OVERRIDE', 'EXPIRED_OVERRIDE')) DEFAULT 'BLOCKED'`
  * `jenis_desain_kartu VARCHAR(20) CHECK (jenis_desain_kartu IN ('NONE', 'KARTU_REGULER', 'KARTU_SEMENTARA')) DEFAULT 'NONE'`
  * `batas_waktu_sementara TIMESTAMPTZ NULL` (Diisi ketika status adalah `TEMPORARY_OVERRIDE`, misal `CURRENT_TIMESTAMP + INTERVAL '3 days'`)
  * `kali_pengajuan_override INT DEFAULT 0` (Mencatat berapa kali siswa mengajukan perpanjangan/pengajuan ulang pada periode ujian tersebut)
  * *Unique Constraint:* `UNIQUE (id_siswa, id_periode_ujian)`
* Tabel `riwayat_override_kasir`:
  * Mencatat setiap kali pengajuan dispensasi disetujui kasir (`id_override`, `id_akses`, `id_shift_kasir`, `id_petugas_kasir`, `alasan_dispensasi`, `nominal_tunggakan_saat_itu`, `berlaku_mulai`, `berlaku_sampai`).

#### D. Skema CBT Custom dengan Persistensi Per Pindah Soal (`cbt_jawaban_peserta`)
* Tabel `cbt_sesi_peserta`: Menyimpan status sesi ujian siswa, waktu mulai, sisa waktu, dan *flag* `status_akses_kartu` (memastikan siswa yang Kartu Ujian Sementaranya sudah `EXPIRED_OVERRIDE` tidak bisa masuk ke ruang CBT sampai memperbarui izin di kasir!).
* Tabel `cbt_jawaban_peserta`:
  * `id_sesi_peserta UUID NOT NULL`
  * `id_butir_soal UUID NOT NULL`
  * `opsi_terpilih VARCHAR(10) NULL` / `jawaban_esai TEXT NULL`
  * `is_ragu_ragu BOOLEAN DEFAULT FALSE`
  * `waktu_jawab_client TIMESTAMPTZ NOT NULL`
  * `waktu_sinkron_db TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP`
  * `urutan_revisi INT DEFAULT 1` (Bertambah setiap kali siswa kembali ke soal tersebut dan mengubah jawabannya)
  * `PRIMARY KEY (id_sesi_peserta, id_butir_soal)` — Memungkinkan operasi PostgreSQL `INSERT INTO cbt_jawaban_peserta ... ON CONFLICT (id_sesi_peserta, id_butir_soal) DO UPDATE SET ... WHERE EXCLUDED.waktu_jawab_client >= cbt_jawaban_peserta.waktu_jawab_client`. Klausa `WHERE` tersebut sangat krusial agar jika ada 2 *job* antrean Redis yang masuk tidak berurutan saat siswa berpindah soal dengan cepat, jawaban yang lebih baru tidak tertimpa oleh jawaban lama (*Out-of-Order Queue Protection*).

---

### BAB III.6: Logika Pemrograman PostgreSQL (`PL/pgSQL` Triggers, Procedures & Functions)
**Penanggung Jawab Utama:** `Yosua`

#### 1. Trigger & Function Evaluasi Kartu Ujian Reguler vs. Sementara (`fn_trg_kelola_kartu_ujian`)
Ditulis dalam bahasa prosedural **`PL/pgSQL`** di PostgreSQL:
* **Alur A — Saat Pembayaran SPP Lunas (`AFTER UPDATE ON tagihan_spp`):**
  * Fungsi mengecek apakah siswa masih memiliki tunggakan jatuh tempo:
    $$\text{Total Tunggakan} = \sum \text{sisa\_tagihan} \quad (\text{untuk } \text{tgl\_jatuh\_tempo} \le \text{CURRENT\_DATE})$$
  * Jika $\text{Total Tunggakan} = 0$, maka untuk periode ujian yang sedang aktif, sistem mengubah `status_akses = 'REGULAR_UNLOCKED'`, mengubah `jenis_desain_kartu = 'KARTU_REGULER'`, dan menghapus `batas_waktu_sementara = NULL`.
* **Alur B — Saat Kasir Memberikan Dispensasi (`BEFORE INSERT ON riwayat_override_kasir`):**
  * Memvalidasi bahwa `id_shift_kasir` milik petugas kasir sedang berstatus `'OPEN'`.
  * Memastikan `berlaku_sampai > CURRENT_TIMESTAMP` dan selisih hari tidak melebihi batas kebijakan sekolah (misal maksimal $3$ atau $5\text{ hari}$).
  * Mengubah baris pada `hak_akses_kartu_ujian` untuk periode ujian terkait menjadi:
    * `status_akses = 'TEMPORARY_OVERRIDE'`
    * `jenis_desain_kartu = 'KARTU_SEMENTARA'` (Sistem akan merender kartu ujian dengan *watermark* "KARTU UJIAN SEMENTARA - BERLAKU S.D. [TANGGAL]", warna *border* berbeda, serta QR Code khusus pengawas ruang ujian).
    * `batas_waktu_sementara = NEW.berlaku_sampai`
    * `kali_pengajuan_override = kali_pengajuan_override + 1`
* **Alur C — Kedaluwarsa Otomatis & Kewajiban Pengajuan Ulang (`fn_cek_validitas_kartu_ujian`):**
  * Setiap kali siswa mencoba mengunduh kartu ujian, saat pengawas memindai QR Kartu Ujian, atau saat siswa login ke portal CBT, fungsi PostgreSQL memeriksa:
    ```sql
    IF status_akses = 'TEMPORARY_OVERRIDE' AND CURRENT_TIMESTAMP > batas_waktu_sementara THEN
        UPDATE hak_akses_kartu_ujian
        SET status_akses = 'EXPIRED_OVERRIDE',
            jenis_desain_kartu = 'NONE'
        WHERE id_akses = p_id_akses;
    END IF;
    ```
  * Dengan demikian, tepat ketika batas hari terlewati dan SPP belum dilunasi, status otomatis berubah menjadi `EXPIRED_OVERRIDE` dan siswa wajib datang kembali ke kasir untuk melunasi atau mengajukan dispensasi ulang.

#### 2. Stored Procedure Generate Rapor & Pembekuan PDF Barcode TTD (`sp_terbitkan_rapor_dan_kunci`)
* **Tahap 1 (Kalkulasi & Perakitan Deskripsi):** Menghitung nilai akhir mata pelajaran dan merakit `deskripsi_capaian_otomatis` dari TP tertinggi dan terendah. Jika wali kelas telah mengisi `deskripsi_capaian_custom`, nilai *custom* tersebut dipertahankan.
* **Tahap 2 (Pengesahan Barcode TTD & Render PDF Permanen ke R2):**
  * Ketika Kepala Sekolah menyetujui penerbitan rapor (atau menandatangani Surat Digital pada Fitur 7 & 8), *worker* aplikasi menghasilkan *Digital Signature* berbasiskan $\text{HMAC-SHA256}$, menyematkan Barcode/QR TTD ke dalam dokumen, merender dokumen menjadi **PDF Statis**, dan mengalirkannya via **Laravel Backend Stream** ke Cloudflare R2.
  * Setelah objek tersimpan di R2, prosedur `sp_kunci_dokumen_resmi(p_id_dokumen, p_id_storage_object)` dijalankan di PostgreSQL untuk mengubah `status_berkas = 'ATTACHED'`, `is_immutable_signed_pdf = TRUE`, dan mengunci baris rapor (`is_locked = TRUE`). Selanjutnya, setiap permintaan unduh rapor oleh siswa/orang tua akan langsung mengambil PDF statis yang sudah tervalidasi dari R2 tanpa perlu me-render ulang dari database.

---

### BAB III.7 (Bagian 1): Transaksi Kasir, Aliran Stream R2 & Sinkronisasi Server 1–Server 2
**Penanggung Jawab Utama:** `[Nama Anggota 3]`

#### A. Alur Transmisi Berkas via Laravel Backend Stream & Garbage Collector R2
1. **Tahap Inisiasi & Streaming:**
   * Klien mengirimkan berkas melalui *multipart/form-data* ke *endpoint* API Laravel di **Server 2 ($6\text{C}/6\text{G}$)**.
   * Laravel melakukan validasi tipe MIME dan ukuran berkas di lapisan *request*, lalu terlebih dahulu melakukan `INSERT` baris baru pada tabel `storage_objects` di PostgreSQL dengan `status_berkas = 'PENDING'`.
   * Selanjutnya, Laravel mengalirkan (*stream*) *file* tersebut menggunakan `fopen()` / `putStream()` melalui S3 API menuju *bucket* Cloudflare R2.
2. **Tahap Komitmen Transaksi Relasional:**
   * Di dalam blok `DB::transaction()` PostgreSQL untuk fitur terkait (misal menyimpan data `berkas_ppdb`, `arsip_dokumen_fisik`, atau `surat_digital`), sistem mengaitkan `id_object` ke tabel transaksi dan memperbarui `status_berkas = 'ATTACHED'`.
   * Jika transaksi PostgreSQL gagal (`ROLLBACK`), maka baris `storage_objects` tetap berstatus `'PENDING'` atau diubah menjadi `'ORPHANED'`.
3. **Mekanisme Pembersihan Otomatis (*R2 Garbage Collector*):**
   * Sebuah *Scheduled Job* berjalan setiap malam pukul $02:00\text{ WIB}$ untuk mencari baris pada tabel `storage_objects` di mana:
     $$\text{status\_berkas} \in \{\text{'PENDING'}, \text{'ORPHANED'}\} \quad \land \quad (\text{CURRENT\_TIMESTAMP} - \text{created\_at}) > 24\text{ jam}$$
   * *Worker* akan mengirimkan perintah `DELETE Object` ke API S3 Cloudflare R2 untuk menghapus berkas yatim tersebut, lalu menghapus/menandai barisnya di PostgreSQL.

#### B. Persistensi Jawaban CBT Setiap Pindah Soal (Server 2: Redis $\rightarrow$ PostgreSQL)
Untuk menjamin keandalan saat siswa berpindah dari Soal $k$ ke Soal $k+1$:
1. Saat siswa mengklik nomor soal lain atau tombol "Selanjutnya", klien mengirim *payload* `{id_sesi, id_soal, jawaban, waktu_jawab_client}` ke API Laravel di Server 2.
2. API Laravel langsung menyimpan *state* jawaban ke **Redis Hash** (`cbt:session:{id_sesi}:answers`) dalam waktu $< 2\text{ ms}$, sekaligus memasukkan *Job* `SyncAnswerToPostgresJob` ke **Redis Queue**, lalu segera mengembalikan respons `200 OK` ke siswa agar perpindahan halaman soal terasa instan.
3. *Queue Worker* di Server 2 mengambil *Job* tersebut di latar belakang dan mengeksekusi kueri `UPSERT` (`INSERT ... ON CONFLICT DO UPDATE`) ke tabel PostgreSQL `cbt_jawaban_peserta`.

---

## 5. Rekapitulasi Keputusan Arsitektur Terkunci & Panduan Bedah Excel Rapor (Untuk Yosua)

### A. Tabel Keputusan Arsitektur yang Telah Dikunci (Architecture Decision Record)

| No | Aspek Arsitektur | Keputusan Terkunci (Final) | Implikasi Teknis pada PostgreSQL & Laravel |
| :-: | :--- | :--- | :--- |
| **2** | Deskripsi Rapor Kurikulum Merdeka | **Disediakan Kolom Custom** (`deskripsi_capaian_custom`) | Tabel `rapor_intrakurikuler` menyimpan deskripsi hasil rakitan otomatis *Stored Procedure* sekaligus kolom *custom* yang dapat disunting manual oleh guru sebelum dikunci. |
| **4** | Definisi "Sekolah 1 Atap" | **Satu Yayasan 5 Jenjang (SD, SMP, SMA, MTs, SMK)** dengan versi sistem masing-masing dari *core* yang sama | Menggunakan isolasi **PostgreSQL Schema-per-Jenjang** (`schema_sd`, `schema_smp`, `schema_sma`, `schema_mts`, `schema_smk` + `schema_yayasan`) dan kolom `JSONB` untuk atribut unik jenjang. |
| **5** | Topologi Server & Transmisi R2 | **Server 1 ($4\text{C}/4\text{G}$) & Server 2 ($6\text{C}/6\text{G}$ Backend)** menggunakan **Laravel Backend Stream** | Server 2 menjalankan Laravel + PostgreSQL + Redis dan mengalirkan berkas ke R2 via *chunked stream*. Server 1 menjalankan Moodle & layanan pendukung. |
| **6** | Penanganan *Orphaned Files* R2 | **Ditambahkan Kolom Status Berkas** (`PENDING`, `ATTACHED`, `ORPHANED`) | Tabel `storage_objects` melacak siklus hidup unggahan dan dibersihkan secara berkala oleh *Garbage Collector Cron Job*. |
| **7** | Dokumen Rapor & Surat Ber-Barcode TTD | **Render PDF Statis Permanen ke Cloudflare R2** | Saat dokumen disahkan dengan Barcode TTD, sistem me-render PDF statis, menghitung hash $\text{SHA-256}$, mengunggah ke R2 via *stream*, dan mengunci record DB (`is_locked = TRUE`). |
| **8** | Aturan *Override* Kartu Ujian Kasir | **Berlaku 1 Periode Ujian, Berbatas Waktu Beberapa Hari, Desain Kartu Ujian Sementara Berbeda** | Status `TEMPORARY_OVERRIDE` memiliki `batas_waktu_sementara` dan merender desain khusus. Jika lewat batas hari, otomatis menjadi `EXPIRED_OVERRIDE` dan wajib mengajukan ulang. |
| **9** | Frekuensi Simpan Jawaban CBT | **Setiap Selesai 1 Soal (Saat Berpindah Soal)** | Setiap navigasi soal memicu penulisan instan ke Redis + antrean asinkron `UPSERT` ke PostgreSQL dengan proteksi `waktu_jawab_client`. |
| **10** | Engine RDBMS Utama | **PostgreSQL 16+** | Seluruh DDL, *Trigger*, *Stored Procedure*, dan *Partitioning* pada BAB III.6 & BAB IV.1 wajib menggunakan sintaks standar PostgreSQL & `PL/pgSQL`. |

---

### B. Checklist Investigasi untuk Pertanyaan 1 & 3 (Saat Tim Menerima File Aplikasi Rapor Lama)

Karena jawaban untuk **Pertanyaan 1 (Struktur TP)** dan **Pertanyaan 3 (Struktur P5)** menunggu file aplikasi rapor lama dari pihak sekolah, berikut adalah **4 poin pemeriksaan cepat** yang harus langsung dicek oleh **Yosua** dan **[Nama Anggota 2]** begitu file Excel/aplikasi rapor tersebut diterima:

1. **Cek Sheet Input Tujuan Pembelajaran (TP):**
   * Perhatikan apakah ada satu *sheet* "Master TP" untuk satu tingkat angkatan, atau apakah setiap guru mata pelajaran mengetik sendiri teks TP di masing-masing file kelasnya.
   * Perhatikan batas maksimal jumlah karakter teks TP (di aplikasi e-Rapor Kemendikbud maupun Kemenag umumnya dibatasi maksimal $100 - 150\text{ karakter}$ agar kalimat deskripsi rapor tidak kepanjangan).
2. **Cek Rumus Nilai Akhir (Bobot Formatif vs. Sumatif):**
   * Klik sel "Nilai Akhir" pada file Excel rapor lama dan lihat bilah rumus (*formula bar*): apakah nilai akhir murni rata-rata Sumatif Lingkup Materi + Sumatif Akhir Semester, atau masih mengikutsertakan persentase nilai Formatif?
   * Periksa apakah ada perbedaan rumus antara versi **SD/SMP/SMA/SMK (Kemendikbud)** dengan versi **MTs (Kemenag - Kurikulum Merdeka Madrasah)**.
3. **Cek Struktur Sheet Rapor P5 (dan P5RA untuk MTs):**
   * Khusus untuk jenjang **MTs**, Kurikulum Merdeka menggunakan istilah **P5RA** (*Projek Penguatan Profil Pelajar Pancasila dan Profil Pelajar Rahmatan lil 'Alamin*). Pastikan tabel dimensi projek memiliki kolom opsional untuk nilai *Rahmatan lil 'Alamin* khusus pada skema MTs (`schema_mts`).
4. **Cek Atribut Spesifik Jenjang di Halaman Sampul & Catatan Akhir Rapor:**
   * **SD:** Biasanya tidak memiliki peminatan/jurusan, fokus pada Fase A (Kelas 1–2), Fase B (Kelas 3–4), dan Fase C (Kelas 5–6).
   * **SMP & MTs:** Fase D (Kelas 7–9).
   * **SMA:** Fase E (Kelas 10) dan Fase F (Kelas 11–12 dengan paket mata pelajaran pilihan).
   * **SMK:** Fase E (Kelas 10) dan Fase F (Kelas 11–12), ditambah tabel khusus **Praktik Kerja Lapangan (PKL / Prakerin)** dan **Uji Kompetensi Keahlian (UKK)** di dalam rapor.