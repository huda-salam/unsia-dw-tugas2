# Bab 14 — Healthcare (Versi Mendalam & Teknis)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · **kedalaman teknis** — struktur skema, SQL, implikasi ETL, analisis trade-off

Fokus teknis: **accumulating snapshot** dengan update destruktif & partisi, **diagnosis group bridge** (dua varian + ETL lookup + twin date), **measure type dimension** (trade-off baris & kalkulasi lintas baris), dan penanganan text/image/inventory.

---

## 1. Accumulating Snapshot — Claims Billing & Payment Workflow

```
Claims Billing and Payment Workflow Fact     grain = 1 baris / line pada tagihan medis
─────────────────────────────────────────    (histori terakumulasi, DI-UPDATE)
  -- 8 role-playing date (view terpisah):
  Treatment Date Key (FK)
  Primary Insurance Billing Date Key (FK)
  Secondary Insurance Billing Date Key (FK)
  Responsible Party Billing Date Key (FK)
  Last Primary Insurance Payment Date Key (FK)
  Last Secondary Insurance Payment Date Key (FK)
  Last Responsible Party Payment Date Key (FK)
  Zero Balance Date Key (FK)
  -- dimensi lain:
  Patient, Physician, Physician Organization, Procedure, Facility,
  Primary Diagnosis, Primary/Secondary Insurance Organization (2 role),
  Responsible Party, Employer Key (FK)
  Master Bill ID (DD)
  -- fakta:
  Billed Amount, Primary/Secondary Insurance Paid, Responsible Party Paid,
  Total Paid (calc), Sent to Collections, Written Off, Unpaid Balance (calc),
  Length of Stay,
  Bill→Initial {Primary/Secondary/Responsible Party} Payment Lag,
  Bill→Zero Balance Lag
```

**Pemilihan grain (3 opsi Bab 4):** transaction (tiap transaksi billing/payment), periodic snapshot (time series panjang — tak cocok proses pendek), **accumulating snapshot** (dipilih — membingkai tiap klaim ke framework standar).

**Mekanika update:**
- Baris dibuat saat charge diterima; 7 tanggal terakhir awalnya menunjuk baris date **"To Be Determined"** (surrogate key **tak boleh NULL**).
- Tiap event (pembayaran, tagihan ke secondary/responsible) → **baris sama di-update destruktif** (key & fakta ditimpa).
- ⚠️ **Implikasi DBA:** update destruktif → fragmentasi. Bila baris menstabil dalam timeframe → **reorganisasi fisik** memulihkan disk & performa. **Partisi pada `treatment_date_key`** terjaga karena treatment date **tidak** direvisi.
- Skenario non-standar (multi-payment per line, satu payment untuk multi-klaim) → **companion transaction schema**.

**Role-playing date (8 view):**
```sql
CREATE VIEW treatment_date AS SELECT date_key, date AS treatment_date, ... FROM date_dim;
CREATE VIEW zero_balance_date AS SELECT date_key, date AS zero_balance_date, ... FROM date_dim;
-- fact join terpisah ke tiap view seolah 8 tabel date independen
```

---

## 2. Multivalued Diagnosis — Bridge Table

**Mengapa BUKAN FK jamak:** diagnosis bukan role independen; sering >3 (lansia rawat inap bisa 20); FK jamak → BI tak tahu slot mana di-constrain.

**Desain A (many-to-many langsung):**
```
Claim Billing Line Item Fact       Diagnosis Group Bridge      Diagnosis Dimension
  ... Diagnosis Group Key (FK) ──►   Diagnosis Group Key (FK)    Diagnosis Key (PK)
      Master Bill ID (DD)            Diagnosis Key (FK) ────────► Diagnosis Code (NK)
                                                                  Diagnosis Section/Category
```
Pasien 3 diagnosis → 1 group + 3 baris bridge.

**Impact query (tanpa weighting → bisa overcount):**
```sql
SELECT SUM(f.billed_amount)
FROM claim_fact f
JOIN diagnosis_group_bridge b ON f.diagnosis_group_key = b.diagnosis_group_key
JOIN diagnosis d ON b.diagnosis_key = d.diagnosis_key
WHERE d.diagnosis_description = 'Congestive Heart Failure';
-- metrik dikaitkan ke SEMUA diagnosis dalam grup → over counting dipahami analis
```
> Weighting factor **elegan tapi tak wajib**. Untuk diagnosis, bobot dampak nyaris mustahil → tinggalkan. Dengan >1 dimensi multivalued, jangan pusingkan interaksi bobot.

**Desain B (jika tool menuntut FK→PK proper):** sisipkan **Diagnosis Group Dimension** (PK = group key) antara fakta & bridge:
```
Fact ──► Diagnosis Group Dimension (PK) ◄── Diagnosis Group Bridge ──► Diagnosis Dim
         (biasanya tanpa info baru,
          kecuali label cluster mis. "Kimball Syndrome")
```
→ semua join jadi many-to-one di segala arah.

