# Bab 10 — Financial Services (Versi Mendalam & Teknis)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · **kedalaman teknis** — struktur skema, SQL, implikasi ETL, analisis trade-off

Fokus teknis: **weighting factor & alokasi bridge**, **multiple mini-dimension + banding**, **dynamic value banding** (join `<`/`>` & performa columnar), dan **skema supertype/subtype** (shrunken conformed, disjoint partitioning, duplikasi fakta).

---

## 1. Dimension Triage — dari 2 dimensi ke 7

**Anti-pola (Figure 10-2):** grain "1 baris/akun/bulan", hanya 2 FK (month, account), semua deskriptor lain ditumpuk sebagai atribut account.
```
Month Account Snapshot Fact          Account Dimension (RAKSASA)
  Month End Date Key (FK)              Account + Primary Customer + Product
  Account Key (FK)                     + Household + Status + Branch Attributes
  Primary Month Ending Balance         (semua ditumpuk di sini)
```
**Masalah:** melanggar cara pikir bisnis; account dimension jadi raksasa (>10 juta baris) yang meledak karena Type 2.

**Solusi triage → 7 dimensi:** month end date, branch, account, primary customer, product, account status, household.
```
Monthly Account Snapshot Fact
  Month End Date Key, Branch Key, Account Key, Primary Customer Key,
  Product Key, Account Status Key, Household Key (semua FK)
  Primary Month Ending Balance   (SEMI-additive → rata-rata lintas waktu)
  Average Daily Balance, Number of Transactions, Interest Paid, Fees Charged
```

**Aturan triage:** model normal 5–20 dimensi. Di bawah batas → cek kandidat: causal, multiple date, degenerate, role-playing, status, audit, junk. Semua bisa ditambah **tanpa mengubah grain** (aplikasi lama tetap jalan). Kriteria: **atribut single-valued di hadapan fakta** = kandidat dimensi.

Pemisahan product & branch (many-to-many, irama Type 2 berbeda) mengurangi ledakan baris account.

---

## 2. Account–Customer Bridge & Weighting Factor

Customer multivalued pada grain akun → **bridge**:
```
Monthly Account Snapshot Fact ──► Account Dimension
        │                          Account-to-Customer Bridge
        └──────────────────────►   Account Key (FK)
                                    Customer Key (FK)  ──► Customer Dimension
                                    Weighting Factor       (Σ = 1.00 per akun)
```

**Correctly weighted report** (alokasi fakta aditif ke pemegang, grand total benar):
```sql
SELECT c.customer_attr, SUM(f.balance * b.weighting_factor)
FROM fact f JOIN acct_cust_bridge b ON f.account_key=b.account_key
            JOIN customer c ON b.customer_key=c.customer_key
GROUP BY c.customer_attr;
```

**Impact report** (tanpa weighting → berpotensi overcount):
```sql
SELECT c.customer_attr, SUM(f.balance)   -- fakta dikaitkan ke SEMUA pemegang
FROM fact f JOIN acct_cust_bridge b ON ... JOIN customer c ON ...
GROUP BY c.customer_attr;
```

**Mengapa TIDAK mengubah grain jadi per-account-holder?** (1) Ukuran fakta dikali rata-rata jumlah pemegang; (2) bila ada >1 dimensi multivalued → baris meledak, makna baris kabur; (3) sulit merekonstruksi angka *unallocated*. → Simpan weighting di bridge, sediakan **dua view** (dengan/tanpa weighting).

Bila customer teridentifikasi per transaksi (nomor kartu unik) → grain transaksi punya account & customer sebagai FK langsung (tak perlu bridge); bridge tetap untuk metrik level-akun.

---

## 3. Multiple Mini-Dimension + Banding

```
Fact Table
  Customer Key (FK)              ──► Customer Dim (atribut relatif konstan)
  Customer Demographics Key (FK) ──► Demographics Mini-Dim (Age/Income Band, Marital)
  Customer Risk Profile Key (FK) ──► Risk Profile Mini-Dim (Risk/Delinquency Cluster)
```
- Fakta = **periodic snapshot berjalan lama** → tiap bulan ada baris per akun = "rumah" bagi semua FK mini-dim (bisa update beda frekuensi).
- ⚠️ Mini-dim = **gumpalan atribut berkorelasi**; jangan tiap atribut jadi mini-dim (kebanyakan dimensi).

**Banding + nilai mentah (redundansi terkendali):** setiap bulan assign band mini-dim (profitability 1–1200 → `≤100`, `101–150`, …), **dan** simpan **skor mentah sebagai fakta** di snapshot (untuk data mining/power analyst); opsional nilai kini di account dimension (Type 1). Label unik agar terbedakan. Timbang nilai tambah vs biaya kompleksitas ETL & BI.

**Mini-dimension pada bridge (monster dimension):** bila account-to-customer bridge membengkak (mis. 20 juta akun × 25 juta customer, keduanya Type 2 → ratusan juta baris) dan customer jadi monster → tambah **Demographics Key** ke **bridge**:
```
Account-to-Customer Bridge
  Account Key (FK), Customer Key (FK), Demographics Key (FK)   -- key = triple
```
Batasi perubahan ke **akhir bulan** (sesuai grain fakta) → menahan pertumbuhan bridge.

