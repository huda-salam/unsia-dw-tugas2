# Bab 19 — ETL Subsystems & Techniques (Versi Mendalam & Teknis)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · **kedalaman teknis** — struktur skema, alur proses, keputusan implementasi, trade-off per subsystem

Versi ini menekankan **detail implementasi** tiap subsystem: struktur tabel (error event, audit, compliance-enabled), alur pemrosesan (surrogate key pipeline, SCD, late arriving), dan analisis trade-off metode (CDC, sorting, parallelizing).

---

## 1. Empat Komponen & 34 Subsystem (peta)

| Komponen | Subsystem | Fungsi |
|---|---|---|
| **Extracting** | 1-3 | profiling, CDC, extract |
| **Cleaning & Conforming** | 4-8 | cleansing, error event, audit, dedup, conform |
| **Delivering** | 9-21 | SCD, key gen, hierarchy, special dim, fact builder, key pipeline, bridge, late arriving, dim manager, fact provider, aggregate, OLAP, propagation |
| **Managing** | 22-34 | scheduler, backup, recovery, version control/migration, workflow, sort, lineage, escalation, parallel, security, compliance, metadata |

**10 requirement area** (checklist pra-desain): business needs, compliance, data quality, security, data integration, **data latency** (batch→microbatch→streaming = paradigm shift), archiving/lineage, BI delivery interfaces, available skills, legacy licenses.

---

## 2. Extracting (1-3) — teknik CDC

**CDC — perbandingan metode:**
| Metode | Cara | Kelemahan |
|---|---|---|
| Audit columns | date/time add/modify (trigger) | Bila diisi aplikasi bukan trigger → risiko; delete tak tertangkap; NULL → butuh alternatif |
| Timed extract | `WHERE modified = SYSDATE-1` | ⚠️ **Tidak andal**: duplikat saat restart; skip hari → data hilang permanen |
| Full diff compare | snapshot kemarin vs hari ini record-by-record | Menyeluruh tapi mahal; lakukan di **source**; pakai **CRC checksum** |
| Log scraping/sniffing | baca redo log | Paling berantakan; log penuh → DBA kosongkan → transaksi hilang |
| Message queue monitoring | monitor queue | Overhead rendah tapi **tanpa replay** → koneksi putus = data hilang |

**Extract — file vs stream:**
```
File:   Extract→file | move ke ETL server | transform | load staging   (4 langkah)
        + mudah restart, encrypt/compress, verifikasi row count
Stream: source → transform engine → staging (1 proses)
        + lebih menarik, tapi sulit restart
```
- ⚠️ XML payload <10% file → boros untuk transfer besar/sering.
- **Compress SEBELUM encrypt** (terenkripsi tak compress baik); compress hemat 30-50%.

---

## 3. Cleaning & Conforming (4-8) — struktur skema

**Quality screens (3 tipe, Jack Olson):**
```
Column screen    : single column (null / range / format)
Structure screen : antar-kolom (hierarki many-to-one / FK-PK / alamat)
Business rule    : kompleks / aggregate threshold (mis. jumlah MRI abnormal utk diagnosis ringan)
```
Respons error: (1) halt (butuh intervensi manual), (2) suspense file (integritas DB dipertanyakan sampai record dipulihkan), (3) **tag data** (terbaik — via audit dimension).

**Subsystem 5 — Error Event Schema:**
```
Error Event Fact                 grain = 1 error dari quality screen
  Error Event Key (PK surrogate)
  Error Event Date Key (FK)      ──► Date Dimension
  Screen Key (FK)                ──► Screen Dimension (type, ETL module,
  Batch Key (FK)                      processing definition, exception action)
  Error Date/Time, Severity Score

Error Event Detail Fact          grain = 1 field per record yang error
  Error Event Key (FK)           ← tie ke fact di atas
  Table Key, Field Key, Record Identifier Key (FK)
  Error Condition
```

**Subsystem 6 — Audit Dimension:**
```
Audit Dimension
  Audit Key (PK)
  Overall Quality Rating, Complete Flag, Validation Flag,
  Out Of Bounds Flag, Screen Failed Flag, Record Modified Flag,
  ETL Master Version Number, Allocation Version Number
```
Run sempurna → 1 baris audit (dilekatkan ke semua baris fakta hari itu); ada out-of-bounds → +1 baris.

**Subsystem 7 — Deduplication:** match (key identik atau **fuzzy** nama/alamat) → **survivorship** (kolom kualitas tertinggi per aturan prioritas → 1 baris conform + back-reference natural key semua sumber).

**Subsystem 8 — Conforming (agile):** 2 dimensi conform bila berbagi ≥1 atribut (nama & konten sama); tanam inkremental (`Customer Category` → geografi → dst.). Flow: extract → clean/dedup lokal → conform/survive → merge/dedup global → replication engine. Special contents: dimension version number + back pointer natural key.

---

## 4. Delivering (9-21) — alur inti

**Subsystem 9 — SCD Manager (alur surrogate key):**
```
Source Extract → CRC Compare (vs Master Dimension Cross Ref)
  CRC match → ignore
  CRC beda → find changed field(s):
    type 1/3 → UPDATE dimension attribute in place
    type 2   → INSERT baris baru (surrogate key baru) + assign dates/indicator
             + UPDATE prior most-recent row + update Most Recent Surrogate Key Map
```
- Type 1: overwrite **semua** baris Type 2 entitas; **pisahkan UPDATE dari INSERT** (UPDATE else INSERT = performance killer); invalidasi agregat → notify fact provider.

