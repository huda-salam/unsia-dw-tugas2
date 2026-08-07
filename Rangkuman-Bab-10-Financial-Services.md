# Bab 10 — Financial Services (Jasa Keuangan)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Studi kasus bank ritel yang memperkenalkan beberapa teknik penting: **dimension triage** (menghindari terlalu sedikit dimensi), penanganan relasi kompleks **account–customer–household**, **multivalued dimension dengan weighting factor**, **multiple mini-dimension** dalam satu fact table, **dynamic value banding** fakta, dan yang paling khas — **skema supertype/subtype** untuk **produk heterogen** serta **hot swappable dimension**.

---

## 1. Studi Kasus Bank & Dimension Triage

Bank ingin menganalisis akun sekaligus **household** (unit ekonomi), untuk cross-sell/up-sell. Kebutuhan: 5 tahun snapshot bulanan tiap akun; membandingkan saldo lintas jenis akun; tiap jenis akun (produk) punya atribut & fakta khusus; tiap akun milik satu household (relasinya volatil); plus demografi & skor perilaku.

**Grain awal:** satu baris per akun per bulan → tergoda pakai hanya 2 dimensi (month, account) dengan segala hal lain sebagai atribut account. **Ini kesalahan** — melanggar cara pikir bisnis natural.

**Dimension triage** — kebanyakan model punya **5–20 dimensi**. Bila di bawah batas bawah, curigai ada dimensi yang tertinggal. Pertimbangkan menambah: **causal** (promotion/deal), **multiple date** (accumulating snapshot), **degenerate** (control number), **role-playing**, **status** (mis. account status), **audit**, **junk**. Dimensi-dimensi ini bisa ditambahkan **anggun** bahkan setelah produksi (tak mengubah grain).
> Atribut deskriptif apa pun yang **bernilai tunggal** di hadapan fakta = kandidat dimensi (baru atau tambahan).

Hasil: 7 dimensi — month end date, branch, account, primary customer, product, account status, household. Fakta: primary balance (semi-aditif seperti stok — rata-rata lintas waktu), transaction count, interest paid, fees charged.

**Mengapa product & branch dipisah dari account?** (1) Mencerminkan cara pikir bisnis; (2) titik masuk lebih kecil ke fakta (performa/usability); (3) mengurangi ledakan baris account akibat SCD Type 2. Product & branch = dua dimensi terpisah (relasi many-to-many, berubah beda irama). **Account status** = semacam mini-dimension.

---

## 2. Household Dimension

Household = unit ekonomi (mis. John & Mary Smith dengan 5 akun berbeda dianggap satu household). **Householding** (mengaitkan akun ke household) tidak sepele — butuh aturan bisnis & algoritma matching; sering pakai produk/layanan khusus.

**Account & household dipisah** karena: ukuran account dimension raksasa (>10 juta baris), dan relasi account↔household **volatil**. Relasinya ditangkap **di fact table** (bukan atribut di tiap baris account) → menghindari Type 2 pada dimensi 10-juta-baris. Household jadi titik masuk lebih kecil.

---

## 3. Multivalued Dimension & Weighting Factor

Satu akun bisa punya beberapa pemegang (customer). Customer **tak bisa** jadi atribut account (langgar granularitas dimensi) maupun dimensi fact table (langgar grain "satu baris per akun per bulan"). → **account-to-customer bridge table** (account key, customer key, **weighting factor**).

- Bila akun punya 2 pemegang → 2 baris bridge, tiap **weighting factor** dijumlahkan = **1.00**. Dipakai untuk **mengalokasikan** fakta aditif ke tiap pemegang → **correctly weighted report** (grand total tetap benar).
- **Impact report** — bila weighting factor **tidak** dipakai: bisa **overcounting** (fakta dikaitkan ke kedua pemegang). Pengguna paham risikonya. Contoh: "total saldo semua individu dengan profil demografi X".
- Sediakan **dua view** (dengan & tanpa weighting) agar tampak seperti fact table biasa berkolom customer.
- Bila customer teridentifikasi per transaksi (mis. nomor kartu kredit unik tiap pemegang) → tak perlu bridge di grain transaksi (account & customer keduanya FK); bridge tetap perlu untuk metrik level-akun (mis. tagihan).

---

## 4. Multiple Mini-Dimension

Banyak atribut akun/customer/household yang **cepat berubah** (atribut biro kredit bulanan, demografi eksternal, skor risk/retention/profitability/delinquency). Type 2 tak realistis untuk account dimension raksasa. → pecah jadi **beberapa mini-dimension** (Type 4), mis. Demographics mini-dim & Risk Profile mini-dim, key-nya masuk fact table.

