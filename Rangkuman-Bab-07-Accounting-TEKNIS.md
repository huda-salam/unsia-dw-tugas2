# Bab 7 — Accounting (Versi Mendalam & Teknis)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · **kedalaman teknis** — struktur skema, SQL, implikasi ETL, analisis trade-off

Versi ini membedah Bab 7 sampai ke tingkat implementasi: struktur tabel, kunci, pertimbangan fisik, snippet SQL, dan konsekuensi ETL dari tiap keputusan desain. Fokus utamanya adalah bagian yang paling berat secara teknis di bab ini: **penanganan hierarki ragged dengan bridge table**.

---

## 1. General Ledger Periodic Snapshot — Struktur & Nuansa

### 1.1 Skema inti

```
General Ledger Snapshot Fact
────────────────────────────
Accounting Period Key (FK)   ──► Accounting Period Dimension
Ledger Key (FK)              ──► Ledger Dimension
Account Key (FK)             ──► Account Dimension
Organization Key (FK)        ──► Organization Dimension
Period End Balance Amount    (semi-additive)
Period Debit Amount          (fully additive)
Period Credit Amount         (fully additive)
Period Net Change Amount     (fully additive)
```

- **Grain:** satu baris per (accounting period × ledger × account × organization) pada level **terendah** chart of accounts. Ini bukan grain transaksi — grain dinyatakan lewat daftar dimensi (khas periodic snapshot).
- **Primary key komposit fisik:** gabungan keempat FK. Karena grain snapshot, tabel ini **dense** relatif terhadap kombinasi akun aktif × periode.

### 1.2 Chart of accounts: dekomposisi intelligent key

Chart of accounts operasional adalah *smart key* multipart (mis. `1234-56-789` = akun-costcenter-subledger). **Aturan desain:**
- **Jangan** wariskan intelligent key ke DW sebagai satu kolom yang harus di-parse. **Dekomposisi** jadi:
  - **Account dimension** — `Account Key (PK surrogate)`, `Account Name`, `Account Category`, `Account Type` (asset/liability/equity/income/expense). Rentang digit (1000–1999) → materialisasi sebagai atribut `Account Type`, bukan filter `LEFT(account_no,1)`.
  - **Organization dimension** — rollup cost center → department → division → business unit → company. Jika rollup **seimbang & kedalaman tetap** → atribut hierarkis paralel (fixed depth). Jika **tak seimbang** → butuh teknik ragged (§5).
- **Uniform chart of accounts** = conformed account dimension lintas multi-organisasi. Implikasi ETL: butuh **cross-reference map** dari kode akun tiap organisasi ke `Account Key` master, plus proses governance untuk menyepakati nama/definisi akun.

### 1.3 Ledger dimension — jebakan double-counting

`Ledger` (set of books: statutory, tax, regulatory) itu **filter fundamental**, bukan sekadar atribut. Bahaya: satu fact table memuat beberapa ledger untuk akun/periode/organisasi yang sama → **query tanpa constraint ledger akan menjumlahkan lintas ledger** (nilai sama dihitung berkali-kali).

**Mitigasi (defensif):**
```sql
-- JANGAN biarkan pengguna query tabel mentah. Rilis view per-ledger:
CREATE VIEW gl_snapshot_final_domestic AS
SELECT ... FROM gl_snapshot_fact f
WHERE f.ledger_key = (SELECT ledger_key FROM ledger_dim
                      WHERE ledger_book_name = 'Final Approved Domestic');
```
Semua BI mengakses via view ber-constraint; tabel fisik tertutup dari ad hoc query.

### 1.4 Balance sebagai fakta semi-aditif

`Period End Balance` **tidak boleh dijumlahkan lintas dimensi Accounting Period**. Konsekuensi:
- Agregasi lintas waktu → **rata-rata** atau ambil nilai periode terakhir, bukan `SUM`.
- Tetap disimpan (meski bukan "aktivitas") karena alternatifnya menghitung ulang saldo dari awal waktu — mahal.
- Di OLAP: definisikan *aggregation rule* `LastNonEmpty`/`Average` pada measure balance agar cube menangani semi-aditivitas otomatis.

### 1.5 Mengapa G/L "tidak nyambung" dengan operasional

Account & Organization **hampir tak pernah conform** ke customer/product/facility operasional. Ini isu **struktural** di data sumber, bukan bug yang bisa diperbaiki tim DW. Kelola ekspektasi pengguna sejak wawancara.

### 1.6 Year-to-date: mengapa jangan disimpan

