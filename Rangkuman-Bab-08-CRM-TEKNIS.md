# Bab 8 — Customer Relationship Management (Versi Mendalam & Teknis)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · **kedalaman teknis** — struktur skema, SQL, implikasi ETL, analisis trade-off

Versi ini membedah Bab 8 ke tingkat implementasi: struktur tabel & kunci, snippet SQL (khususnya query timespan & anti-pola fact-to-fact), mekanika bridge table, dan konsekuensi ETL. Fokus teknis utama: **bridge table multivalued**, **timespan fact table dengan dual date/time stamp**, **step dimension**, dan **behavior study group**.

---

## 1. Anatomi Customer Dimension

Customer dimension = dimensi tersulit: **deep** (>100 juta baris di peritel/kartu kredit/pemerintah besar), **wide** (puluhan–ratusan atribut), **cepat berubah**, dan **amalgamasi** banyak sumber internal+eksternal.

### 1.1 Name/address parsing — dari kolom generik ke atribut elemental

**Anti-pola sumber operasional:**
```
Name        : "Ms. R. Jane Smith, Atty"
Address 1   : "123 Main Rd, North West, Ste 100A"
Address 2   : "PO Box 2348"
City/State  : "Kensington" / "Ark."
Phone       : "888-555-3333 x776 main, 555-4444 fax"
```
Masalah: tak ada mekanisme konsisten untuk salutation/title/suffix; bisa ada **beberapa customer dalam satu kolom nama**; singkatan tak konsisten; tak bisa validasi ZIP↔state.

**Target: dekomposisi elemental** (contoh individu AS):
```
Salutation, Informal Greeting Name, Formal Greeting Name,
First and Middle Names, Surname, Suffix, Ethnicity, Title,
Street Number, Street Name, Street Type, Street Direction,
City, District, Second District, State, Region, Country, Continent,
Primary/Secondary Postal Code, Postal Code Type,
{Office/Mobile} Telephone {Country/Area} Code + Number + Extension,
E-mail, Web Site, Public Key Authentication, Certificate Authority,
Unique Individual Identifier
```

**Pipeline ETL:** *parse* (pisah kolom kotor) → *standardize* (`Rd`→`Road`, `Ste`→`Suite`) → *verify* (ZIP↔state, lat/long). Pakai tool name/address cleansing. Customer komersial: banyak alamat (fisik, shipping) → tiap alamat ikut struktur yang sama.

### 1.2 Internasional: character set, bukan font

- **Font** = rendering artistik; **character set** = masalah sesungguhnya. ASCII = 8-bit, maks 255 karakter (~100 dari keyboard Inggris) — tak cukup untuk Cyrillic/Arab/CJK.
- **Unicode** (v6.2.0: 110.182 karakter) adalah fondasi. **Implementasi di lapisan fondasi:** OS harus Unicode-compliant, lalu semua device yang capture/store/transmit/display.

---

## 2. Customer-Centric Dates sebagai Outrigger

```
Date of 1st Purchase Dimension (outrigger)      Customer Dimension
────────────────────────────────────           ──────────────────
Date of 1st Purchase Key (PK)      ◄──────────  Customer Key (PK)
Date of 1st Purchase Month                       Customer ID (NK)
Date of 1st Purchase Year                        ...
Date of 1st Purchase Fiscal Quarter              Date of 1st Purchase Key (FK)
Date of 1st Purchase Season
```

- Ubah kolom tanggal SQL menjadi **FK ke date dimension** agar bisa dikelompokkan per season/fiskal.
- Diimplementasi sebagai **view role-playing** dengan label kolom unik (`Date of 1st Purchase Month`) — berperilaku seolah tabel date fisik terpisah; constraint-nya independen dari date dimension utama.
- ⚠️ Semua tanggal **wajib** dalam rentang date dimension korporat.

---

## 3. Aggregated Facts sebagai Atribut — beban ETL

Simpan mis. `Last Year Spend` / `Lifetime Spend` sebagai **atribut** customer dimension → pengguna filter "high spender" seperti filter geografi, **tanpa** query fakta dua tahap.

**Aturan & trade-off:**
- Hanya untuk **constraining/labeling**, **bukan** kalkulasi numerik.
- Beban ada di **ETL**: harus akurat, mutakhir, konsisten dengan baris fakta ("care and feeding" signifikan).
- Pilih atribut **sering dipakai** & **minim update** — `Last Year Spend` jauh lebih ringan dari `Year-to-Date` (yang harus di-refresh terus).
- Sering diganti/dilengkapi **label deskriptif** (`High Spender`) → mengurangi risiko angka tak nyambung ke fakta + menjamin definisi konsisten lintas pengguna.

---

