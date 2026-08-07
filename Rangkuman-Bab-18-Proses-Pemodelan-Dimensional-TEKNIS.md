# Bab 18 — Dimensional Modeling Process & Tasks (Versi Mendalam & Teknis)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · **kedalaman teknis** — artefak, template, checklist, keputusan yang dapat ditindaklanjuti

Bab ini adalah proses (bukan skema), jadi versi teknis menekankan **artefak konkret** (struktur design worksheet, isi issues log, format bus matrix detail), **checklist per tahap**, dan **keputusan implementasi** yang mengubah desain logis menjadi source-to-target mapping siap-ETL.

---

## 1. Alur Proses & Deliverable

```
Preparation
   → High Level Dimensional Model (bubble chart)
       → Detailed Dimensional Model Development ⇄ Iterate and Test
           → Model Review and Validation
               → Final Design Documentation
```

| Fase | Input | Deliverable |
|---|---|---|
| Persiapan | preliminary bus matrix, detailed requirements | tim, tools, naming convention, jadwal |
| High-level | bus matrix | bubble chart (grain + dimensi) |
| Detail | bubble chart, source profiling | design worksheet per tabel, issues log |
| Review | worksheet | model tervalidasi (IT/user) |
| Final | working papers | dokumen desain final |

Durasi: **3-4 minggu** per proses bisnis; **conformed dimension yang sudah ada** memangkas waktu (fokus tinggal fakta).

---

## 2. Checklist Persiapan

**Peserta:**
- Data modeler (fasilitator, pemilik deliverable).
- **SME bisnis** (wajib — historis tahu ekstraksi data).
- Source system expert.
- Perwakilan DBA + ETL (belajar insight; **tahan** 3NF & penundaan ke BI).
- Data steward (governance conformed dimension).
- Prinsip: **tukar kompleksitas ETL ↔ kesederhanaan + prediktabilitas presentasi BI**.

**Tools:**
- **Spreadsheet** untuk dokumentasi awal (iterasi cepat) → modeling tool saat mengeras (forward-engineer: tabel/index/partisi/view).
- **Data profiling** (SQL atau tool ETL) — konten & relasi source nyata; verifikasi usable / cacat terkelola.

**Naming convention** — 3 bagian: **prime word · qualifier · class word**. Deskriptif & konsisten bisnis (nama = antarmuka BI).

**Logistik** — sesi 2-3 jam pagi/sore, 3-4 hari/minggu; ruang dedikasi + whiteboard + proyektor.

---

## 3. Empat Keputusan & Bubble Chart

**Empat langkah (Bab 3):** business process → grain → dimensi → fakta.

**Bubble chart high-level** (contoh Orders, grain = 1 baris per order line):
```
Due Date ─┐   ┌─ Bill To Customer
Order Date├─┐ ├─ Sold To Customer
Currency  │ │ ├─ Ship To Customer
Promotion │ ├─┤ ORDERS ├─ Sales Person
          │ │ │ (grain = 1 row per order line)
Order Profile┘ ├─ Channel
               └─ Product
```
**Keputusan grain (teknis):**
- Berakar pada **realita data fisik** yang tersedia.
- Satu baris bus matrix → **≥1 bubble chart** (tiap fact table = granularitas unik).
- Dimensi muncul alami dari grain. Mismatch dimensi-grain → **(a)** hilangkan dimensi, **(b)** ubah grain, atau **(c)** solusi multivalued (bridge).

---

## 4. Design Worksheet — struktur artefak

Contoh worksheet dimensi (DimOrderProfile):
```
Table Type   : Dimension
Display Name : OrderProfile
Used in schemas: Orders
Size         : (estimasi baris)

Kolom worksheet per atribut:
  Target Column Name | Datatype | Datatype Size | Example Values
  Description | SCD Type
  Source System | Source Table | Source Field | Source Datatype
  ETL Rules

Contoh baris:
  OrderProfileKey | surrogate PK | ETL Rules: Surrogate key
  OrderMethod     | "Method used to place order" | e.g. Internet
                  | Source: OrderHeader
  OrderSource     | "Source of the order" | e.g. Sales | Source: OrderHeader
  CommissionInd   | "Indicates whether order is commission" | Derived
                  | 0=Non-Commission, 1=Commission | Source: OrderHeader
```

