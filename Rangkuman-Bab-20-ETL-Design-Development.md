# Bab 20 — ETL System Design & Development Process & Tasks

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab penutup buku — pasangan **praktis** dari Bab 19: bagaimana **merencanakan, mendesain, dan membangun** sistem ETL (menerapkan 34 subsystem). Mengikuti 10 langkah, dari high-level plan → spesifikasi → **one-time historic load** (dimensi & fakta) → **incremental processing** → agregat/OLAP → operasi & otomasi, ditutup dengan **implikasi real-time** (triage instantaneous/intra-day/daily, trade-off arsitektur, real-time partition).

---

## 1. ETL Process Overview & Prasyarat

Sebelum mendesain ETL: selesaikan **logical design**, draf **high-level architecture plan**, dan draf **source-to-target mapping** semua elemen. Kumpulkan info relevan (beban pemrosesan yang boleh dibebankan ke source), uji alternatif (transformasi di source/target/platform sendiri?).

---

## 2. Develop the ETL Plan (Langkah 1-4)

**Langkah 1 — High-level plan.** Skematik sederhana **sumber & target** (contoh: perusahaan utilitas dari sistem COBOL 30 tahun). Sorot pertanyaan & isu belum terselesaikan; update & rilis sering. Bisa dua versi: sederhana (komunikasi eksternal) & detail (internal).

**Langkah 2 — Pilih ETL tool.** Meski hand-coding tampak lebih cepat di fase awal, **ETL tool = best practice industri**: self-documentation grafis (vs hand-code = "kekacauan tak tertembus"), fondasi metadata, version control, logika transformasi lanjut (fuzzy match, dedup nama/alamat), performa lebih baik di keahlian lebih rendah, parallelizing & fail-over otomatis. Jangan harap balik modal di fase pertama (kurva belajar curam); keuntungan besar di fase & modifikasi berikutnya.

**Langkah 3 — Default strategies** untuk aktivitas umum: extract per source (push/stream/log; pakai native extractor bukan ODBC), archive extracted/staged data (≥1 bulan), police data quality (monitor selama ETL, jangan tunggu user), manage dimension changes (SCD subsystem 9), system availability (dokumentasikan kapan tiap source tersedia), data auditing subsystem, organize staging area.

**Langkah 4 — Drill down per target table.** Profiling lebih dalam; finalisasi source-to-target. **Pastikan hierarki bersih** — hierarki paling andal dikelola di source ternormalisasi (FK antar level); bila source Excel di desktop user → bersihkan atau akui bukan hierarki.

**Detailed table schematics** — diagram alur per tabel (contoh: unbucketize 13 bucket bulanan → baris per bulan; sort → lookup key bertahap → bulk load). Semua **dimensi diproses sebelum** key lookup fakta.

**ETL specification document** — gabungkan source-to-target mapping, profiling report, physical design + keputusan bab ini. Per tabel (2-10 halaman): table design, historic & incremental volume, late arriving handling, load frequency, SCD per atribut, partisi, source overview, source-to-target detail, profiling (min/max/distinct/null), extract strategy, dependensi, **transformation logic (pseudocode/diagram)**, precondition, cleanup, estimasi kesulitan. (Realistis: one-pager flow + data model + source-to-target + 5 halaman deskripsi sudah lebih baik dari kebanyakan tim.)

**Sandbox source system** — snapshot statis source untuk eksplorasi tanpa "killer query"; salin penuh (subset hanya bila volume ekstrem); bisa jadi bahan training.

---

## 3. One-Time Historic Load (Langkah 5-6)

Sering ETL historic & incremental **dibangun terpisah** (banyak kesamaan, bisa reuse).

### 3.1 Dimensi

**Type 1 dimension** — termudah: extract, transformasi, load. Transformasi: **data type conversion** (paling umum), combine from separate sources, **decode production codes** (kode → deskripsi), **validate many-to-one & one-to-one** (snowflake berguna di **staging** untuk mencegah relasi rusak). **Surrogate key assignment** — integer naik satu; maintain tabel production key ↔ surrogate key di staging. **Loading** — pakai **bulk/fast loader** (bukan stream).

**Type 2 dimension history** — user ingin histori **mundur ke masa lalu**, bukan hanya sejak sekarang. Set row effective/end date saat data mengalir; end date versi lama = tepat sebelum efektif versi baru.

**Date & static dimension** — dibangun manual (spreadsheet), rentang mencakup data fakta historis.

### 3.2 Fakta

Historic fact load **berbeda signifikan** dari incremental. **Transformasi:** null handling, **improve fact table content** (semua fakta harus **same grain** — tak boleh ada total tahunan bercampur; boleh derived fact). Lalu masuk **surrogate key pipeline** (tukar natural key → surrogate key). **Dimensi harus selesai diproses dulu** sebelum fakta masuk pipeline. Bulk load per partisi.

---

## 4. Incremental ETL Processing (Langkah 7-8)

Tantangan terbesar: **identifikasi baris baru/berubah/dihapus**. Historic = mayoritas insert; incremental = insert + **update/delete** (mahal). **Lebih efisien memuat hanya record yang berubah.**

**Langkah 7 — Dimensi.** Mirip historic. Master file bisa jadi sumber. Bila source tak bisa identifikasi perubahan → tarik snapshot penuh & bandingkan. Cari baris baru (natural key tak ada di dimensi → insert). Deteksi perubahan: bandingkan kolom, atau pakai **hash/checksum** (hash type-1 attributes & type-2 attributes terpisah → bandingkan hash lebih cepat dari banyak kolom). Aturan bisnis tentukan Type 1 vs Type 2.