- Cocok untuk jasa keuangan karena fakta utamanya **periodic snapshot berjalan lama** — tiap bulan ada baris untuk tiap akun → "rumah" bagi semua FK mini-dimension.
- ⚠️ **Jangan berlebihan** — mini-dimension harus **gumpalan atribut berkorelasi**, jangan tiap atribut jadi mini-dimension sendiri.
- **Banding** (kompromi) — nilai kontinu di-band (income → `$30.000–34.999`; profitability score → rentang tetap). Untuk data mining/power analyst yang butuh nilai diskret, simpan juga **nilai mentah sebagai fakta** di snapshot; opsional nilai kini di account dimension (Type 1 overwrite). Beri label unik agar terbedakan.

**Menambah mini-dimension ke bridge table** — bila account-to-customer bridge membengkak (Type 2 pada account & customer) dan customer jadi "monster dimension", tambahkan **FK demographics** ke **bridge** (key bridge = account + customer + demographics). Batasi perubahan pada akhir bulan (sesuai grain fakta) untuk menahan pertumbuhan.

---

## 5. Dynamic Value Banding of Facts

Pengguna ingin laporan band atas **fakta** (mis. saldo) **tanpa** band pra-definisi. SQL tak punya `GROUP BY` untuk mengelompokkan nilai aditif ke rentang, apalagi rentang tak sama besar bernama teks ("10001 and up") yang bisa didefinisi ulang saat query.

**Solusi:** **band definition table** (band group name, band range name, lower/upper value, sort order) yang di-join ke fakta lewat **join `<` dan `>`**. Bisa memuat banyak set band. Laporan pakai band range name sebagai row header.
> ⚠️ **Performa berat** — query band nyaris tanpa constraint (scan jutaan saldo). Perlu **indeks langsung pada fakta saldo**; **columnar database** (dipelopori Sybase IQ) sangat membantu (sort & kompresi fakta).

---

## 6. Supertype/Subtype untuk Produk Heterogen

Bank menawarkan produk sangat beragam (checking, kartu kredit, deposito) ke customer sama. Tiap produk punya atribut & fakta khusus yang **tak dibagi** produk lain (checking: minimum balance, overdraft; deposito: maturity date, compounding). Butuh **dua perspektif**:

- **Global (supertype)** — slice semua akun serentak (untuk cross-sell/up-sell). **Satu supertype fact table** + supertype product dimension, tapi **hanya memuat fakta/atribut yang umum** ke semua lini (tak bisa menampung ratusan fakta inkompatibel).
- **Line-of-business (subtype)** — detail mendalam satu lini (mis. checking). **Subtype fact & dimension** khusus, memuat semua fakta/atribut khas + **juga** fakta supertype (agar tak perlu join supertype↔subtype). Menghindari tabel "Swiss cheese" penuh null.

**Kunci:** subtype account dimension memakai **surrogate key yang sama** dengan supertype (satu akun = key sama di keduanya). Tiap subtype = **shrunken conformed dimension** (subset baris supertype + atribut spesifik). Subtype saling **disjoint** (tak tumpang tindih). Dari sisi pengguna: analisis lintas-produk pakai supertype; analisis satu jenis pakai satu subtype. Hanya DBA yang melihat semua tabel.

> Diperlukan **keluarga supertype/subtype fact table** bila bisnis punya **produk heterogen** (fakta/deskriptor berbeda alami) tapi **satu basis customer** yang menuntut pandangan terintegrasi. Berlaku umum (mis. perusahaan teknologi: hardware/software/services).

**Supertype/subtype dengan common facts** — bila metrik proses **tidak** bervariasi per lini (mis. solicitation akun baru), cukup **satu supertype fact table**; tetapi tetap bisa punya portofolio **subtype account dimension** yang dipakai sesuai kebutuhan.

---

## 7. Hot Swappable Dimension

Rumah broker punya banyak klien yang mengakses **fact table harga saham yang sama** (high-low-close harian), tapi tiap klien punya **set atribut rahasia** sendiri untuk tiap saham. Solusi: **salinan stock dimension terpisah per klien**, di-join ke fact table tunggal **saat query**. Implementasi: constraint referential integrity antara fakta & berbagai stock dimension biasanya harus **dimatikan** agar penukaran bisa terjadi per-query.

---

## Ringkasan Bab

Bab 10 (bank ritel) mengajarkan **dimension triage** (5–20 dimensi; jangan terlalu sedikit), pemisahan **account/household** (relasi volatil ditangkap di fakta), **multivalued dimension** account-to-customer dengan **weighting factor** (correctly weighted vs impact report), **multiple mini-dimension** Type 4 (demographics/risk, dengan banding + nilai mentah sebagai fakta), dan **penambahan mini-dimension ke bridge table**. Diperkenalkan **dynamic value banding** fakta (band definition table + join `<`/`>`, butuh columnar/indeks fakta), dan kontribusi utama: **skema supertype/subtype** untuk **produk heterogen** (supertype = pandangan global fakta umum; subtype = detail per lini, shrunken conformed, key sama, disjoint) plus **hot swappable dimension**. Bab 11 (Telekomunikasi) akan membahas pendekatan desain dari sudut pandang laporan & general design review.