Fakta harus konsisten dengan grain. Kolom YTD/QTD **tidak true-to-grain** → saat baris di-`SUM`/`GROUP BY` arbitrer, YTD ikut terjumlah dan menghasilkan angka *overstated* yang ngawur. **Aturan:** hitung to-date di BI (window function `SUM() OVER (PARTITION BY ... ORDER BY period)`), bukan materialisasi di fakta. OLAP menangani measure YTD lebih anggun via kalkulasi cube.

---

## 2. Journal Entry Transaction — Struktur & Sorting

```
General Ledger Journal Entry Fact
─────────────────────────────────
Post Date Key (FK)              ──► Post Date Dimension (grain harian)
Journal Entry Effective Date/Time   ← timestamp presisi (opsional, untuk sorting)
Ledger Key (FK), Account Key (FK), Organization Key (FK)
Debit-Credit Indicator Key (FK) ──► Debit-Credit Indicator Dim (tepat 2 baris)
Journal Entry Number (DD)       ← degenerate dimension
Journal Entry Amount
```

- **Grain:** satu baris per jurnal entry (baris debit/kredit). Melengkapi snapshot untuk drill ke detail saat ada anomali di level ringkasan.
- **Sorting:** dimensi tanggal (grain hari) **terlalu kasar** untuk mengurutkan entry dalam satu hari. Jika `Journal Entry Number` terurut → pakai DD itu untuk `ORDER BY`. Jika tidak → **wajib** tambahkan `Effective Date/Time` timestamp.
- **Transaction profile:** jika ada tipe/deskripsi (bukan freeform), buat **profile dimension** terpisah — baris jauh lebih sedikit dari fakta (satu baris fakta per jurnal line). Nomor jurnal **tetap** degenerate.

### 2.1 Multiple fiscal calendar — empat opsi implementasi

Masalah: periode fiskal (mis. 13 × 4-minggu mulai 1 Sep, atau pola **5-4-4**) ≠ bulan Masehi.

| Skenario | Solusi | Implikasi |
|---|---|---|
| 1 kalender fiskal | Atribut hierarkis di date dim harian | Termudah; date dim conform ke calendar-month & fiscal-period dim sekaligus |
| Sedikit kalender (per subsidiary) | Set atribut fiskal berlabel unik di **satu** date dim | Satu baris tanggal = period 1 utk Subsidiary A, period 7 utk Subsidiary B |
| Banyak kalender | **Date outrigger** key = (date_key, subsidiary_key), 1 baris/hari/subsidiary; filter subsidiary via view | ETL memelihara outrigger; view menyamarkannya jadi bagian date dim |
| Banyak kalender, fakta terdesentralisasi | **Date dim fisik terpisah** per subsidiary, surrogate key bersama | Atau: FK ke subsidiary-fiscal-period dim (~36 periode × jumlah kalender) — **beban ETL** menyisipkan fiscal period key saat transform |

---

## 3. Drilling Down Multilevel Ledger — Parent Snapshot Key

Enterprise besar: ledger bertingkat (enterprise → division → department). Modelkan dengan **fact table surrogate key** + **parent snapshot key** (relasi parent/child antar baris **fakta**):

```
GL Snapshot Fact
────────────────
Fact Table Surrogate Key (PK)   ← integer, di-increment saat insert
Parent Snapshot Key (FK)        ← menunjuk Fact Table Surrogate Key induk
... (FK dimensi + fakta)
```

Drill-down (temukan kontributor level bawah dari entry travel besar):
```sql
SELECT * FROM GL_Fact
WHERE Parent_Snapshot_Key = (
  SELECT fact_table_surrogate_key
  FROM GL_Fact f JOIN Account a ON <joins>
  WHERE a.Account = 'Travel' AND f.Amount > 1000
);
```

---

## 4. Budget Chain — Grain "Net Change" (analisis mendalam)

Rantai: **Budget Fact → Commitment Fact → Payment Fact**. Dimensi bertambah sepanjang rantai (commitment menambah commitment party/dokumen; payment menambah payee).

### 4.1 Mengapa grain "net change", bukan "snapshot status"?

Grain snapshot status (saldo tiap line item tiap bulan) tampak wajar (mirip laporan manajemen) tapi **buruk**:
1. Faktanya jadi **saldo semi-aditif**, bukan fully additive.
2. Menghitung "berubah berapa sejak bulan lalu" perlu mengambil beberapa periode lalu menguranginya.
3. **Baris duplikat** menumpuk saat tak ada perubahan di bulan-bulan berturut.