## 4. Behavior Tag Time Series — desain posisional

**RFI/RFM:** Recency, Frequency, Intensity (atau Monetary), diskalakan **kuintil 1–5** → tiap pelanggan = titik di kubus RFI. Data mining → cluster jadi **behavior tag** (A..H).

**Deret waktu tekstual:** `John Doe: C C C D D A A A B B`.

**Keputusan desain (kritis):**
- **JANGAN** simpan tag sebagai baris fakta terpisah → query pola akan butuh *cascade of correlated subqueries* yang sangat mahal.
- **Simpan sebagai deret atribut posisional di customer dimension** — satu kolom per periode → BI sederhana (kolom di tabel sama), performa baik (**bitmap index**).
- Tambahkan **kolom konkatenasi** `CCCDDAAABB` untuk wildcard pattern (`%D%B%` = "D lalu B").
- Opsional: nilai tag kontemporer di **mini-dimension** untuk menganalisis fakta berdasar tag yang berlaku saat baris fakta dimuat.

**Data mining velocity mismatch:** decision tree memproses ratusan record/detik; drill-across 7-arah (census + demographic + credit + purchases + returns + web) menghasilkan jutaan "customer observation" tapi tak bisa secepat itu. **Solusi:** materialisasi answer set ke **file**, serahkan ke server data mining (analisis berulang: decision tree "diarahkan" ke kolom target mis. `Total Value Purchases Lifetime` untuk cari prediktor variance) — jangan query ulang on-demand.

---

## 5. Counts dengan SCD Type 2 — hindari overcounting

Satu individu = banyak baris (Type 2) → hitung naif = overcount.
```sql
-- Hitung berdasar nilai terkini:
SELECT COUNT(DISTINCT customer_durable_id) FROM customer_dim WHERE current_row_ind='Y';

-- Hitung pada titik historis (1 Jan 2013):
SELECT COUNT(DISTINCT customer_durable_id) FROM customer_dim
WHERE row_effective_date <= '2013-01-01' AND row_expiration_date >= '2013-01-01';
```
Operator perbandingan bergantung aturan bisnis effective/expiration (di sini expiration baris lama = 1 hari sebelum effective baris baru).

---

## 6. Outrigger — kapan snowflake diperbolehkan

```
County Demographics Outrigger (150 atribut)     Customer Dimension
──────────────────────────────────────          ──────────────────
County Demographics Key (PK)   ◄─────────────    ... Customer County ...
Total Population, % under 5, % 65+,              County Demographics Key (FK)
Home Ownership Rate, ...
```
Pembenaran melanggar aturan "no snowflake": (1) grain berbeda (per county, bukan per customer) & nilai analitik lebih rendah; (2) di-load pada waktu berbeda; (3) hemat ruang besar bila customer dimension raksasa; (4) bisa disembunyikan di view untuk tool yang menuntut star murni.
> ⚠️ **Merah** bila desain penuh outrigger → gejala over-normalisasi.

**Hierarki customer komersial:** transfer langsung teknik Bab 7 — fixed depth (level tetap, isi ketiga level), slightly variable (duplikasi nilai level bawah ke atas agar semua level menjumlah sama), **ragged variable depth** (bridge table). Jangan `Level-1/Level-2` abstrak.

---

## 7. Bridge Table Multivalued — mekanika

**Aturan grain:** dimensi yang menempel ke fakta idealnya **single-valued** (satu FK bersih). Dimensi *multivalued* pada grain transaksi: demografi multi-sumber, alamat kontak, keahlian, hobi, diagnosis, fitur mobil, joint account holder, tenant.

| Pendekatan | Struktur | Trade-off |
|---|---|---|
| **Positional** | Kolom bernama per nilai | Mudah di-query, tapi **tak skalabel** (habis kolom, banyak null); batas ~100 kolom (columnar DB lebih toleran, kompresi low-cardinality bagus) |
| **Bridge table** | Baris hanya bila perlu | Skalabel, bebas null; **query kompleks**, ⚠️ kadang di luar jangkauan SQL tool BI → sembunyikan di view |

### 7.1 Bridge untuk name-value pair (sparse)

```
Loan Application Fact                Application Disclosure Bridge     Disclosure Item Dimension
─────────────────────                ─────────────────────────────    ─────────────────────────
... Application Disclosure Key (FK) ─► Application Disclosure Key (FK)  Disclosure Item Key (PK)
                                       Disclosure Item Key (FK) ──────► Item Name
                                                                        Item Value Type
                                                                        Item Value Text String
```
Nilai disimpan sebagai **text string** (menampung modalitas terbuka: numerik/teks/URL/pointer file/rekursif). Cocok saat jumlah variabel **open-ended & tak terprediksi** (ratusan–ribuan). Grain bridge = satu baris per (disclosure group × item).

