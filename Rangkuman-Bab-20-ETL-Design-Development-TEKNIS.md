# Bab 20 — ETL System Design & Development (Versi Mendalam & Teknis)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · **kedalaman teknis** — alur proses, struktur staging, keputusan implementasi, trade-off real-time

Fokus teknis: **10 langkah** proses ETL dengan alur konkret (detailed load schematic, hash/checksum change detection, error handling RI 4 opsi, snapshot loading per tipe), teknik percepatan (parallel structures), dan **trade-off arsitektur real-time** (tabel triage).

---

## 1. Prasyarat & 10 Langkah

**Prasyarat:** logical design selesai, high-level architecture plan, source-to-target mapping.

```
Plan:      1 High-level plan → 2 Choose ETL tool → 3 Default strategies → 4 Drill-down per table
Build:     5 Historic dimension load → 6 Historic fact load
           7 Incremental dimension → 8 Incremental fact
           9 Aggregate & OLAP loads → 10 Operation & automation
```

---

## 2. Plan (Langkah 1-4) — artefak

**Default strategies (Langkah 3):** extract per source (native extractor **bukan ODBC**), archive (≥1 bulan), data quality police, SCD (subsystem 9), availability (dokumentasi source ready), audit subsystem, staging area.

**Detailed load schematic (Langkah 4)** — contoh fact table utilitas:
```
DMR Extract (EBCDIC, 80 kolom, 13 bucket bulanan, sorted by customer)
  → Compress/encrypt/ftp
  → Unbucketize (Usage1→yyyymm, Usage2→yyyymm-1, ...; stop bila yyyymm<subscribe_date)
     [~13× baris; presorted by cust → cust_key lookup 1 pass]  → Fact_stage1 (+cust,month key)
  → Sort by meter_type, lookup meter_key   → Fact_stage2 (+meter key)
  → Sort by geog, lookup geog_key          → Fact_stage3 (+geog key)
  → Sort by date, lookup date_key          → Fact_stage4 (+read_date key)
  → Bulk-load into electric_usage_fact
```
**Semua dimensi diproses sebelum key lookup fakta** (dependensi = titik tetap job control).

**ETL specification (per tabel, 2-10 hal):** table design (kolom/tipe/key/constraint), historic volume (bulan/row count), incremental volume (new+updated/cycle), late arriving, load frequency, SCD per atribut, partisi, source overview, source-to-target detail, **profiling** (min/max/distinct/null per kolom), extract strategy, dependensi, **transformation logic (pseudocode/diagram)**, precondition, cleanup, estimasi kesulitan.

---

## 3. Historic Load (Langkah 5-6)

**Dimensi — transformasi:**
```
Data type conversion (paling umum)
Combine from separate sources
Decode production codes (kode → deskripsi)
Validate many-to-one & one-to-one:
  Snowflake di STAGING mencegah relasi rusak, mis:
    Product → Product Model → Product Subcategory → Product Category
    (FK antar level; validasi rollup sebelum denormalisasi ke dimensi flat)
Surrogate key assignment: integer++; tabel production key ↔ surrogate key di staging
Loading: bulk/fast loader (bukan stream)
```

**Type 2 history:** row effective/end date; end date versi lama = tepat sebelum efektif versi baru; histori **mundur** ke masa lalu.

**Fakta — historic:**
```
Transform: null handling, improve content (SAME GRAIN — tak boleh total tahunan campur;
           boleh derived fact)
→ Surrogate key pipeline (natural key → surrogate key)   [dimensi HARUS selesai dulu]
→ Bulk load per partisi
```

---

## 4. Incremental (Langkah 7-8)

**Langkah 7 — Dimensi (change detection):**
```
Baris baru: natural key tak ada di dimensi → INSERT (+ set effective date bila Type 2;
            hindari NULL end date → pakai tanggal jauh depan)
Deteksi perubahan: bandingkan kolom ATAU hash/checksum:
  hash_type1 = hash(concat semua atribut Type 1)
  hash_type2 = hash(concat semua atribut Type 2)
  → bandingkan hash incoming vs stored (1 kolom, cepat) daripada banyak kolom
Aturan bisnis → Type 1 (update in place) atau Type 2 (insert baris baru)
```

**Langkah 8 — Fakta (error handling RI, 4 opsi):**
| Opsi | Cara | Penilaian |
|---|---|---|
| 1 | Buang error row | OK bila missing dimension = data tak relevan |
| 2 | Tulis ke file/suspense | **Kurang baik** (kapan dipulihkan?) |
| 3 | **Dummy dimension row** + kembalikan surrogate key | **Paling menarik** (dikoreksi Type 1 saat info tersedia) |
| 4 | Map ke single unknown member | **Tidak disarankan** (semua error tertumpuk) |
- Fakta incremental **otomatis penuh** (historic bisa manual).
- Late arriving fact → surrogate key yang berlaku saat transaksi (`fact_date BETWEEN eff AND end`).

