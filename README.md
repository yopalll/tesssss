# Product Requirements Document (PRD) & Blueprint Arsitektur MSBD
## Sistem Informasi Manajemen Sekolah 1 Atap Terdistribusi

**Status Dokumen:** Draft Arsitektur Tingkat Tinggi (High-Level Technical Design)  
**Skala Target:** $\ge 1400$ Siswa Aktif, $\pm 120$ Guru & Staf Administrasi  
**Paradigma Arsitektur:** Modular Monolith (API-First) + Distributed E-Learning Node + S3 Object Storage  
**Fokus Akademik:** Tugas Besar Manajemen Sistem Basis Data (MSBD)

---

## 1. Ringkasan Eksekutif & Visi Arsitektur

Proyek ini bertujuan mentransformasi operasional sekolah berskala besar ($\ge 1400$ siswa) dari pengelolaan konvensional berbasis dokumen terpisah (seperti *spreadsheet* Excel untuk Rapor Kurikulum Merdeka dan pencatatan kasir manual) menjadi ekosistem terintegrasi **Sekolah 1 Atap** yang mencakup **15 fitur utama**.

Mengingat keterbatasan sumber daya perangkat keras dan tuntutan performa tinggi pada momen kritis (seperti pelaksanaan ujian serentak pada fitur *Custom Computer-Based Test* / CBT dan penerimaan siswa baru / PPDB), sistem dirancang menggunakan pendekatan **Infrastruktur Terdistribusi Hibrida**:

1. **Core Node (VPS 1 — $6\text{ vCPU} / 6\text{ GB RAM}$):** Menjalankan *backend* **Laravel** dengan arsitektur **Modular Monolith** dan pendekatan **API-First** (*stateless* JWT) guna menjamin kesiapan integrasi Aplikasi Mobile di tahun berikutnya. Node ini menampung Basis Data Relasional Utama (Primary RDBMS) dan *in-memory data structure store* (**Redis**) untuk manajemen antrean (*queue*), *caching*, dan penyangga konkurensi CBT.
2. **E-Learning Node (VPS 2 — $2\text{ vCPU} / 2\text{ GB RAM}$):** Dikhususkan untuk beban kerja **Moodle** (materi pembelajaran statis dan forum diskusi). Terhubung ke ekosistem utama melalui mekanisme sinkronisasi/replikasi parsial untuk data pengguna, kelas, dan kepesertaan (*enrolment*).
3. **Distributed Object Storage (Cloudflare R2):** Mengatasi limitasi *disk I/O* dan kapasitas penyimpanan lokal VPS. Seluruh berkas tak terstruktur berukuran besar (berkas PPDB, hasil pindai arsip fisik, PDF surat digital, dan repositori materi Moodle) disimpan pada penyimpanan objek kompatibel S3 (**Cloudflare R2**). Basis data relasional murni menyimpan *metadata*, atribut integritas (*checksum*), dan *path/URI* objek.

---

## 2. Matriks Pembagian Tugas Tim & Struktur Laporan (WBS)

Pembagian kerja dirancang agar setiap anggota tim memiliki kepemilikan penuh (*end-to-end ownership*) atas domain teknis dan sub-bab laporan Tugas Besar MSBD:

| Anggota Tim | Peran Spesialis | Tanggung Jawab Bab Laporan MSBD | Fokus Teknis & Deliverables Utama |
| :--- | :--- | :--- | :--- |
| **Yosua** | *Backend Architect & Core DB Logic* | **BAB III.4, BAB III.5, BAB III.6** (Serta supervisi arsitektur API-First) | Desain Konseptual/Logis ERD 15 Fitur, Normalisasi ($1\text{NF} \rightarrow 3\text{NF}/\text{BCNF}$) Rapor Kurikulum Merdeka dari Excel, Desain Skema *High-Concurrency* CBT, Spesifikasi *Trigger* Blokir/Override Kartu Ujian, *Stored Procedure* Penilaian, dan Strategi *Cold Storage Archiving*. |
| **[Nama Anggota 2]** | *Systems Analyst & Modeler* | **BAB I, BAB II, BAB III.1, BAB III.2, BAB III.3** | Perencanaan proyek, studi kelayakan (teknis, operasional, jadwal), analisis kelemahan sistem berjalan (Excel/manual), elisitasi *User Requirement* 15 fitur, serta pemodelan *Data Flow Diagram* (DFD Level Konteks hingga Level 2). |
| **[Nama Anggota 3]** | *Infrastructure & Distributed DB Specialist* | **BAB III.7 (Bagian Transaksi), BAB IV.2** (Serta topologi terdistribusi) | Perancangan mekanisme *ACID Transaction* & *Concurrency Control* pada Kasir Offline (termasuk *shift audit log*), topologi replikasi/sinkronisasi DB antara VPS 1 dan VPS 2 (Moodle), arsitektur integrasi API S3 Cloudflare R2, dan skema pemisahan *Hot vs. Cold Storage*. |
| **[Nama Anggota 4]** | *Security & API Architect* | **BAB III.7 (Bagian Keamanan & Hak Akses)** | Desain skema *Multi-Role Role-Based Access Control* (RBAC) untuk peran ganda, matriks *Database User Privilege* (DCL: `GRANT`/`REVOKE`), kebijakan keamanan data tingkat baris/kolom, serta formulasi *Stateless Authentication* (JWT + *Refresh Token Rotation*) untuk kesiapan aplikasi Mobile. |
| **[Nama Anggota 5]** | *UI/UX & Implementation Lead* | **BAB III.8, BAB IV.1, BAB IV.3, BAB V** | Perancangan *wireframe* antarmuka pengguna (terutama pemetaan UI Rapor, Kasir, dan CBT terhadap skema DB), kompilasi skrip implementasi DDL/DML, pengujian fungsional (*black-box*) & pengujian beban (*stress testing* CBT), serta evaluasi kinerja kueri. |