### 7.2 Bridge untuk multiple contact

```
Customer Dim ──► Contact Group Bridge ──► Contact Dim
                 (Contact Group Key FK,
                  Contact Key FK,
                  Contact Role)
```
Tiap kontak punya **role** (decision maker, purchasing agent, dll.). Batasi contact dimension pada konteks relasi pelanggan — jangan jadi "dumping ground".

---

## 8. Behavior Study Group (cohort) — teknik durable key

**Masalah:** "berapa pelanggan yang bulan ini belanja > rata-rata bulanan tahun lalu?" — terlalu kompleks untuk satu SQL.

**Teknik:**
1. Jalankan query (atau serangkaian) untuk mengidentifikasi himpunan (mis. top 100 customer).
2. **Tangkap durable key** mereka ke **tabel fisik satu kolom** (study group dimension). Durable key → **kebal SCD Type 2** yang terjadi setelahnya.
3. **Equijoin** study group ke durable key customer dimension (bisa via view → terlihat star biasa).

```
POS Retail Sales Fact ──► Customer Dim (Customer ID durable) ◄── Customer Behavior Study Group Dim
                                                                  (Customer ID durable)
```

- Constraint apa pun pada tabel mana pun kini cukup butuh **customer key reference** di fakta.
- **Set operations:** study group bisa di-UNION/INTERSECT/EXCEPT (mis. pelanggan bermasalah bulan ini ∩ bulan lalu).
- Tambah **kolom tanggal kejadian** untuk *panel study* (mis. konsumen masuk studi saat ganti merek → lacak pembelian berikutnya; butuh timestamp benar untuk urutan).
- **Konsekuensi DBA/ETL:** butuh UI untuk capture/create/administer tabel study group; tabel harus hidup di ruang yang sama dengan fakta (join langsung ke customer dim).

---

## 9. Step Dimension — perilaku sekuensial

Dimensi **abstrak** dibuat di muka; tiap baris menempatkan satu langkah dalam konteks sesi:
```
Step Dimension (contoh baris)
Step Key | Total Steps | This Step # | Steps Until End
   1     |     1       |     1       |      0          ← sesi 1-langkah
   2     |     2       |     1       |      1          ← sesi 2-langkah, langkah 1
   3     |     2       |     2       |      0
   4..6  |     3       |   1,2,3     |    2,1,0
```
Prebuild s/d ≥100 langkah. Di fact table (grain = page event) dipakai **beberapa peran**:
```
Transaction Fact
  Session Step Key (FK)   ← peran: sesi keseluruhan
  Purchase Step Key (FK)  ← peran: subsesi pembelian sukses
  Abandon Step Key (FK)   ← peran: keranjang ditinggalkan
```
→ query "halaman **pertama** dari sesi sukses" (attractant) atau "halaman **terakhir** keranjang ditinggalkan". Alternatif: kolom teks lebar berisi urutan kode (`11254|45882|53340|…`, DBMS modern menampung 64K+ char) untuk pencarian wildcard sekuensial.

---

## 10. Timespan Fact Table — dual date/time stamp

**Tujuan:** status pelanggan pada **instant arbitrer** di masa lalu. Kunci: sepasang stamp per transaksi membentuk **rangkaian tanpa celah**.
```
Customer Transaction Fact
  Transaction Date Key (FK), Customer Key (FK),
  Demographics Key (FK), Status Key (FK), Transaction Number (DD)
  Begin Effective Date/Time   ← momen transaksi ini
  End Effective Date/Time     ← momen transaksi berikutnya
  Amount
```

**Status pada instant tertentu** (Jane Smith, 18 Jul 2013 06:33):
```sql
SELECT c.Customer_Name, s.Status
FROM Transaction_Fact f JOIN Customer_dim c ON f.Customer_Key=c.Customer_Key
                        JOIN Status_dim  s ON f.Status_Key = s.Status_Key
WHERE c.Customer_Name='Jane Smith'
  AND #2013-07-18 06:33:00# >= f.Begin_Eff_DateTime
  AND #2013-07-18 06:33:00# <  f.End_Eff_DateTime;
```

**Semua yang pernah "Fraud Alert" di 2013** (menangani semua kasus straddle awal/akhir tahun dengan satu query):
```sql
... WHERE s.Status_Description='Fraud Alert'
    AND f.Begin_Eff_DateTime <= 2013-12-31:23:59:59
    AND f.End_Eff_DateTime   >= 2013-01-01:00:00:00;
```