Grain **net change** menyelesaikan ketiganya:
```
Budget Fact (grain = net change per line item × account × cost center × effective month)
──────────
Budget Effective Date Key (FK)
Budget Line Item Key (FK)
Account Key (FK)
Organization Key (FK)
Budget Amount   (fully additive)
```
- **Current budget** = `SUM(Budget Amount)` dari awal waktu s/d tanggal tertentu → tepat menghasilkan budget disetujui terkini, hanya line item yang punya budget.
- **Perubahan bulan X** = constrain `effective_month = X` → hanya line item yang berubah bulan itu.
- Contoh: approval awal $200.000 (baris Jan) → +$40.000 (baris Jun) → −$25.000 (baris Okt). Menjumlahkan = $215.000.

### 4.2 Detail implementasi

- **Ragged budget line hierarchy:** bila sebagian line hanya punya category tanpa subcategory → **replikasi** nama category ke kolom subcategory (hindari bucket "Not Applicable").
- **Budget tahunan vs bulanan:** jika budget per spending month, tambahkan **month dimension kedua** yang berperan sebagai *spending month* (role-playing) di samping *effective month*.
- **Satu budget line → banyak akun G/L:** **alokasikan**; satu budget line cost center = beberapa baris fakta (grain per akun).
- **Drill-across** budget vs commitment vs payment: multipass SQL, gabungkan pada row header (organization + line item), lalu hitung variance di BI.

---

## 5. Hierarki Ragged — Bridge Table (bagian paling teknis)

### 5.1 Mengapa recursive pointer bermasalah

Desain klasik parent/child: kolom `Organization Parent Key` (self-reference) di organization dimension. Ditelusuri via `CONNECT BY` (Oracle) atau *recursive CTE* (SQL Server). **Dua masalah fatal:**
1. Definisi pohon **terjerat** di dimensi (bergantung recursive pointer di data) → **tak praktis** mengganti struktur rollup alternatif tanpa memodifikasi pointer secara destruktif.
2. **Tak praktis** menerapkan SCD Type 2 — mengubah key node level-atas akan merambatkan perubahan key ke seluruh anak-cucu di bawahnya.

### 5.2 Solusi: hierarchy bridge table

Buat tabel jembatan **independen** dari dimensi, berisi seluruh informasi rollup:

```
Organization Map Bridge
───────────────────────
Parent Organization Key (FK)
Child Organization Key (FK)
Depth from Parent
Highest Parent Flag
Lowest Child Flag
```

- **Grain:** satu baris untuk **setiap path** dari tiap parent ke tiap child di bawahnya — **termasuk baris parent-ke-dirinya-sendiri**.
- Pohon 13 node (contoh buku) → **43 baris** (13 path dari node 1, 5 path dari node 2, 1 path node 3→3, dst.).
- `Highest Parent Flag` = path berasal dari puncak pohon; `Lowest Child Flag` = path berakhir di *leaf node*.

**Query rollup (join dimension → bridge → fact):**
```sql
-- Roll up seluruh subtree di bawah node 1 (constrain dimensi ke SATU baris):
SELECT SUM(f.amount)
FROM org_dim d
JOIN org_map_bridge b ON d.org_key = b.parent_org_key
JOIN gl_fact f        ON b.child_org_key = f.org_key
WHERE d.org_number = 1;             -- 13 hits, seluruh pohon, tanpa traversal runtime

-- Hanya leaf node:
... AND b.lowest_child_flag = 'Y';  -- 6 hits (node 3,5,6,8,10,11)
```

⚠️ **Bahaya overcounting:** constraint dimensi **harus** memilih **satu** parent. Jika memfilter atribut yang cocok ke **banyak** node (mis. `state = 'California'`), join sederhana akan menghitung anak/cucu berkali-kali. Gunakan subquery:
```sql
SELECT SUM(f.amount) FROM gl_fact f
WHERE f.org_key IN (
  SELECT DISTINCT b.child_org_key
  FROM org_dim d JOIN org_map_bridge b ON d.org_key = b.parent_org_key
  WHERE d.state = 'California');
```

### 5.3 Varian bridge table

**Shared ownership** — tambah kolom `Percent Ownership`. Bila node 10 dimiliki 50% oleh node 6 & 50% oleh node 11: tambahkan path baru yang menghubungkan node 10 ke node 11 (dan induknya); semua path yang berakhir di node 10 diberi bobot 50%. Fakta yang di-rollup dikalikan `percent_ownership`.

**Time-varying** — tambah `Begin Effective Date/Time` & `End Effective Date/Time`. Saat relasi berubah: expire path lama, insert path baru. ⚠️ **Query wajib constrain ke satu titik waktu** untuk "membekukan" pohon — tanpa itu, path yang tak mungkin koeksis ikut terambil.