---

## 3. Pemetaan 15 Fitur Utama ke Domain Modul (Modular Monolith)

Untuk menjaga keteraturan kode dan skema basis data pada VPS 1, ke-15 fitur dikelompokkan ke dalam **6 Bounded Contexts (Modul Inti)**:

1. **Modul Akademik & Kurikulum:** (3) Nilai Online, (4) Rapor Otomatis Kurikulum Merdeka, (11) Kalender Akademik, (12) Roster/Jadwal Pelajaran.
2. **Modul Kehadiran & Kepegawaian:** (1) Presensi Siswa, (2) Absensi Guru & Staf.
3. **Modul Keuangan & Kasir:** (5) SPP & Kasir Offline (dengan *Shift Audit* dan *Override* Kartu Ujian).
4. **Modul Evaluasi & CBT:** (6) Online Exam / CBT Custom (*High Concurrency* dengan Redis Queue).
5. **Modul Administrasi, Persuratan & Arsip:** (7) Surat Digital, (8) Barcode TTD (Verifikasi Kriptografis), (9) Direktori Berkas Digital, (10) Arsip Dokumen Fisik (Pemetaan Lokasi Rak/Laci + Berkas Pindai R2).
6. **Modul Ekstensi, Layanan & Integrasi Eksternal:** (13) Pengaduan Warga Sekolah, (14) E-Learning (Integrasi Moodle VPS 2), (15) PPDB Online.

---

## 4. Rancangan Detail Berdasarkan Sistematika Laporan MSBD

### BAB I & BAB II: Perencanaan dan Studi Kelayakan Sistem
**Penanggung Jawab Utama:** `[Nama Anggota 2]`

*   **Identifikasi Masalah (BAB I):** Mendokumentasikan risiko redundansi data, inkonsistensi rumus pada pengelolaan rapor berbasis *file* Excel terpisah per wali kelas, antrean validasi pembayaran SPP manual menjelang masa ujian, serta ketiadaan jejak audit (*audit trail*) pada kasir sekolah.
*   **Studi Kelayakan Teknis & Beban Kerja (BAB II):**
    *   Estimasi ukuran transaksi tahunan: Dengan $N = 1400$ siswa dan $\pm 220$ hari efektif sekolah, tabel presensi siswa akan menerima sedikitnya $1400 \times 220 = 308.000$ baris per tahun ajaran.
    *   Pada pelaksanaan CBT serentak untuk 1 angkatan ($\pm 480$ siswa) atau seluruh sekolah ($\ge 1400$ siswa) dengan rata-rata $40$ butir soal per sesi ujian, sistem harus memproses sedikitnya $1400 \times 40 = 56.000$ operasi penyimpanan jawaban dalam rentang waktu $60 - 90\text{ menit}$.
    *   Analisis efisiensi biaya penyimpanan: Memindahkan beban berkas statis (estimasi $\ge 150\text{ GB}$/tahun dari dokumen PPDB dan arsip) ke Cloudflare R2 menghindari biaya peningkatan *block storage* VPS dan membebaskan *egress bandwidth* dari VPS utama.

---

### BAB III.1 – III.3: Analisis Sistem Berjalan, Kebutuhan Pengguna, dan DFD
**Penanggung Jawab Utama:** `[Nama Anggota 2]`

*   **III.1 Analisis Sistem Berjalan:** Memetakan alur kerja *as-is* terutama pada dua titik kritis: (a) Pengumpulan nilai dari guru mata pelajaran ke wali kelas via *file* Excel, dan (b) Proses verifikasi tunggakan SPP di loket kasir saat siswa meminta cap basah pada Kartu Ujian.
*   **III.2 Spesifikasi Kebutuhan Pengguna (User Requirements):** Merumuskan kebutuhan fungsional (FR) dan non-fungsional (NFR) untuk ke-15 fitur. Setiap aktor (Siswa, Orang Tua, Guru Mapel, Wali Kelas, Petugas Kasir, Staf TU, Kepala Sekolah, Panitia PPDB, Admin Sistem) didefinisikan hak aksesnya.
*   **III.3 Model Analisis (Data Flow Diagram):**
    *   **Diagram Konteks (Level 0):** Menggambarkan aliran data antara entitas eksternal (Siswa, Guru, Kasir, Kepala Sekolah, Pendaftar PPDB) dengan sistem terpusat.
    *   **DFD Level 1:** Memecah sistem ke dalam 6 proses utama sesuai klaster modul di atas, memperlihatkan aliran data menuju *Data Store* utama dan penyimpanan eksternal.
    *   **DFD Level 2:** Diwajibkan merinci minimal tiga proses paling kompleks: (1) Proses Pengolahan Nilai & Generate Rapor Kurikulum Merdeka, (2) Proses Transaksi Kasir, Rekonsiliasi Shift, dan Validasi Kartu Ujian, serta (3) Proses Pelaksanaan CBT & Sinkronisasi Antrean Jawaban.