**Snapshot loading:**
```
Transaction:  insert saja (terbesar)
Periodic:     month-end; current month update harian; PARTISI BULANAN → reload cepat
Accumulating: banyak update/baris (mahal, tapi kecil)
```

---

## 5. Speed Up, Aggregate, Automation (Langkah 9-10)

**Percepatan:**
- Loading lebih sering (nightly = 1/30 monthly; preprocessing siang → ODS/staging).
- **Parallel:** multiple step independen + parallel execution DBMS. ⚠️ Parallel naif (extract semua → load dimensi → cek RI serentak) bisa **lebih lambat** (bersaing network/IO/memory) → perhitungkan resource, bukan hanya urutan logis.
- **Parallel structures:** three-way mirror/cluster (1 server load, 1 query; swap disk → downtime menit); load ke offline partition/table lalu swap.

**Langkah 9 — Aggregate/OLAP:**
- Agregat = query agregat besar disimpan sebagai tabel.
- Agregasi sepanjang date → current month update/drop-recreate.
- Atribut **Type 1** → perubahan **back out** fakta dari level agregat lama ke baru.
- ⚠️ **Wajib sinkron** dengan atomik (query ke detail = query ke agregat).

**Langkah 10 — Automation (lights-out):**
- Schedule jobs (ETL tool/Perl; eksekusi bersyarat pasca-sukses; poll DB flag/file).
- Predictable errors (early arriving fact, NULL) → desain perbaiki & lanjut (bangun dari awal).
- Unpredictable errors (garbled, power outage) → **single-column sequential surrogate key** di fakta → resume dari titik andal atau back out via range key kontigu.

---

## 6. Real-Time Trade-Offs (tabel triage)

| Kategori | Teknologi | CDC/Extract | Data quality | Hasil |
|---|---|---|---|---|
| **Daily** | Batch ETL | Wait file ready | Column+structure+business rule screen | Reconciled, final; complete transaction set |
| **Intra-day** | Micro-batch ETL | Probe query / message bus | Column+structure screen | Provisional; individual transactions; dikoreksi malam |
| **Instantaneous** | Streaming EII/ETL | Drive dari source app | Column screen saja | Provisional; transaction fragment; **terpisah dari fact table**; mungkin di-repudiate malam |

**Trade-off (tujuan quality/security/compliance TETAP):**
- **Replace batch files** → message queue/log = data mentah (salah/tak lengkap, FK belum resolve, butuh ETL batch paralel koreksi 24 jam); jangan recapitulate business rule source.
- **Limit data quality screens** → column screen + decode saja (structure/business rule mahal: butuh multi-field/record/table — mungkin tak sempat address analyzer/RI/credit check); ETL batch paralel menimpa.
- **Post facts with old dimensions** → early arriving fact pakai copy dimensi lama / generik kosong; user paham jendela sesaat dimensi tak persis cocok.
- **Eliminate staging** (EII tanpa permanent storage) → diskusi manajemen soal backup/recovery/archiving/compliance (jadi tanggung jawab source?).

**Real-time partition (requirement):**
- Semua aktivitas sejak update terakhir DW statis.
- Link mulus ke grain fakta statis (idealnya **partisi fisik by activity date**).
- **Index sangat ringan** (idealnya tanpa index — data menetes masuk).
- Responsif via **pin di memori**.

---

## Ringkasan Teknis

**10 langkah.** Plan: high-level schematic, ETL tool (self-doc/metadata/version control/parallel), default strategies (native extractor, archive ≥1 bulan), drill-down (detailed load schematic: unbucketize→sort→lookup key bertahap→bulk load; dimensi sebelum fakta) + ETL spec (profiling min/max/distinct/null). **Historic:** dimensi (type conversion, decode, validate rollup via snowflake staging, surrogate key map, bulk loader; Type 2 histori mundur), fakta (same grain, surrogate key pipeline pasca-dimensi). **Incremental:** change detection **hash type1/type2**, fakta otomatis, **error RI 4 opsi (dummy row terbaik)**, snapshot (periodic partisi bulanan). **Speed up:** parallel (perhatikan resource, mirror/cluster swap), aggregate (Type 1 back out, sinkron atomik), automation lights-out (**sequential surrogate key** untuk restart/back-out). **Real-time:** triage daily/intra-day/instantaneous (tabel: teknologi/CDC/screen/hasil); trade-off (replace batch, limit screen column-only, post facts w/ old dimensions, eliminate staging); **real-time partition** (index ringan, pin memori).
