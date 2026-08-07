# Bab 19 — ETL Subsystems & Techniques (34 Subsystem)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab terpanjang buku — memetakan **34 subsystem** yang membentuk arsitektur setiap sistem ETL, dikelompokkan dalam empat komponen: **Extracting** (1-3), **Cleaning & Conforming** (4-8), **Delivering** (9-21), dan **Managing** (22-34). Didahului **10 area requirement** yang harus dikumpulkan sebelum mendesain ETL.

---

## 1. Round Up the Requirements (10 Area)

Sebelum mendesain, kumpulkan requirement (mayoritas **constraint** yang harus diadaptasi):
1. **Business needs** — kebutuhan informasi user (KPI + target drill-down/across).
2. **Compliance** — daftar data/laporan yang tunduk regulasi; **chain of custody**.
3. **Data quality** — daftar elemen berkualitas buruk; kesepakatan koreksi di sumber.
4. **Security** — DW cenderung publikasi luas vs security membatasi; termasuk backup.
5. **Data integration** — conforming dimension & fact (pakai bus matrix untuk prioritas).
6. **Data latency** — seberapa cepat data harus tersaji; batch → microbatch → streaming = **paradigm shift** (bukan evolusi bertahap).
7. **Archiving & lineage** — stage tiap tahap; arsipkan kecuali dipastikan tak akan dipulihkan; sertai metadata asal-usul.
8. **BI delivery interfaces** — tim ETL bertanggung jawab atas struktur yang membuat BI **sederhana & cepat**; ⚠️ kesalahan serius = serahkan model ternormalisasi penuh lalu pergi.
9. **Available skills** — inventaris keahlian (OS, ETL tool, scripting, SQL, DBMS, OLAP); keputusan hand-code vs vendor tool.
10. **Legacy licenses** — lisensi yang wajib/direkomendasikan dipakai.

---

## 2. Extracting (Subsystem 1-3)

**Subsystem 1 — Data Profiling.** Analisis teknis konten, konsistensi, struktur data. Dua peran: **strategis** (asesmen ringan segera saat kandidat sumber teridentifikasi → go/no-go dini; diskualifikasi dini = tanggung jawab) & **taktis** (upaya panjang saat modeling→ETL design untuk memeras masalah). Lakukan **di depan**; hasilnya menentukan seberapa canggih cleansing yang dibutuhkan & mengelola ekspektasi sponsor.

**Subsystem 2 — Change Data Capture (CDC).** Transfer hanya data yang **berubah** sejak load terakhir (tabel terlalu besar untuk full refresh). Tujuan: isolasi perubahan, tangkap semua (delete/edit/insert termasuk via interface non-standar), tag reason code, dukung compliance, lakukan sedini mungkin. Metode: **audit columns** (date/time add/modify; hati-hati bila diisi aplikasi bukan trigger, & delete tak tertangkap), **timed extract** (⚠️ **tidak andal** — duplikat saat restart, data hilang bila skip hari), **full diff compare** (menyeluruh tapi mahal; lakukan di source, pakai CRC checksum), **log scraping/sniffing** (paling berantakan; log bisa penuh & dikosongkan DBA → transaksi hilang), **message queue monitoring** (overhead rendah tapi tanpa replay → koneksi putus = data hilang).

**Subsystem 3 — Extract System.** Ekstrak dari beragam sumber (mainframe: COBOL copybook, EBCDIC→ASCII, packed decimal; RDBMS, flat file, XML, web log, ERP). Dua metode: **file** (mudah restart, bisa encrypt/compress, verifikasi row count) vs **stream** (data mengalir langsung ke staging). ⚠️ XML boros untuk transfer besar (payload <10%). Big data → MapReduce/Hadoop sebagai fact extractor. **Compress sebelum encrypt** (file terenkripsi tak compress baik).

---

## 3. Cleaning & Conforming (Subsystem 4-8)

**Budaya kualitas data** — kesalahan hilir sering = **indikasi proses bisnis yang rusak** (Michael Hammer). Solusi teknis tak cukup tanpa **budaya kualitas dari atas** (analogi manufaktur mobil Jepang). Template 9 langkah: komitmen tinggi, reengineering di level eksekutif, investasi lingkungan entry/integrasi/proses, kesadaran tim end-to-end, kerja sama antar-departemen, rayakan keunggulan, ukur terus.

**Subsystem 4 — Data Cleansing.** Seimbangkan "perbaiki data kotor" vs "gambaran akurat data sebagaimana ditangkap". Inti: **quality screens** (filter diagnostik) 3 tipe (Jack Olson): **column** (single column: null, range, format), **structure** (antar-kolom: hierarki, FK/PK, alamat), **business rule** (kompleks/threshold). Respons error: (1) halt, (2) suspense file, (3) **tag data** (terbaik — via audit dimension).