**Minimum per worksheet:**
- **Dimensi:** attribute name, description, sample values, **SCD type indicator** tiap atribut.
- **Fakta:** tiap **FK relationship**, **degenerate dimension**, dan aturan **additive / semi-additive / non-additive** tiap fakta.

→ Worksheet = **langkah pertama source-to-target mapping**; physical design melengkapi nama fisik, datatype, key declaration.

**Urutan pengerjaan detail:**
1. Dimensi dulu (mulai **date dimension** — sukses awal).
2. Identifikasi atribut + **conformed dimension** (tugas **bisnis** menyepakati nama/definisi; governance committee bila buntu; timbang **junk/mini-dimension**).
3. Identifikasi fakta (true to grain; base + **derived facts**).
4. **Circle back** ke dimensi → tetapkan **SCD technique** per atribut (Type 0-7; tanya source expert: perubahan = koreksi?).

---

## 5. Issues Log & Bus Matrix Detail

**Issues log** — tangkap: isu, definisi, transformation rule, tantangan data quality. Penanggung jawab (sering PM); review di akhir tiap sesi; antar-sesi: profiling, klarifikasi definisi, temui source expert.

**Bus matrix ter-update** — penemuan → fact/dimension baru, split/merge dimensi. **Detailed implementation bus matrix** (Bab 16) menambah kolom **granularity** & **facts/metrics** per baris fact table:
```
Fact Table | Granularity | Facts | [kolom dimensi: X menandai partisipasi]
```

---

## 6. Review Berlapis — taktik

**IT review (pertama):**
- ⚠️ Peserta sering **3NF modeler** → cenderung terapkan aturan OLTP.
- **Taktik:** edukasi dimensional **proaktif** dulu (jangan berdebat disiplin).
- Urutan: bus matrix (scope/conformed/prioritas) → high-level diagram → worksheet detail → open issues.
- Tunjuk pencatat issue & rekomendasi.

**Core user review:**
- Sering **skip** (core user = anggota tim). Bila perlu, mirip IT review (user lebih teknis). Organisasi kecil: **gabung** IT + core user.

**Broader business user review:**
- **Edukasi = review**. Jangan membanjiri; tunjukkan dukungan ke requirement.
- Urutan: bus matrix (roadmap) → bubble chart → dimensi kritis (customer/product) + **drill path hierarki**:
```
Department# ─► Category ─► Brand ─► Package Type ─► SKU/Product
(relasi many-to-one bertingkat, ditunjukkan sebagai jalur drill)
```
- Walk-through contoh pertanyaan dari requirement document.

**Finalisasi dokumentasi:** deskripsi proyek + high-level diagram + design worksheet tiap tabel + open issues.

---

## Ringkasan Teknis

Proses iteratif (3-4 minggu; conformed dimension memangkas waktu). **Persiapan:** tim (SME bisnis + DBA/ETL penahan-3NF + steward), tools (spreadsheet→modeling tool, **data profiling**), naming (prime-qualifier-class), sesi 2-3 jam. **Desain:** 4 langkah; **bubble chart** (grain berakar realita fisik; 1 baris matrix → ≥1 chart; mismatch → hilangkan/ubah-grain/multivalued); **design worksheet** (atribut+SCD type; fakta+FK+degenerate+additivity → source-to-target awal); urutan dimensi-dulu (date favorit), conformed = tugas bisnis, SCD per atribut. **Issues log** + **detailed bus matrix** (granularity+facts). **Review berlapis:** IT (edukasi dimensional dulu, lawan bias 3NF), core user (sering skip/gabung), broader business (edukasi + drill path). Finalisasi dokumentasi.