---

### BAB III.4 & III.5: Perancangan Basis Data (ERD & Normalisasi)
**Penanggung Jawab Utama:** `Yosua`

#### A. Strategi Normalisasi Rapor Kurikulum Merdeka (Dari Excel ke Relasional)
Pada sistem lama berbasis Excel, satu *sheet* rapor biasanya menggabungkan identitas siswa, identitas kelas, daftar seluruh mata pelajaran, kolom berulang untuk Tujuan Pembelajaran ($\text{TP}_1, \text{TP}_2, \dots, \text{TP}_n$), nilai formatif, nilai sumatif lingkup materi, nilai sumatif akhir semester, hingga teks deskripsi capaian tertinggi dan terendah yang digenerate dengan rumus `VLOOKUP`/`CONCATENATE`.

Yosua harus mendokumentasikan tahapan dekomposisi skema secara matematis dan struktural:
1.  **Bentuk Tidak Normal (UNF) $\rightarrow$ Bentuk Normal Pertama ($1\text{NF}$):**
    *   *Masalah Excel:* Adanya *repeating groups* pada kolom Tujuan Pembelajaran ($\text{TP}_1 \dots \text{TP}_k$), kolom Lingkup Materi ($\text{LM}_1 \dots \text{LM}_m$), dan dimensi Projek Penguatan Profil Pelajar Pancasila (P5).
    *   *Solusi $1\text{NF}$:* Menghilangkan kolom berulang dengan memisahkan setiap butir penilaian TP dan Sumatif menjadi baris atomik tersendiri.
2.  **Bentuk Normal Pertama ($1\text{NF}$) $\rightarrow$ Bentuk Normal Kedua ($2\text{NF}$):**
    *   *Masalah:* Ketergantungan fungsional parsial (*partial dependency*), di mana atribut non-kunci seperti `nama_siswa`, `nisn`, `nama_mapel`, `fase_kurikulum`, dan `deskripsi_tp` hanya bergantung pada sebagian dari *composite primary key* `(id_siswa, id_mapel, id_semester, no_urut_tp)`.
    *   *Solusi $2\text{NF}$:* Memecah tabel menjadi entitas master dan referensi: `siswa`, `mata_pelajaran`, `tahun_ajaran_semester`, `master_capaian_pembelajaran (CP)`, dan `master_tujuan_pembelajaran (TP)`.