**Subsystem 5 — Error Event Schema.** Skema dimensional terpusat merekam **setiap error** dari quality screen. **Grain = satu error** per baris di error event fact (dimensi: date, batch, screen; fakta: severity, date/time). Ada **error event detail fact** grain lebih rendah (per field per record).

**Subsystem 6 — Audit Dimension Assembler.** Rakit audit dimension per fact table (metadata konteks saat baris fakta dibuat: quality rating, flag, versi ETL/alokasi). "Metadata dinaikkan jadi data nyata." Run sempurna = satu baris audit; ada error out-of-bounds → baris audit tambahan.

**Subsystem 7 — Deduplication.** Merge dimensi dari banyak sumber (customer dari banyak LOB). Match via key identik atau **fuzzy** (nama/alamat mirip). **Survivorship** = gabungkan kolom kualitas tertinggi jadi satu baris conform (aturan prioritas per sumber); simpan back-reference natural key ke semua sumber.

**Subsystem 8 — Conforming System.** Selaraskan kolom dimensi dengan dimensi serupa di bagian lain (align domain). Mengikuti pendekatan **agile**: dua dimensi conform bila berbagi ≥1 atribut (nama & konten sama); tanam inkremental (mulai satu atribut, tambah bertahap). Menggabung deduplication + survivorship (Figure 19-3). Dikelola oleh dimension manager (17) & fact provider (18).

---

## 4. Delivering (Subsystem 9-21)

**Subsystem 9 — SCD Manager.** Implementasi logika SCD saat atribut berubah: **Type 1** (overwrite — propagasi ke semua baris Type 2 entitas; invalidasi agregat → notify fact provider; pisahkan UPDATE dari INSERT demi performa), **Type 2** (copy baris + surrogate key baru — teknik utama; update surrogate key map; butuh CDC kuat), **Type 3** (kolom baru), plus teknik lain. Kelola housekeeping column Type 2 (effective/expiration/current indicator).

**Subsystem 10 — Surrogate Key Generator.** Hasilkan surrogate key **tak bermakna** (integer) independen per dimensi. Hindari trigger DB (bottleneck) & **jangan** konkatenasi natural key + timestamp (tak scale). Pakai sequence generator DB atau ETL tool.

**Subsystem 11 — Hierarchy Manager.** Kelola multiple hierarki dalam satu dimensi (tiap atribut single-valued). Fixed depth → atribut per level; slightly ragged → fixed; **profoundly ragged** (organisasi) → **bridge table** (organization map). Snowflake boleh di **staging** (bukan presentasi).

**Subsystem 12 — Special Dimensions Manager** (catch-all): **date/time** (dibuat di muka, tanpa sumber — spreadsheet; menantang bila multi-kalender); **junk dimension** (dari flag fakta; pra-buat atau on-the-fly); **mini-dimension** (dari input dimensi; butuh multicolumn surrogate key lookup untuk pipeline); **shrunken subset** (dari base dimension, PK dibuat independen); **small static** (lookup dibuat langsung); **user-maintained** (owner bisnis; ETL beri default `"Not Yet Assigned <key>"`).

**Subsystem 13 — Fact Table Builders.** Bangun tiga tipe: **transaction** (grain per event; bulk load baris baru; partisi per waktu; update = insert lalu delete 2 fase), **periodic snapshot** (grain per periode; hot rolling snapshot untuk saldo terkini), **accumulating snapshot** (proses awal-akhir; banyak date FK; **update destruktif**; drop-reload berkala karena variable row length).

**Subsystem 14 — Surrogate Key Pipeline.** Ganti natural key fakta dengan surrogate key dimensi **sebelum load** → jaga **referential integrity**. Update dimensi **dulu**. Pin dimensi di memori; jangan simpan natural key di fakta; tangani **key collision** (halt/suspend/perbaiki). Untuk reload history/late arriving: cari surrogate key yang berlaku saat transaksi (effective/expiration).

**Subsystem 15 — Multivalued Dimension Bridge Table Builder.** Bangun & rawat bridge (healthcare, komisi, ragged hierarchy). Pilih unique group vs reuse group; bila atribut Type 2 → bridge **time-varying**; **weighting factor** (kadang tanpa dasar rasional).

**Subsystem 16 — Late Arriving Data Handler.** **Late fact** (cari dimensi key yang berlaku saat aktivitas; sesuaikan saldo semi-aditif berikutnya). **Late dimension** (batch → tunggu; real-time → tak bisa): Type 2 update terlambat (baris baru + modif destruktif FK fakta berikutnya + reset effective date) atau fakta dengan customer belum termuat (assign surrogate key + dummy attribute, lalu Type 1 overwrite kemudian — hindari ubah key fakta).

**Subsystem 17 — Dimension Manager System.** Otoritas terpusat yang menyiapkan & **mempublikasikan conformed dimension** (harus sumber tunggal konsisten). Tugas: label agreed, tambah baris baru/Type 2 (surrogate key baru), modif Type 1/3 in place, update **version number**, replikasi serentak ke semua fact provider (version number untuk jamin drill-across konsisten).