---

## 4. Dynamic Value Banding of Facts

**Masalah:** band atas **fakta** (saldo), rentang tak sama besar & bernama teks, redefinisi saat query. SQL `GROUP BY` tak bisa mengelompokkan nilai aditif ke rentang.

**Solusi:**
```
Band Definition Table                    Monthly Account Snapshot Fact
  Band Group Key (PK)                      Primary Month Ending Balance
  Band Group Sort Order (PK)
  Band Group Name
  Band Range Name       -- "10001 and up"
  Band Lower Value, Band Upper Value
```
Join dengan **`>=` dan `<`** (bukan equijoin):
```sql
SELECT d.band_range_name, COUNT(*) AS num_accounts, SUM(f.balance) AS total
FROM fact f JOIN band_def d
  ON f.balance >= d.band_lower_value AND f.balance < d.band_upper_value
WHERE d.band_group_name = 'Standard Balance Bands'
GROUP BY d.band_range_name, d.band_group_sort_order
ORDER BY d.band_group_sort_order;
```
⚠️ **Performa:** query nyaris tanpa constraint (scan jutaan saldo); join banding tidak me-restrict (justru grouping). Butuh **indeks langsung pada fakta saldo**; **columnar DB** (Sybase IQ, dst.) sort & kompres fakta → percepatan besar.

---

## 5. Supertype/Subtype untuk Produk Heterogen

**Dilema:** produk heterogen (checking vs deposito) — fakta/atribut sebagian besar disjoint. Dua perspektif tak muat di satu fakta.

**Supertype (global):**
```
Monthly Account Snapshot Fact (supertype)
  ... hanya fakta UMUM lintas lini (balance, interest, fees, txn count)
Supertype Account/Product Dimension
  ... hanya atribut UMUM
```
Tak bisa menampung ratusan fakta inkompatibel.

**Subtype (line-of-business, mis. checking):**
```
Checking Account Fact (subtype)          Checking Account Dimension (subtype)
  Balance, Change in Balance,              Account Key (PK) ← SAMA dgn supertype
  Total Deposits/Withdrawals,              + atribut khusus checking
  Max Backup Reserve, Number Overdraws,   Checking Product Dimension
  Overdraw Penalties, ATM/Online counts,   Premium Flag, Checking Type,
  Days Below Minimum, +10 fakta lain       Overdraft Policy, +12 atribut
```

**Aturan kunci (teknis):**
- Subtype fact/dim memuat fakta khusus **DAN** fakta supertype (hindari join supertype↔subtype untuk set lengkap).
- **Surrogate key sama** di supertype & subtype (satu akun = key sama di keduanya).
- Tiap subtype account dim = **shrunken conformed dimension** (subset baris supertype + atribut spesifik).
- Subtype **disjoint** (partisi akun tanpa tumpang tindih) → jarang masuk akal menggabungkan >1 subtype.
- Hanya **DBA** yang lihat semua tabel; pengguna pakai supertype (cross-product) **atau** satu subtype (deep dive).
- Bila lini terpisah fisik → data supertype **diduplikasi tepat sekali** untuk mengisi semua subtype (disjoint, tanpa overlap).

**Varian common facts:** bila metrik proses tak bervariasi per lini (mis. solicitation) → cukup **satu supertype fact table**, tapi tetap punya portofolio subtype **account dimension** untuk dipakai sesuai analisis.

> Keluarga supertype/subtype fact table diperlukan saat **produk heterogen** (fakta/deskriptor berbeda alami) + **satu basis customer** menuntut pandangan terintegrasi. Berlaku umum (mis. hardware/software/services).

---

## 6. Hot Swappable Dimension

Broker: banyak klien, **satu** fact table harga saham (HLC harian), tapi tiap klien punya **stock dimension rahasia** sendiri.
```
Stock Price Fact ◄── (query time) ── Stock Dimension [Client A]
                                   ── Stock Dimension [Client B]  (ditukar per query)
```
**Implementasi:** RI constraint fakta↔berbagai stock dimension biasanya **dimatikan** agar penukaran per-query bisa terjadi.

---

## Ringkasan Teknis

**Dimension triage:** 2→7 dimensi (5–20 normal; atribut single-valued = kandidat; tambah tanpa ubah grain). **Balance semi-aditif** (rata-rata lintas waktu). **Account-customer bridge + weighting factor** (Σ=1.00): correctly weighted (`SUM(fact*weight)`) vs impact (`SUM(fact)`, overcount); jangan ubah grain (ledakan baris, unallocated hilang); dua view. **Multiple mini-dimension** Type 4 (gumpalan berkorelasi) + banding + nilai mentah sebagai fakta; mini-dim bisa ditambah ke **bridge** (key triple, batasi ke akhir bulan). **Dynamic value banding**: band definition table + join `>=`/`<`, butuh indeks fakta/columnar. **Supertype/subtype**: supertype = fakta umum (global), subtype = fakta khusus + fakta umum (deep dive), **key sama**, **shrunken conformed**, **disjoint**, duplikasi fakta sekali. **Hot swappable dimension**: RI dimatikan, tukar dimensi per query.