3.  **Bentuk Normal Kedua ($2\text{NF}$) $\rightarrow$ Bentuk Normal Ketiga ($3\text{NF}$) & BCNF:**
    *   *Masalah:* Ketergantungan transitif (*transitive dependency*), seperti:
        $$\text{id\_kelas\_rombel} \rightarrow \text{id\_guru\_wali\_kelas} \rightarrow \text{nama\_wali\_kelas}, \text{nip\_wali\_kelas}$$
        Serta atribut kalkulasi:
        $$\{\text{nilai\_tp\_tertinggi}, \text{id\_tp\_tertinggi}\} \rightarrow \text{teks\_deskripsi\_capaian\_tertinggi}$$
    *   *Solusi $3\text{NF}$/BCNF:* Memisahkan tabel penugasan `rombel_kelas` dari master `pegawai_guru`, serta menormalisasi struktur penilaian menjadi:
        *   `kurikulum_fase` (Fase A s.d. Fase F, dipetakan ke tingkat kelas).
        *   `capaian_pembelajaran` (Elemen dan CP per mata pelajaran & fase).
        *   `tujuan_pembelajaran` (Butir TP turunan dari CP yang diinput guru).
        *   `nilai_formatif_tp` (Menyimpan skor/ketercapaian siswa per TP, menjadi dasar penentuan otomatis deskripsi capaian tertinggi dan terendah di rapor).
        *   `nilai_sumatif` (Skor sumatif per lingkup materi serta Sumatif Akhir Semester / SAS).
        *   `rapor_intrakurikuler` (Tabel agregat akhir per mata pelajaran per siswa per semester yang menyimpan nilai akhir terhitung serta teks deskripsi yang telah dikunci/*frozen* saat rapor diterbitkan).
        *   `projek_p5`, `dimensi_elemen_p5`, dan `nilai_rapor_p5` (Menangani rapor P5 yang bersifat kualitatif: *Mulai Berkembang* [MB], *Sedang Berkembang* [SB], *Berkembang Sesuai Harapan* [BSH], *Sangat Berkembang* [SAB]).

#### B. Rancangan Skema High-Concurrency untuk CBT Custom
Untuk mencegah *database lock contention* dan *deadlock* ketika $\ge 1400$ siswa menekan tombol pindah soal/simpan jawaban secara bersamaan:
*   **Pemisahan Tabel Master Soal dan Sesi Ujian:** Entitas `bank_soal`, `butir_soal`, dan `opsi_jawaban` bersifat *read-heavy* dan di-*cache* sepenuhnya di **Redis** sebelum sesi ujian dimulai.
*   **Arsitektur *Write-Behind* pada Tabel Jawaban:** Saat siswa menjawab soal di antarmuka CBT, *request* API menulis perubahan status jawaban ke struktur *Hash/Stream* di Redis (latensi $\mathcal{O}(1)$) dan mendorong *job* ke **Laravel Queue**. *Worker* antrean kemudian melakukan operasi *Bulk Upsert* (`INSERT ... ON DUPLICATE KEY UPDATE` atau `ON CONFLICT DO UPDATE`) secara periodik ke tabel relasional `cbt_jawaban_peserta`.
*   **Desain Indeks Komposit:** Tabel `cbt_jawaban_peserta` wajib memiliki *Unique Composite Index* pada `(id_sesi_peserta, id_butir_soal)` tanpa *foreign key cascade* berlapis yang berat guna memaksimalkan kecepatan *write throughput*.

#### C. Entitas Penyimpanan Berkas (Cloudflare R2 Metadata Abstraction)
Daripada menyebar kolom `file_url` di puluhan tabel berbeda, Yosua merancang satu tabel terpusat `storage_objects` (atau pendekatan *Polymorphic File Attachment*):
*   Atribut: `id_object (UUID)`, `bucket_name`, `object_key (path di R2)`, `original_filename`, `mime_type`, `file_size_bytes`, `sha256_checksum`, `visibility (PUBLIC/PRIVATE)`, `uploaded_by`, `created_at`.
*   Tabel fitur seperti `surat_digital`, `arsip_dokumen_fisik` (untuk hasil scan), `berkas_ppdb`, dan `direktori_berkas` cukup merujuk ke `id_object` melalui *Foreign Key*.

---

### BAB III.6: Logika Pemrograman Basis Data (Triggers, Stored Procedures & Archiving)
**Penanggung Jawab Utama:** `Yosua`

Bagian ini mendemonstrasikan kapabilitas RDBMS dalam menegakkan aturan bisnis (*business rules*) secara langsung di lapisan data, sehingga integritas tetap terjaga meskipun diakses dari berbagai *endpoint* API:

#### 1. Trigger Otomatisasi Blokir & Override Kartu Ujian
*   **Konteks Aturan Bisnis:** Secara *default*, hak cetak kartu ujian (`status_cetak_kartu`) bernilai `BLOCKED` apabila siswa memiliki tagihan SPP jatuh tempo yang belum lunas (`status_tagihan != 'LUNAS'`). Namun, petugas kasir yang sedang bertugas dalam *shift* aktif dapat memberikan dispensasi (*override access*).
*   **Mekanisme Trigger (`trg_evaluasi_akses_kartu_ujian` & `trg_validasi_override_kasir`):**
    *   **Kondisi 1 (Pelunasan Normal):** *After Update Trigger* pada tabel `tagihan_spp`. Ketika tagihan terakhir yang menunggak berubah status menjadi `'LUNAS'`, *trigger* mengecek apakah masih ada tunggakan lain pada periode berjalan; jika jumlah tunggakan $= 0$, kolom `is_eligible_exam_card` pada tabel `status_akademik_siswa` otomatis diperbarui menjadi `TRUE`.
    *   **Kondisi 2 (Intervensi / Override Kasir):** Ketika kasir mengaktifkan flag `is_override_allowed = TRUE` pada tabel `dispensasi_kartu_ujian`, sebuah *Before Insert/Update Trigger* akan memvalidasi tiga syarat mutlak:
        1. `id_shift_kasir` yang menyetujui harus berstatus `'OPEN'` (aktif) pada saat transaksi dilakukan.
        2. Kolom `alasan_override` dan `tanggal_batas_dispensasi` (atau dokumen perjanjian) tidak boleh bernilai `NULL`.
        3. Secara otomatis menyisipkan baris *immutable* ke dalam tabel `audit_log_keuangan` yang mencatat `id_kasir`, `id_siswa`, `nominal_tunggakan_saat_override`, dan `timestamp`.

#### 2. Stored Procedure Kalkulasi Nilai & Penerbitan Rapor (`sp_kalkulasi_rapor_merdeka`)
*   Menerima parameter `(p_id_rombel, p_id_semester, p_id_mapel)`.
*   Mengagregasi rata-rata nilai Sumatif Lingkup Materi ($\bar{S}_{LM}$) dan nilai Sumatif Akhir Semester ($S_{AS}$) berdasarkan bobot konfigurasi sekolah ($w_1, w_2$ di mana $w_1 + w_2 = 1$):
    $$\text{Nilai Akhir (NA)} = \text{ROUND}\left( w_1 \cdot \left(\frac{1}{m}\sum_{i=1}^{m} S_{LM, i}\right) + w_2 \cdot S_{AS} \right)$$
*   Mencari nilai maksimum dan minimum dari tabel `nilai_formatif_tp` untuk siswa bersangkutan guna merangkai kalimat deskripsi capaian tertinggi dan capaian yang membutuhkan bimbingan secara otomatis, lalu menyimpannya ke tabel `rapor_intrakurikuler`.

#### 3. Strategi Data Archiving Tahunan (Cold Storage Strategy)
*   Mengingat pertumbuhan baris yang masif pada tabel `presensi_siswa`, `absensi_guru`, `cbt_jawaban_peserta`, dan `log_transaksi_kasir`, sistem menerapkan **Table Partitioning** berdasarkan `id_tahun_ajaran` (`RANGE` atau `LIST Partitioning`).
*   Setiap pergantian tahun ajaran baru (setelah masa tutup buku rapor dan keuangan selesai), *Stored Procedure* `sp_archive_tahun_ajaran(p_id_tahun_ajaran)` akan dijalankan untuk memindahkan partisi data transaksi historis yang berusia $> 2\text{ tahun}$ dari skema utama (`schema_operational_hot`) ke skema arsip (`schema_archive_cold`) atau mengekspor *dump* terkompresi Parquet/SQL ke *bucket* khusus di Cloudflare R2, sembari menyisakan ringkasan agregat (seperti rekap persentase kehadiran semester dan transkrip nilai rapor) di basis data utama.

---

### BAB III.7 (Bagian 1): Transaksi Basis Data Kasir & Arsitektur Terdistribusi
**Penanggung Jawab Utama:** `[Nama Anggota 3]`

#### A. Desain Transaksi Kasir Offline & Audit Shift (ACID Compliance)
Karena pembayaran SPP dilakukan secara *offline* (tunai atau bukti transfer fisik di loket kasir tanpa *Payment Gateway* otomatis), risiko selisih kas (*discrepancy*) dan manipulasi data sangat tinggi.
1.  **Manajemen Siklus Hidup Shift Kasir (`shift_kasir`):**
    *   Setiap petugas kasir wajib membuka *shift* sebelum dapat memproses pembayaran (`waktu_buka`, `saldo_awal_kas_fisik`, `status_shift = 'OPEN'`).
    *   Seluruh transaksi pembayaran SPP (`transaksi_pembayaran_spp`) wajib memiliki *Foreign Key* `id_shift_kasir` yang merujuk pada *shift* yang sedang terbuka.
    *   Saat penutupan *shift* (`status_shift = 'CLOSED'`), sistem menghitung total penerimaan sistem ($\sum \text{nominal\_bayar}$) dan membandingkannya dengan uang fisik yang dihitung kasir (`saldo_akhir_kas_fisik`), mencatat selisihnya (`selisih_kas`), serta mengunci seluruh baris transaksi pada *shift* tersebut agar tidak dapat diubah/dihapus (*immutable*).
2.  **Kontrol Konkurensi & Isolation Level:**
    *   Transaksi pembayaran menggunakan tingkat isolasi **`REPEATABLE READ`** atau **`SERIALIZABLE`** dengan mekanisme *Pessimistic Locking* (`SELECT ... FOR UPDATE`) pada baris `tagihan_spp` yang sedang dibayar.
    *   Hal ini mencegah anomali *Double Payment* atau *Lost Update* apabila orang tua siswa dan siswa secara tidak sengaja melunasi tagihan yang sama di dua loket kasir yang berbeda pada waktu bersamaan.
    *   Operasi pembayaran dibungkus dalam blok transaksi atomik:
        1. Validasi status `shift_kasir` masih `'OPEN'`.
        2. Kunci baris `tagihan_spp` (`FOR UPDATE`).
        3. Catat rincian pembayaran di `transaksi_pembayaran_spp`.
        4. Perbarui `sisa_tagihan` dan `status_tagihan` pada `tagihan_spp`.
        5. Catat jejak mutasi pada `jurnal_audit_kasir`.
        6. Evaluasi *Trigger* pembukaan blokir Kartu Ujian.
        7. `COMMIT` (atau `ROLLBACK` penuh jika salah satu langkah gagal).

#### B. Topologi Sistem Terdistribusi (VPS 1 vs VPS 2)
*   **Pemisahan Beban Kerja:** VPS 1 ($6\text{C}/6\text{G}$) menangani beban transaksional (OLTP) tinggi dari 15 fitur inti, sementara VPS 2 ($2\text{C}/2\text{G}$) dikhususkan untuk Moodle agar lonjakan unduhan materi atau diskusi forum tidak menguras memori Redis dan RDBMS Utama saat ujian CBT berlangsung.
*   **Mekanisme Integrasi Data VPS 1 $\leftrightarrow$ VPS 2:**
    *   *Single Sign-On (SSO) & Provisioning:* Basis data utama di VPS 1 bertindak sebagai *Single Source of Truth* (SSOT) untuk entitas `users`, `siswa`, `guru`, `rombel_kelas`, dan `mata_pelajaran`.
    *   Sinkronisasi ke VPS 2 dilakukan menggunakan **Database View Tereplikasi Terbatas (Read-Only Replica / Foreign Data Wrapper)** atau *Event-Driven Webhook* melalui antrean Redis di VPS 1 yang memanggil *Moodle External Database Enrolment / REST Web Services* setiap kali terjadi perubahan susunan kelas atau pengguna di awal semester.

#### C. Arsitektur Integrasi S3 API untuk Cloudflare R2
*   Untuk menghindari *bottleneck* memori dan jaringan pada VPS 1 maupun VPS 2 saat ribuan siswa mengunggah berkas PPDB atau mengunduh materi pembelajaran:
    *   **Alur Unggah (Upload) & Unduh (Download):** Aplikasi Laravel menggunakan pola **Pre-Signed URLs** (melalui protokol AWS S3 SDK kompatibel R2). Klien (Web/Mobile) meminta izin unggah ke API Laravel $\rightarrow$ Laravel memvalidasi kuota dan otorisasi pengguna, membuat baris *pending* pada tabel `storage_objects`, lalu mengembalikan *Pre-Signed PUT URL* berdurasi singkat (misal $5\text{ menit}$) $\rightarrow$ Klien mengunggah berkas langsung ke *edge network* Cloudflare R2 $\rightarrow$ Setelah selesai, klien mengirim konfirmasi ke API Laravel untuk memverifikasi keberadaan objek (`HEAD Object`) dan mengaktifkan status berkas di basis data.
    *   **Moodle di VPS 2:** Dikonfigurasi menggunakan *plugin* **Object Storage (`tool_objectfs`)** yang diarahkan ke *bucket* Cloudflare R2 sehingga direktori `moodledata/filedir` lokal di VPS 2 tidak membengkak melebihi kapasitas disk $2\text{C}/2\text{G}$.

---

### BAB III.7 (Bagian 2): Manajemen User, Multi-Role Privilege & Keamanan API
**Penanggung Jawab Utama:** `[Nama Anggota 4]`

#### A. Skema Multi-Role RBAC (Role-Based Access Control)
Satu akun pegawai di sekolah sering kali memegang beberapa peran sekaligus dengan cakupan data (*data scope*) yang berbeda—misalnya: seorang pengguna adalah **Guru Mata Pelajaran** (hanya boleh menginput nilai pada kelas yang diajarnya), sekaligus **Wali Kelas X-A** (boleh mencetak rapor seluruh mapel tetapi *hanya* untuk siswa di kelas X-A), sekaligus **Panitia PPDB**.

Desain tabel keamanan meliputi:
1.  `users`: Menyimpan kredensial autentikasi utama (`id_user`, `username/nip/nisn`, `password_hash`, `is_active`).
2.  `roles`: Daftar peran sistem (`SUPER_ADMIN`, `KEPALA_SEKOLAH`, `GURU_MAPEL`, `WALI_KELAS`, `KASIR`, `STAF_TU`, `SISWA`, `ORANG_TUA`, `PANITIA_PPDB`).
3.  `permissions`: Hak akses atomik (`nilai.input`, `rapor.generate`, `spp.terima_bayar`, `kartu_ujian.override`, `surat.tanda_tangan`).
4.  `role_permissions`: Pemetaan *Many-to-Many* antara peran dan hak akses.
5.  `user_roles_context` (*Contextual Role Assignment*): Memetakan `id_user` dengan `id_role` beserta **atribut pembatas cakupan (*scope*)** seperti `id_tahun_ajaran`, `id_rombel_kelas` (khusus Wali Kelas), atau `id_pengampu_mapel` (khusus Guru Mapel).

#### B. Keamanan Tingkat Basis Data (Database-Level Security)
Untuk memenuhi standar keamanan MSBD, aplikasi tidak boleh terhubung ke RDBMS menggunakan akun `root` atau `superuser`:
*   **Pemisahan Akun Koneksi RDBMS:**
    *   `db_user_app_oltp`: Digunakan oleh aplikasi utama Laravel (hanya memiliki hak `SELECT, INSERT, UPDATE` pada tabel operasional; **dilarang** melakukan `DROP`, `TRUNCATE`, atau `DELETE` pada tabel `log_transaksi_kasir` dan `audit_log_keuangan`).
    *   `db_user_moodle_sync`: Akun *Read-Only* (`SELECT` saja) yang hanya diizinkan membaca *View* `vw_moodle_users` dan `vw_moodle_enrolments` dari IP privat VPS 2.
    *   `db_user_migration_admin`: Akun khusus untuk eksekusi DDL dan prosedur *archiving* tahunan.
*   **Keamanan Barcode TTD (Fitur 8):** Pada fitur Surat Digital dan Rapor Otomatis, kode QR/Barcode tidak sekadar menyimpan URL statis, melainkan menyimpan *Digital Signature Hash* ($\text{HMAC-SHA256}$) dari atribut `(id_dokumen, nip_penandatangan, timestamp_pengesahan, hash_file_r2)`. Saat dipindai, sistem memverifikasi keaslian dokumen langsung terhadap catatan basis data.

#### C. Arsitektur Stateless Authentication (JWT) untuk Kesiapan Mobile
*   Menggunakan skema **Dual-Token**:
    *   **Access Token (JWT):** Berumur pendek ($15\text{ menit}$), ditandatangani secara asimetris ($\text{RS256}$), memuat *payload* ringan berisi `sub (id_user)`, `active_role`, dan `context_scope_id`. Memungkinkan pengguna yang memiliki peran ganda melakukan *Role Switching* tanpa perlu login ulang.
    *   **Refresh Token (Opaque Token):** Berumur panjang ($7 - 30\text{ hari}$), disimpan dalam bentuk *hash* di tabel `user_sessions` (dilengkapi informasi `device_id`, `ip_address`, `user_agent`) dan di-*cache* di Redis. Mendukung mekanisme *Refresh Token Rotation* dan *Instant Revocation* (pencabutan akses seketika jika siswa lulus/keluar atau perangkat staf hilang).

---

### BAB III.8 & BAB IV: Desain Antarmuka, Implementasi SQL, dan Pengujian Sistem
**Penanggung Jawab Utama:** `[Nama Anggota 5]` (Berkolaborasi dengan `[Nama Anggota 3]` pada BAB IV.2)

*   **III.8 Desain Antarmuka (UI/UX Mapping):**
    *   Merancang *wireframe* berfokus pada ergonomi data: (1) Dashboard Kasir (menampilkan status *shift*, rincian tunggakan siswa, tombol pelunasan cepat, dan modal otorisasi *override* Kartu Ujian), (2) Antarmuka Input Nilai & Pratinjau Rapor Kurikulum Merdeka (menggantikan tampilan *grid* Excel tanpa membuat guru kebingungan), dan (3) Antarmuka CBT Siswa (indikator sinkronisasi antrean jawaban secara *real-time*).
*   **IV.1 Implementasi SQL (DDL & DML):**
    *   Menyusun skrip pembuatan skema tabel lengkap dengan batasan integritas (*Primary Key*, *Foreign Key*, `CHECK` constraints untuk rentang nilai $0 \le \text{nilai} \le 100$, serta indeks pencarian).
*   **IV.2 Konfigurasi Infrastruktur & Basis Data Terdistribusi (Bersama Anggota 3):**
    *   Dokumentasi konfigurasi koneksi RDBMS Utama di VPS 1, konfigurasi *worker* Redis untuk CBT, koneksi sinkronisasi dari VPS 2, serta pengujian *endpoint* S3 Cloudflare R2.
*   **IV.3 Pengujian & Evaluasi Kinerja Basis Data:**
    *   **Analisis Rencana Eksekusi Kueri (`EXPLAIN ANALYZE`):** Membandingkan waktu eksekusi kueri cetak rapor dan rekap presensi sebelum dan sesudah penerapan *Composite Indexing*.
    *   **Simulasi Pengujian Konkurensi & Transaksi:** Menguji skenario *race condition* pada loket kasir dan menguji ketahanan *Trigger* blokir Kartu Ujian terhadap berbagai *edge cases*.
    *   **Stress Testing CBT:** Mensimulasikan beban konkuren hingga $\ge 1400$ *virtual users* untuk membuktikan efektivitas *buffer* Redis dalam menjaga penggunaan CPU RDBMS di bawah ambang aman ($< 70\%$).

---

## 5. Daftar Pertanyaan Klarifikasi Teknis untuk ITLC (Keputusan Pra-Implementasi)

Sebagai **Backend Architect & Core DB Logic Lead**, **Yosua** memegang kendali atas fondasi skema basis data dan kontrak integrasi penyimpanan. Sebelum tim mulai menulis skrip DDL (`CREATE TABLE`) maupun *migration* Laravel, Yosua perlu menetapkan keputusan atas **10 pertanyaan arsitektural** berikut:

### Kategori A: Struktur DDL Kurikulum Merdeka & Akademik
1.  **Fleksibilitas Tujuan Pembelajaran (TP) pada Rapor:**
    *   Pada file Excel sekolah saat ini, apakah daftar Tujuan Pembelajaran (TP) bersifat **terpusat/baku per tingkat kelas**, ataukah **setiap guru mata pelajaran diizinkan merumuskan butir TP yang berbeda-beda** untuk masing-masing rombel yang diajarnya?
    *   *Dampak DDL:* Ini menentukan apakah tabel `tujuan_pembelajaran` berelasi langsung ke `mata_pelajaran + tingkat_kelas` (skema terpusat) atau harus berelasi ke `pengampu_jadwal_mengajar` (skema fleksibel per guru).
2.  **Mekanisme Pembentukan Deskripsi Capaian Rapor:**
    *   Dalam Kurikulum Merdeka, deskripsi rapor terdiri dari kalimat capaian tertinggi dan capaian yang perlu bimbingan. Apakah teks deskripsi di tabel `rapor_intrakurikuler` akan **di-generate murni secara dinamis via Stored Procedure** (menggabungkan *template* kalimat dengan teks TP tertinggi/terendah), ataukah wali kelas/guru tetap diberikan kolom `teks_deskripsi_custom` untuk menyunting kalimat tersebut secara manual sebelum rapor dikunci?
3.  **Skema Penilaian Projek Penguatan Profil Pelajar Pancasila (P5):**
    *   Berapa jumlah tema P5 yang dijalankan sekolah dalam satu tahun ajaran, dan apakah penilaian P5 dilakukan oleh **Wali Kelas** atau oleh **Koordinator/Fasilitator Projek khusus** yang lintas kelas?
    *   *Dampak DDL:* Jika lintas kelas, kita memerlukan tabel kelompok belajar khusus (`kelompok_p5` dan `fasilitator_p5`) yang terpisah dari struktur `rombel_kelas` reguler.
4.  **Struktur Jenjang "Sekolah 1 Atap":**
    *   Apakah istilah "Sekolah 1 Atap" pada studi kasus ini berarti satu jenjang terpadu (misal: SMA saja dengan 1400 siswa), ataukah **multi-jenjang dalam satu yayasan/kompleks** (misal: SD + SMP + SMA/SMK)?
    *   *Dampak DDL:* Jika multi-jenjang, tabel `kurikulum_fase` harus mengakomodasi Fase A s.d. Fase F sekaligus, dan entitas `siswa` membutuhkan riwayat kelulusan lintas jenjang internal (integrasi langsung dengan fitur 15: PPDB Online jalur internal).

### Kategori B: Batas Integrasi Cloudflare R2 & Manajemen Berkas
5.  **Pola Transmisi Berkas (Pre-Signed URL vs. Backend Stream Proxy):**
    *   Untuk pengunggahan dokumen sensitif (seperti berkas KK/Ijazah pada PPDB dan hasil pindai Arsip Dokumen Fisik), apakah Yosua menyetujui penggunaan **Direct Client-to-R2 Upload via Pre-Signed URL** (hemat RAM/CPU VPS 1, namun membutuhkan alur *callback* konfirmasi status di DB), ataukah memilih **Upload via Laravel Backend Stream** (lebih mudah memvalidasi *file signature* dan *virus scanning* di server, tetapi membebani *worker* VPS 1)?
6.  **Strategi Garbage Collection untuk *Orphaned Files* di R2:**
    *   Karena RDBMS di VPS 1 dan Object Storage di Cloudflare R2 tidak berbagi transaksi ACID lintas jaringan, bagaimana skema tabel `storage_objects` menangani kegagalan parsial (misal: berkas berhasil terunggah ke R2, namun transaksi `INSERT` pada tabel `surat_digital` mengalami `ROLLBACK`)?
    *   *Usulan Keputusan:* Apakah Yosua akan menambahkan kolom `status_berkas ENUM('PENDING', 'ATTACHED', 'ORPHANED')` pada tabel `storage_objects` untuk dibersihkan secara berkala oleh *scheduled cron job*?
7.  **Penyimpanan File Rapor PDF & Surat Digital yang Telah Ditandatangani (Barcode TTD):**
    *   Ketika Rapor Otomatis (Fitur 4) atau Surat Digital (Fitur 7) disahkan dengan Barcode TTD (Fitur 8), apakah sistem akan **merender dokumen menjadi PDF statis dan mengunggahnya secara permanen ke Cloudflare R2** (sehingga dokumen bersifat *immutable* selamanya), ataukah PDF selalu **di-generate secara *on-the-fly*** dari baris database setiap kali diunduh?

### Kategori C: Aturan Bisnis Kasir, Trigger & Konkurensi CBT
8.  **Granularitas Izin Cetak Kartu Ujian (*Override Scope*):**
    *   Ketika petugas kasir memberikan *override* izin cetak Kartu Ujian bagi siswa yang menunggak SPP, apakah izin tersebut berlaku untuk **satu periode ujian spesifik saja** (misal: hanya untuk UTS Ganjil, sehingga saat UAS Ganjil otomatis terblokir kembali jika belum lunas), atau berlaku untuk **satu semester penuh**?
    *   *Dampak DDL & Trigger:* Menentukan apakah *Trigger* mengevaluasi status blokir terhadap tabel `periode_ujian_aktif` atau terhadap tabel `semester_akademik`.
9.  **Batas Persistensi Jawaban CBT dari Redis ke RDBMS Utama:**
    *   Pada desain CBT Custom, kapan data jawaban sementara di Redis ditulis (*flushed*) ke tabel relasional `cbt_jawaban_peserta` di PostgreSQL/MySQL? Apakah dilakukan secara **asinkron setiap $N$ detik / setiap kali siswa berpindah soal melalui Queue Worker**, ataukah **hanya disinkronkan ke RDBMS saat siswa menekan tombol "Selesai Ujian" (atau saat waktu habis)**?
10. **Pemilihan Engine RDBMS Utama (PostgreSQL vs. MySQL/MariaDB):**
    *   Untuk VPS 1 ($6\text{C}/6\text{G}$), engine RDBMS mana yang ditetapkan Yosua sebagai standar tim?
    *   *Catatan Arsitek:* Keputusan ini akan menentukan sintaks spesifik yang harus ditulis tim pada BAB III.6 dan BAB IV.1 (misalnya penggunaan `JSONB`, *Table Partitioning* deklaratif, dan sintaks *Trigger* `PL/pgSQL` pada **PostgreSQL** dibandingkan sintaks prosedural pada **MySQL 8.0 / MariaDB**).