**Jumlah hari tiap customer fraud alert di 2013** (klip ke batas tahun):
```sql
SELECT c.Customer_Name,
       SUM( LEAST(2013-12-31:23:59:59, f.End_Eff_DateTime)
          - GREATEST(2013-01-01:00:00:00, f.Begin_Eff_DateTime) )
FROM ... WHERE s.Status_Description='Fraud Alert' AND <rentang seperti di atas>
GROUP BY c.Customer_Name;
```

**Administrasi ETL (dua langkah, hati-hati):**
- End effective **tepat sama** dengan begin transaksi berikutnya (bukan "1 tick kurang") → hindari celah tempat transaksi bisa jatuh; konsekuensinya query pakai `>= begin AND < end`, bukan `BETWEEN`.
- Saat insert baris baru: (1) set end effective baris terkini ke **tanggal fiktif jauh di masa depan** (hindari NULL — bikin error di constraint); (2) setelah insert, ETL me-retrieve transaksi sebelumnya & set end effective-nya = begin transaksi baru.

---

## 11. Partial Conformity, Anti-Pola Fact-to-Fact, Low Latency

### 11.1 Partial conformity multiple customer dimension

Puluhan sumber (20 internal + 50+ eksternal), beda granularitas/kualitas, tanpa key konsisten → **satu** customer dim komprehensif sering mustahil. **Solusi ringan:** dua dim cukup *conformed* bila berbagi **subset atribut yang diadministrasi khusus** (nama kolom & nilai sama). Tanam bertahap: mulai `Customer Category`, lalu geografi (city/county/state/country), dst. — **tanpa mengubah grain** & tanpa mematahkan aplikasi lama. Drill-across bisa dilakukan pada dim mana pun yang sudah punya atribut itu. Cocok pendekatan **agile inkremental**.

### 11.2 Anti-pola fact-to-fact join

Men-join **dua fact table berbeda kardinalitas** ke satu customer dim bersamaan (many-to-one-to-many) → **jawaban SALAH**, walau DB bekerja sempurna. Contoh: `Solicitation Fact` vs `Response Fact` (tak semua solicitation berujung response, & ada response tanpa solicitation).
```
Solicitation Fact ──► Customer Dim ◄── Response Fact
   (satu SELECT gabungan = SALAH karena kardinalitas beda)
```
⚠️ Tak ada kombinasi inner/outer/left/right yang benar. **Solusi:** drill-across — query tiap fakta **terpisah**, lalu **outer-join** answer set (juga bagus untuk kontrol performa & fakta di lokasi fisik berbeda). Bila sangat sering → **consolidated fact table** (dengan aturan bisnis kardinalitas: sertakan semua, atau hanya yang punya solicitation **dan** response?).

### 11.3 Low latency reality check

| Latency | Jaminan | Kualitas |
|---|---|---|
| Batch 24 jam | Set transaksi **lengkap** (lolos credit check, komitmen final) | Cek kualitas **penuh** |
| Beberapa kali/hari | Set mungkin **tak lengkap**; key mungkin belum resolve (pakai entri dimensi sementara) | Cek **parsial** |
| Real-time instan | Hanya **fragmen** transaksi | Nyaris **tanpa** cek |

Makin cepat → makin banyak masalah kualitas. Beri tahu pengguna trade-off. **Hybrid:** low-latency intraday + koreksi batch malam (perbaiki masalah yang tak sempat ditangani siang).

---

## Ringkasan Teknis

Customer dimension menuntut **parsing elemental** (pipeline ETL parse→standardize→verify) dan **Unicode** (masalah character set, di lapisan fondasi). **Customer-centric dates** = outrigger role-playing; **aggregated facts sebagai atribut** membebani ETL (pilih yang minim update, label deskriptif). **Behavior tag** RFI/RFM disimpan **posisional** (kolom per periode + konkatenasi wildcard, bitmap index) — jangan sebagai fakta. **Count** dengan Type 2 pakai `COUNT DISTINCT durable_id` + effective/expiration. **Outrigger** = pengecualian snowflake (grain beda, load beda, hemat ruang). **Bridge table** untuk multivalued/sparse (name-value pair via text string; multiple contact via role) — skalabel tapi query kompleks (sembunyikan di view). **Behavior study group** = tangkap **durable key** ke tabel satu-kolom (kebal Type 2, mendukung set operation). **Step dimension** abstrak multi-peran untuk sekuensial. **Timespan fact table** = dual date/time stamp tanpa celah (query `>= begin AND < end`; ETL 2-langkah, end = begin berikutnya, hindari NULL). Ditutup: **partial conformity** (tanam atribut conformed inkremental), **anti-pola fact-to-fact** (drill-across, bukan satu SELECT; kardinalitas beda → jawaban salah), dan **low latency** (kecepatan ↔ kualitas berbanding terbalik; hybrid intraday+batch malam).
