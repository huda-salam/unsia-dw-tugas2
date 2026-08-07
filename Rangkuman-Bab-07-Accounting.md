# Bab 7 — Accounting (Akuntansi)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab ini memodelkan data keuangan **general ledger (G/L)**, yang membutuhkan **dua skema komplementer**: periodic snapshot (saldo akhir periode) dan transaction (jurnal). Konsep baru yang penting: **chart of accounts** sebagai conformed dimension, **grain "net change"** untuk budget, dan — yang paling teknis — teknik menangani **hierarki** (fixed depth, slightly ragged, dan ragged variable depth dengan **bridge table**). Ditutup dengan **consolidated fact table** dan peran OLAP.

---

## 1. General Ledger Periodic Snapshot

General ledger adalah sistem keuangan inti yang mengikat subledger (purchasing, payables, receivables). **Proses bisnis = general ledger**; **grain = satu baris per periode akuntansi untuk level terendah chart of accounts**. Fakta: period end balance, period debit, period credit, period net change.

- **Chart of accounts** adalah tulang punggung G/L dan contoh klasik *intelligent key* (mis. akun 1000–1999 = aset, 2000–2999 = liabilitas). Di DW, **jadikan account type sebagai atribut dimensi**, bukan memaksa pengguna memfilter digit pertama. Terurai jadi dua dimensi: **Account** dan **Organization** (rollup cost center → department → division → business unit).
- **Uniform chart of accounts** = versi *conformed* dari account dimension — nama akun bermakna sama lintas organisasi (mis. "Capital Expenditures" dan "Office Supplies" identik maknanya).
- **Balance = fakta semi-aditif** (tak bisa dijumlahkan lintas waktu). Tetap disimpan karena berguna — kalau tidak, harus menghitung ulang dari awal waktu.
- ⚠️ **Ledger dimension** memungkinkan banyak ledger dalam satu fact table, tapi **setiap query wajib membatasi ledger ke satu nilai** (mis. "Final Approved Domestic Ledger") agar tidak *double count*. Cara aman: rilis **view** dengan ledger sudah di-*pre-constrain*.
- **"G/L tidak nyambung dengan laporan operasional saya"** — frustrasi khas pengguna. Dimensi account & organization hampir tak pernah *conform* dengan dimensi operasional (customer, product). Jelaskan ini sejak awal wawancara, jangan janjikan perbaikan.

**Year-to-date facts:** ⚠️ **jangan simpan** "to-date" (QTD/YTD) di fact table — tidak konsisten dengan grain, menghasilkan hasil ngawur saat di-agregasi arbitrer. **Hitung di aplikasi BI.** (OLAP cube menangani ini lebih anggun.)

**Multiple currency:** seperti Bab 6 — simpan fakta dalam mata uang lokal **dan** korporat standar, plus currency dimension.

---

## 2. General Ledger Journal Transactions

Melengkapi snapshot untuk menggali detail. **Grain = satu baris per jurnal entry.** Menggunakan ulang dimensi Account, Organization, Ledger; ditambah **Debit-Credit Indicator dimension** (hanya dua nilai), **Post Date** (grain harian), dan **Journal Entry Number sebagai degenerate dimension**.

- Bila nomor jurnal terurut, DD ini bisa dipakai mengurutkan entry (dimensi tanggal terlalu kasar). Jika tidak, tambahkan *effective date/time stamp*.
- Bila ada tipe/deskripsi transaksi, buat **journal entry transaction profile dimension** terpisah (baris jauh lebih sedikit dari fakta); nomor jurnal tetap degenerate.

**Multiple fiscal accounting calendars:** periode fiskal sering tak sejajar dengan bulan Masehi (mis. 13 periode 4-mingguan mulai 1 September).
- **Satu kalender fiskal** → kalender & periode akuntansi cukup jadi **atribut hierarkis** di date dimension harian.
- **Sedikit kalender berbeda** (per subsidiary) → sertakan tiap set atribut fiskal berlabel unik di **satu** date dimension.
- **Banyak kalender** → beberapa opsi: **date dimension outrigger** (key = date + subsidiary), date dimension fisik terpisah per subsidiary (surrogate key bersama), atau FK ke subsidiary fiscal period dimension (membebani ETL).

---

## 3. Drilling Down & Financial Statements