**Subsystem 18 — Fact Provider System.** Terima conformed dimension dari dimension manager; kelola fact table. Tugas: terima/download dimensi, proses key map, tambah baris (ganti natural→surrogate), modif (koreksi, accumulating, late arriving), **hapus & recalc agregat** (bila version number berubah → recalc historis penuh), QA, bawa online, informasikan user.

**Subsystem 19 — Aggregate Builder.** Agregat = cara paling dramatis meningkatkan performa (seperti index). Populasikan & rawat aggregate fact + shrunken dimension. Incremental tercepat; perubahan besar → drop-rebuild. Selalu konsisten dengan atomik (fact provider ambil offline bila tidak). Log query lambat untuk desain agregat.

**Subsystem 20 — OLAP Cube Builder.** OLAP = **ekstensi**, bukan pesaing relational (biar relational lakukan storage). Load cube **setelah** ETL konvensional; enforce integritas hierarki dulu. **Type 2 cocok** (member baru); ⚠️ **Type 1 buruk** (overwrite → cube reprocess/corrupt/drop — "baca lagi kalimat ini").

**Subsystem 21 — Data Propagation Manager.** ETL untuk menyajikan data ke lingkungan lain (partner, pemerintah/Medicare, aplikasi analitik paket, data mining). Requirement target **tak bisa dinegosiasi**.

---

## 5. Managing (Subsystem 22-34)

Tiga kriteria: **reliability, availability (SLA), manageability**.

- **22 — Job Scheduler** — kendali metadata-driven; sadar relasi/dependensi antar-job; capture metadata progres; notifikasi ke escalation. Layanan: job definition, scheduling (time/event-based), metadata capture, logging (ke DB lebih baik), notification.
- **23 — Backup System** — disk fail, dsb.; DW simpan data lebih lama; backup intermediate staging.
- **24 — Recovery & Restart** — resume job halted atau back out & restart; butuh checkpoint andal; date/time stamp atau sequential surrogate key untuk restart logic.
- **25 — Version Control** — "snapshotting" logic & metadata ETL (check-out/in); arsipkan konteks ETL bersama data.
- **26 — Version Migration** — dev → test → production (test identik produksi); interface ke version control untuk back out.
- **27 — Workflow Monitor** — dashboard dari metadata scheduler (record diproses, error, aksi); **titik awal analisis bottleneck** performa.
- **28 — Sorting System** — sort (aggregate/join flat file); pilih resource paling efisien (dedicated sort package sering jauh lebih cepat).
- **29 — Lineage & Dependency Analyzer** — **lineage** (dari elemen → sumber asal) & **dependency** (dari sumber → tujuan); penting di lingkungan compliant.
- **30 — Problem Escalation System** — feed otomatis (error log, notifikasi operator) ke prosedur eskalasi terstruktur.
- **31 — Parallelizing/Pipelining** — manfaatkan multi-processor/grid; idealnya otomatis (partisi range atribut untuk ekstraksi).
- **32 — Security System** — **role-based** pada semua data & metadata; rekam historis akses; encrypt transfer; amankan backup media setara online.
- **33 — Compliance Manager** — **maintain chain of custody** (seperti barang bukti polisi); link ke versi terarsip & time-stamped; **compliance-enabled fact table** (tambah 5 kolom, tak overwrite → jaga chain of custody). Jangan asumsikan semua data tunduk compliance — minta pedoman chief compliance officer. Fondasi: interaksi beberapa subsystem + audit dimension + security + rerun transform.
- **34 — Metadata Repository Manager** — tangkap **process, technical, business metadata**; strategi seimbang (jangan nihil/berlebih); tunjuk **metadata manager**.

---

## Ringkasan Bab

Bab 19 memetakan **34 subsystem ETL** dalam empat komponen. **Extracting (1-3):** data profiling (go/no-go dini), CDC (audit column/timed/diff/log scraping/message queue — masing-masing trade-off), extract system (file vs stream, compress-sebelum-encrypt). **Cleaning & Conforming (4-8):** budaya kualitas data (masalah data = proses rusak), cleansing (quality screen column/structure/business rule; tag = respons terbaik), error event schema, audit dimension, deduplication (survivorship), conforming (agile inkremental). **Delivering (9-21):** SCD manager, surrogate key generator, hierarchy manager, special dimensions, fact table builders (3 tipe), **surrogate key pipeline** (RI, update dimensi dulu, key collision), bridge builder, late arriving handler, **dimension manager** & **fact provider** (conformed dimension terpusat + version number), aggregate builder, OLAP cube (Type 2 cocok/Type 1 buruk), data propagation. **Managing (22-34):** scheduler, backup, recovery/restart, version control/migration, workflow monitor, sorting, lineage/dependency, problem escalation, parallelizing, security (role-based), compliance (chain of custody), metadata repository. Bab 20 memberi proses praktis merancang & membangun sistem ETL ini.