**Modifikasi struktur (static)** — pindahkan node 4,5,6 dari induk 2 ke induk 9:
```sql
DELETE FROM org_map WHERE child_org IN (4,5,6) AND parent_org NOT IN (4,5,6);
INSERT INTO org_map (parent_org, child_org)
  SELECT parent_org, 4 FROM org_map WHERE parent_org IN (1,7,9);
-- ulangi utk 5 dan 6
```

### 5.4 Alternatif (dan kelemahannya)

| Pendekatan | Cara | Kelemahan |
|---|---|---|
| **Pathstring** | Atribut string per node (mis. `AABA.`), `+`=punya anak, `.`=leaf; navigasi via wildcard (`A*` seluruh pohon, `*.` leaf) | Rentan **relabeling ripple**: sisipkan node → semua node kanan di bawah parent sama harus dilabeli ulang |
| **Modified preorder tree traversal** | Sepasang angka Left/Right per node; subtree = `Left BETWEEN x AND y`; leaf = `Right − Left = 1` | **Paling rentan** relabeling: seluruh pohon dinomori berurutan; perubahan apa pun → seluruh sisi kanan dinomori ulang |

Keduanya juga **mengunci** definisi hierarki ke dalam dimensi (tak bisa punya rollup alternatif).

### 5.5 Keunggulan bridge table (ringkasan)

Meski butuh lebih banyak kerja ETL (build) & query (join), bridge table memberi: **rollup alternatif dipilih saat query**, **shared ownership**, **time-varying**, **dampak minimal** saat SCD Type 2 & saat struktur pohon berubah. Bridge yang sama bisa dipakai untuk drill-across ketiga fact table budget chain.

---

## 6. Consolidated Fact Table & OLAP

```
Budget Variance Fact (consolidated)
───────────────────────────────────
Accounting Period Key (FK), Account Key (FK), Organization Key (FK)
Accounting Period Actual Amount
Accounting Period Budget Amount
Accounting Period Budget Variance   ← selisih terhitung
```

- Buat consolidated fact **hanya jika drill-across (actual vs budget) sangat sering** — menggabungkan sekali di ETL lebih murah daripada stitch berulang di BI (isu kompleksitas, akurasi, kapabilitas tool, performa).
- ⚠️ **Syarat mutlak:** fakta gabungan harus pada **grain & dimensionalitas sama**. Karena jarang alami sama grain, sebagian dimensi **diagregasi/dihilangkan** → consolidated = kompromi *least-common-denominator*. **Data atomik tetap di fact table terpisah.** **Jangan** membuat fakta/dimensi artifisial demi memaksa konsolidasi grain berbeda.
- **Multi-currency variance:** simpan actual dalam mata uang lokal + korporat, dan konversi berdasar **effective rate** maupun **planned rate** — agar manajer tak "dihukum" fluktuasi kurs di luar kendalinya, dan finance memahami dampak kurs pada rencana tahunan.

**OLAP & paket analitik:** OLAP unggul untuk keuangan (rollup organisasi kompleks, fungsi finansial NPV/CAGR, urutan laporan income-before-expenses, penanganan debit/kredit per account type, security berlapis). Relational star sering **memberi makan** cube finansial (volume GL balance umumnya tak melampaui batas praktis OLAP). Paket siap-pakai mempercepat implementasi **tetapi wajib conform dimensi** — kalau tidak, berujung stovepipe antar paket (finance, CRM, HR, ERP dari vendor berbeda).

---

## Ringkasan Teknis

Dua skema komplementer: **GL periodic snapshot** (grain per periode×akun; balance semi-aditif via `LastNonEmpty`/average; ledger **wajib** di-constrain via view; account/organization dari **uniform chart of accounts** conformed) dan **journal transaction** (grain per entry; debit/credit 2-nilai; sorting via DD terurut atau timestamp). Tangani **multi-currency** (lokal+korporat), **multi fiscal calendar** (atribut/outrigger/dim-terpisah/FK), **YTD** (window function di BI, jangan materialisasi), dan **multilevel ledger** (parent snapshot key). Budget chain memakai grain **net change** (fully additive, bebas duplikat). Puncak teknis: **ragged hierarchy bridge table** (grain = path parent→child, flag highest-parent & lowest-child, hati-hati **overcounting** → pakai subquery `DISTINCT child_key`), dengan varian **shared ownership** (percent), **time-varying** (date stamp, wajib freeze ke satu waktu), dan modifikasi via DELETE/INSERT — lebih unggul daripada recursive pointer/pathstring/preorder-traversal yang rentan *relabeling ripple*. **Consolidated fact table** = kompromi least-common-denominator (data atomik tetap terpisah). OLAP cocok untuk keuangan tetapi tetap butuh conformance.