**Multilevel ledger hierarchy** (enterprise → division → department): modelkan dengan menaruh **parent snapshot fact surrogate key** di fact table (relasi parent/child antar baris fakta). Untuk menggali: ambil surrogate key entry level atas, lalu ambil semua entry yang parent-nya = key tersebut.

**Financial statements** (neraca, laba-rugi): **jangan** menggantikan laporan sistem operasional. DW/BI bisa membuat **data teragregasi komplementer** yang diberi tag nomor & label baris laporan keuangan, untuk penyebaran tren lebih luas.

---

## 4. Budgeting Process — Grain "Net Change"

Rantai budget: **budget → commitment → payment** (tiga fact table dengan aliran logis; daftar dimensi bertambah sepanjang rantai).

- **Grain = net change** dari budget line item per cost center per bulan — **bukan** snapshot status bulanan. Alasan menghindari grain snapshot status: faktanya jadi saldo semi-aditif, sulit menghitung perubahan antar-periode, dan menimbulkan banyak baris duplikat saat tak ada perubahan.
- Dengan net change: menjumlahkan semua entry dari awal waktu = budget kini yang benar; membatasi satu bulan = perubahan bulan itu. Perbandingan budget vs commitment vs payment = *drill-across* via multipass SQL.
- Bila satu budget line memengaruhi >1 akun G/L → **alokasikan** ke tiap akun (satu budget line bisa jadi beberapa baris fakta).

---

## 5. Hierarki Atribut Dimensi

Hierarki = rangkaian relasi many-to-one. Tiga jenis penanganan:

**Fixed depth positional** — level tetap dengan **label bermakna** (mis. day → month → year). Beberapa hierarki paralel bisa hidup di satu dimensi (mis. kalender fiskal & Masehi). ⚠️ **Hindari label abstrak** (Level-1, Level-2) — itu cara murah menghindari pemodelan ragged hierarchy yang benar; pengguna jadi tak tahu makna tiap level.

**Slightly ragged / variable depth** — kedalaman sedikit bervariasi (mis. lokasi: sebagian punya zone/district, sebagian tidak). Kompromi: **propagasikan** nama level (mis. isi district dengan nama city bila tak ada district), atau "Not Applicable". Hanya berhasil bila rentang level **sempit** (mis. 4–6 level); untuk 4–10+ level, kompromi ini gagal.

**Ragged / variable depth (kedalaman tak tentu)** — mis. struktur organisasi. Solusi utama: **bridge table** (lihat versi teknis untuk detail). Bridge table jauh lebih fleksibel daripada recursive pointer/pathstring.

---

## 6. Consolidated Fact Table

Bila *drill-across* antar proses (mis. actual vs budget) **sangat sering** dilakukan, buat **consolidated fact table** yang menggabungkan metrik sekali (mis. Budget Variance Fact: actual amount, budget amount, variance).

- ⚠️ Fakta dari beberapa proses hanya bisa digabung bila **granularitas & dimensionalitas sama**. Karena jarang alami sama grain-nya, sebagian dimensi harus diagregasi/dihilangkan → consolidated fact adalah kompromi *"least common denominator"*. **Data atomik tetap disimpan di fact table terpisah.** Jangan memaksa membuat fakta/dimensi artifisial demi konsolidasi.

**OLAP & paket analitik:** OLAP sangat cocok untuk keuangan (rollup organisasi kompleks, fungsi finansial seperti NPV, penanganan debit/kredit, keamanan berlapis). Paket analitik siap-pakai bisa mempercepat implementasi, tapi **tetap wajib meng-conform dimensi** agar tak jadi stovepipe.

---

## Ringkasan Bab

Bab 7 memodelkan **general ledger** dengan **periodic snapshot** (grain per periode per akun; balance semi-aditif; ledger wajib di-constrain tunggal) dan **journal transaction** (grain per entry; debit/credit indicator). Ditangani pula tantangan G/L: **multiple currency**, **multiple fiscal calendar**, **year-to-date** (hitung, jangan simpan), dan **drilling** lewat multilevel ledger. Proses **budgeting** memakai grain **net change** (bukan snapshot status). Kontribusi teknis terbesar: penanganan **hierarki** — fixed depth, slightly ragged, dan **ragged variable depth dengan bridge table**. Ditutup dengan **consolidated fact table** (kompromi least-common-denominator) dan peran OLAP. Bab 8 (CRM) akan fokus pada dimensi customer yang kompleks.