**Subsystem 13 — Fact Table Builders:**
| Tipe | Grain | Loading |
|---|---|---|
| Transaction | per event | bulk insert; partisi per waktu; update = insert lalu delete (2 fase, pakai sequential surrogate key) |
| Periodic snapshot | per periode | insert/update sama; **hot rolling** (baris current di-update harian) |
| Accumulating snapshot | proses awal-akhir | banyak date FK; **update destruktif**; drop-reload berkala (variable row length) |

**Subsystem 14 — Surrogate Key Pipeline:**
```
Fact (natural key ID) → [Replace tiap ID dengan surrogate key dari dimensi]
  → Referential Integrity Failures (feed balik ke ETL)
  → Key Collisions (halt/suspend/perbaiki + tulis error event)
  → Fact (surrogate key) → load DBMS
```
- Update **dimensi dulu** (jadi sumber sah PK).
- Pin dimensi di memori (RAM besar); jangan simpan natural key di fakta; jangan tulis ke disk sampai semua baris lolos.
- Reload history/late arriving → cari surrogate key di mana `fact_date BETWEEN effective_start AND end_date`.

**Subsystem 16 — Late Arriving Handler:**
```
Late FACT:      cari dimensi key yang berlaku SAAT aktivitas;
                sesuaikan saldo semi-aditif baris berikutnya;
                interface ke compliance (mengubah history)
Late DIM (Type 2 terlambat): INSERT baris (surrogate baru)
                + modif destruktif FK fakta berikutnya + reset effective date
                + scan forward untuk Type 2 subsequent
Late DIM (customer belum termuat): assign surrogate key + DUMMY attribute
                → kembali Type 1 overwrite saat info lengkap (hindari ubah key fakta)
```

**Subsystem 17 & 18 — Dimension Manager / Fact Provider:**
```
Dimension Manager (otoritas terpusat, sumber tunggal):
  - label agreed; INSERT baris baru/Type 2 (surrogate baru); modif Type 1/3 in place
  - UPDATE version number (bila Type 1/3); replikasi SERENTAK ke semua fact provider
Fact Provider (per fact table):
  - terima/download dimensi; proses key map (new+current, new+postdated)
  - INSERT fakta (natural→surrogate); modif (koreksi/accumulating/late)
  - hapus agregat invalid; recalc (version number berubah → recalc HISTORIS penuh)
  - QA; online; informasikan user (termasuk dimension version change)
```
Version number di **tiap baris** dimensi → jamin drill-across pakai rilis sama.

**Subsystem 20 — OLAP Cube:** load **setelah** ETL relational; enforce hierarki dulu. **Type 2 cocok** (member baru); ⚠️ **Type 1 buruk** (overwrite → cube reprocess/corrupt/drop).

---

## 5. Managing (22-34) — highlight teknis

**Subsystem 22 — Job Scheduler (layanan):** job definition (dependensi — mis. gagal load customer → berisiko load sales fact customer baru), scheduling (time/event: monitor DB flag, cek file, bandingkan tanggal), metadata capture, logging (**ke DB** > text file, untuk time series analisis), notification (ke escalation).

**Subsystem 24 — Recovery & Restart:** checkpoint andal; date/time stamp atau **sequential fact table surrogate key** → restart logic (resume atau back out semua baris load).

**Subsystem 27 — Workflow Monitor:** dashboard dari metadata scheduler; **titik awal analisis bottleneck** (sort di RDBMS, filtering/CDC terlambat di pipeline, peluang parallel/pipeline tak terpakai).

**Subsystem 29 — Lineage & Dependency:** lineage (elemen → sumber asal, untuk compliance) vs dependency (sumber → semua tujuan).

**Subsystem 32 — Security:** **role-based** semua data & metadata; rekam historis akses per individu/role; encrypt transfer; backup media aman setara online.

**Subsystem 33 — Compliance Manager — chain of custody:**
```
Compliance-Enabled Fact Table (Figure 19-12)
  [kolom asli] + 5 kolom tambahan:
  Begin/End effective date, change reference, source reference, ...
  → JANGAN overwrite (overwrite = history hilang = chain of custody putus)
```
- Tabel compliance-enabled bisa di **background** (tak perlu di-index) sementara tabel operasional normal tetap dipakai.
- Fondasi: rerun transform vs original data + archived/time-stamped restore + security log + **audit dimension** (runtime metadata → baris fakta).
- ⚠️ Jangan asumsikan semua data tunduk compliance — minta pedoman chief compliance officer.

**Subsystem 34 — Metadata Repository:** tangkap **process + technical + business metadata**; strategi seimbang; tunjuk **metadata manager**.

---

## Ringkasan Teknis

34 subsystem dalam 4 komponen (+ 10 requirement area, latency = paradigm shift). **Extracting:** CDC (tabel trade-off: audit column/timed **tidak andal**/diff+CRC/log scraping/message queue), extract file-vs-stream (compress **sebelum** encrypt). **Cleaning:** quality screen 3 tipe (column/structure/business rule), respons **tag** terbaik; **error event schema** (fact + detail fact); **audit dimension**; deduplication+survivorship; conforming agile inkremental. **Delivering:** SCD manager (alur CRC→type 1/3 update / type 2 insert+key map; pisahkan UPDATE/INSERT); fact builders 3 tipe; **surrogate key pipeline** (update dimensi dulu, pin memori, RI failure & key collision, late arriving pakai effective-date lookup); late arriving handler (fact vs dim, dummy attribute); **dimension manager/fact provider** (conformed terpusat, version number, recalc agregat historis); OLAP (Type 2 cocok/Type 1 buruk). **Managing:** scheduler (logging ke DB), recovery (checkpoint/sequential key), workflow monitor (bottleneck), lineage/dependency, security role-based, **compliance** (compliance-enabled fact table +5 kolom, chain of custody, jangan overwrite), metadata repository (process/technical/business).