**ETL portofolio grup:** jangan grup unik per encounter (baris astronomis, banyak identik). Saat ETL, *lookup* set diagnosis di **master diagnosis group**: temukan → pakai; tidak → buat baru. (Detail bridge di Bab 19.)

**Inpatient evolving (twin date):**
```
Diagnosis Group Bridge
  Diagnosis Group Key (FK), Diagnosis Key (FK)
  Begin Effective Date, End Effective Date   -- change tracking (Bab 7)
```

---

## 3. Supertype/Subtype untuk Charges

```
Inpatient Hospital Claim Billing & Payment Workflow Fact
  ... (8 role-playing date sama)
  Patient Key (FK)
  Admitting Physician Key (FK),  Admitting Physician Organization Key (FK)
  Attending Physician Key (FK),  Attending Physician Organization Key (FK)
  Procedure Key (FK), Facility Key (FK)
  Admitting Diagnosis Group Key (FK)   -- ditentukan di awal, sama utk semua baris 1 rawat inap
  Discharge Diagnosis Group Key (FK)   -- baru diketahui saat pulang
  Primary/Secondary Insurance Org Key, Responsible Party Key, Employer Key
  Master Bill ID (DD), Facts...
```
- Facility charge (inpatient) vs professional charge (outpatient) = pola supertype/subtype (Bab 10).
- **Physician role ganda** (admitting/attending, tiap organisasi). Tim bedah kompleks → primary responsible physician (FK) + sisanya via **bridge multivalued**.
- **Dua diagnosis group multivalued** per baris (admitting & discharge).

---

## 4. Measure Type Dimension — fakta sparse

```
Lab Test Result Facts                     Lab Test Measurement Type Dimension
  Order Date Key, Test Date Key,            Lab Test Measurement Type Key (PK)
  Patient Key, Physician Key, Lab Test Key, Description
  Lab Test Measurement Type Key (FK) ─────► Unit of Measure
  Observed Test Result Value                (+ additivity restriction)
```
Grain = **1 baris per pengukuran per event** (bukan per event).

| Aspek | Konsekuensi |
|---|---|
| Fleksibilitas | Tambah tipe = tambah **baris** dimensi, tak ubah struktur fakta |
| Null | Hilang (baris ada hanya bila pengukuran ada) |
| Jumlah baris | **Meledak** (10 pengukuran = 10 baris) — hanya cocok **sangat sparse** (lab, uji manufaktur) |
| Densitas naik | Terlalu banyak baris → **kembali** ke desain kolom tetap |
| Kalkulasi | Sulit antar-pengukuran (SQL suka aritmetika **dalam** baris; hati-hati campur amount inkompatibel di satu kolom); **OLAP lebih toleran** |

---

## 5. Text, Image, Inventory, Late Arriving

**Freeform text comment:**
- ⚠️ **JANGAN** di fact table (boros, jarang di-query, **bukan** degenerate dimension).
- Kardinalitas ≈ jumlah event (hampir unik) → atribut **transaction event dimension**.
- Banyak "No Comment" berulang (kardinalitas < transaksi) → **comment dimension** + FK.
- Query gabungan teks+metrik lambat (join dua tabel voluminous) → drill setelah filter selektif.

**Image:** filename JPEG di fakta (akses bebas program lain, tapi **sinkronkan** DB file grafis terpisah) vs **blob** embedded (satu tempat, tapi berat di DB).

**Facility/equipment inventory** (pola Bab 4):
- **Periodic snapshot** (factless): status tiap bed di titik reguler (midnight/shift), FK patient/attending physician/nurse.
- **Transaction**: 1 baris per gerakan masuk/keluar bed (type: filled/vacated); operating room: pre-op/post-op/downtime + durasi.
- **Timespan** (Bab 8): effective/expiration date-time bila perubahan tak volatil (rehab/eldercare).

**Late arriving data (retroaktif):** di kesehatan (legacy) **lazim** — prosedur beberapa minggu lalu, update profil back-dated berbulan. Makin terlambat, makin menantang ETL; bisa jadi **mode dominan** (teknik di Bab 19).

---

## Ringkasan Teknis

**Accumulating snapshot** claims workflow: grain per line klaim, **8 role-playing date** (view terpisah, TBD date non-null), **update destruktif** (reorg fisik, partisi pada treatment date), fakta **lag** + companion transaction. **Diagnosis group bridge**: desain A (many-to-many) vs B (group dimension untuk FK→PK proper); **tanpa weighting** → impact query yang overcount; **ETL lookup** portofolio grup; **twin date** untuk inpatient evolving. **Supertype/subtype charges**: admitting/attending physician role, admitting/discharge diagnosis group, bridge tim bedah. **Measure type dimension**: grain per pengukuran (fleksibel, bebas null, tapi baris meledak & kalkulasi lintas-baris sulit → OLAP toleran; kembali ke kolom tetap saat padat). **Text** di comment/transaction dimension (kardinalitas menentukan), **image** filename vs blob, **inventory** snapshot/transaction/timespan, **late arriving** mode dominan.