**Langkah 8 — Fakta.** Tak ada load window tunggal; **incremental & otomatis penuh** (historic bisa manual). Extract + data quality checkpoint. **Surrogate key pipeline** incremental mirip historic, beda di **error handling RI**: (1) buang error row (bila missing dimension = data tak relevan), (2) tulis ke file/suspense (kurang baik), (3) **buat dummy dimension row** & kembalikan surrogate key (**paling menarik** — dikoreksi Type 1 saat info tersedia), (4) map ke single unknown member (**tidak disarankan** — semua error tertumpuk). **Late arriving fact** → cari surrogate key yang berlaku saat transaksi.

**Load snapshot fact tables** — transaction (insert saja), periodic (month-end; current month di-update harian; **partisi bulanan** memudahkan reload), accumulating (banyak update per baris; mahal tapi kecil).

---

## 5. Speed Up, Aggregate, Automation (Langkah 9-10)

**Speed up load cycle:** (1) proses hanya data berubah, (2) **loading lebih sering** (nightly = 1/30 volume monthly; bisa preprocessing siang hari ke ODS/staging), (3) **parallel processing** (multiple step independen + parallel execution DBMS; ⚠️ parallel buruk bila step bersaing resource — perhatikan resource, bukan hanya urutan logis), (4) **parallel structures** (mirror/cluster: satu server load, satu query; swap disk → downtime menit; load ke offline partition lalu swap).

**Langkah 9 — Aggregate & OLAP loads.** Aggregate = hasil query agregat besar disimpan. Kompleks bila agregasi sepanjang date (current month di-update/drop-recreate) atau atribut Type 1 (perubahan Type 1 → back out fakta dari level lama ke baru). ⚠️ **Wajib sinkron** dengan atomik (hasil query ke detail vs agregat harus sama).

**Langkah 10 — Operation & automation.** Ideal: **lights-out** tanpa intervensi. **Schedule jobs** (ETL tool/script Perl; eksekusi bersyarat setelah sukses). **Handle predictable errors** (early arriving fact, NULL → desain untuk perbaiki & lanjut; bangun dari awal). **Handle unpredictable errors** (data garbled, power outage) → **single-column surrogate key** di fakta memungkinkan resume dari titik andal atau back out via range key kontigu.

---

## 6. Real-Time Implications

**Real-time triage** — jangan tanya user "mau real-time?" (jawaban "boleh!" tak berguna). Bagi tiga kategori:

- **Instantaneous** — layar = kondisi sesungguhnya source tiap saat (mis. inventory status). Implementasi **EII** (source layani query; **tanpa caching** di pipeline; batasi kompleksitas query).
- **Intra-day** — di-update berkali-kali/hari, tak dijamin absolut kini (mis. quote saham 15 menit). **Micro-batch** ETL konvensional (full CDC/extract/clean/conform/surrogate); beda dari daily di **CDC & extract** (tap message queue/transaction log/trigger bandwidth tinggi).
- **Daily** — valid per batch/rekonsiliasi akhir hari kerja. Paling sederhana & sering paling andal (source sudah koreksi data). Jelaskan kompromi bila user menuntut lebih cepat.

**Trade-off arsitektur real-time** (tujuan quality/integration/security/compliance **tetap**):
- **Replace batch files** dengan message queue/log → data mentah (mungkin salah/tak lengkap, FK belum resolve, butuh ETL batch paralel koreksi tiap 24 jam). Jangan recapitulate business rule source.
- **Limit data quality screens** ke column screen + decode sederhana (structure & business rule screen mahal → mungkin dihilangkan; beri tahu user data provisional; ETL batch paralel menimpa).
- **Post facts with dimensions** — fakta tiba sebelum dimensi (early arriving); pakai copy dimensi lama atau versi generik kosong; user paham ada jendela sesaat dimensi tak persis cocok.
- **Eliminate data staging** (EII stream tanpa permanent storage) → diskusi serius dengan manajemen soal backup/recovery/archiving/compliance.

**Real-time partition** — ekstensi partisi dari DW statis untuk menutup gap hingga saat kini. Idealnya: memuat semua aktivitas sejak update terakhir, link mulus ke grain fakta statis (partisi fisik by activity date), **di-index sangat ringan** (idealnya tanpa index — data "menetes" masuk), responsif via **pin di memori**.

---

## Ringkasan Bab

Bab 20 menutup buku dengan proses **praktis membangun ETL**. **Plan (1-4):** high-level schematic, pilih ETL tool (best practice: self-doc/metadata/version control/performa), default strategies, drill-down per tabel (hierarki bersih) → **ETL specification document** & sandbox. **Historic load (5-6):** dimensi (Type 1 termudah; surrogate key assignment; Type 2 histori mundur; date manual) & fakta (same grain, surrogate key pipeline setelah dimensi selesai). **Incremental (7-8):** identifikasi perubahan (hash/checksum), fakta otomatis penuh, error RI (dummy row paling menarik), snapshot loading. **Speed up (9-10):** loading lebih sering, parallel (perhatikan resource), aggregate/OLAP (sinkron dengan atomik), automation lights-out (surrogate key untuk restart). **Real-time:** triage instantaneous(EII)/intra-day(micro-batch)/daily(batch); trade-off (replace batch, limit screen, post facts with old dimensions, eliminate staging); **real-time partition** (ringan-index, pin memori). Ini menutup *The Data Warehouse Toolkit* — dari pemodelan dimensional hingga implementasi ETL end-to-end.
