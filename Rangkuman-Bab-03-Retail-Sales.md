# Bab 3 — Retail Sales (Penjualan Ritel)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Studi kasus pertama, sekaligus **contoh paling penting** di buku ini. Industri ritel dipilih karena akrab bagi semua orang, tetapi polanya berlaku untuk hampir semua model dimensional di industri apa pun. Bab ini mendemonstrasikan **proses desain 4 langkah** secara konkret dan memperkenalkan banyak teknik dasar: jenis fakta (aditif/turunan/non-aditif), dimensi tanggal, dimensi kausal (promosi), degenerate dimension, factless fact table, dan surrogate key.

---

## 1. Proses Desain 4 Langkah (diterapkan)

**Langkah 1 — Pilih Proses Bisnis.** Proses bisnis = aktivitas operasional (kata kerja) yang didukung sistem operasional dan menghasilkan metrik kinerja. Contoh: menerima order, menagih, memproses klaim. **Bukan** departemen/fungsi organisasi. Dengarkan bisnis untuk mengidentifikasinya (pengguna biasanya bicara "inisiatif strategis" — Anda harus menguraikannya jadi proses konkret). Publikasikan data **sekali** agar konsisten.

**Langkah 2 — Deklarasikan Grain.** Nyatakan **apa arti satu baris fact table** dalam istilah bisnis (mis. "satu baris per scan produk pada satu transaksi penjualan"). Ini **langkah paling krusial** — kesalahan paling sering dalam ribuan desain adalah tidak mendeklarasikan grain di awal. Tanpa grain jelas, "desain berdiri di atas pasir hisap". Boleh mulai dari grain atomik; jika di langkah 3–4 ternyata grain salah, kembali ke langkah 2.

**Langkah 3 — Identifikasi Dimensi.** Jawaban atas "bagaimana orang bisnis mendeskripsikan data ini?" — "siapa, apa, di mana, kapan, mengapa, bagaimana". Contoh umum: date, product, customer, employee, facility.

**Langkah 4 — Identifikasi Fakta.** Jawaban atas "apa yang diukur proses ini?" Semua fakta harus konsisten dengan grain; fakta beda grain → fact table terpisah. Umumnya numerik aditif.

> Kunci: pertimbangkan **kebutuhan bisnis** dan **realita data sumber** bersamaan. Jangan memodelkan hanya dari data sumber saja (pendekatan "jalan termudah" yang sering gagal).

---

## 2. Studi Kasus: Allstar Grocery

Jaringan grosir 100 toko di 5 negara bagian; tiap toko ~60.000 SKU. Data paling berharga dikumpulkan di **kasir (POS)** yang men-scan barcode produk saat pelanggan membeli. Manajemen fokus pada logistik order/stok/jual sambil memaksimalkan profit; keputusan terpenting seputar **harga & promosi** (potongan harga sementara, iklan, display, kupon).

**Desain 4 langkah untuk kasus ini:**
- **Proses bisnis:** penjualan ritel di POS.
- **Grain:** satu baris per produk yang di-scan pada satu transaksi POS (grain paling atomik).
- **Dimensi:** Date, Product, Store, Promotion, Cashier, Payment Method (+ degenerate: nomor transaksi POS).
- **Fakta:** Sales Quantity, Regular/Discount/Net Unit Price, Extended Discount/Sales/Cost/Gross Profit Dollar Amount.

---

## 3. Jenis Fakta: Aditif, Turunan, Non-Aditif

- **Aditif** — bisa dijumlahkan di **semua** dimensi: sales quantity, extended discount/sales/cost dollar amount. Ini yang paling berharga.
- **Turunan (derived)** — dihitung tapi tetap aditif: **gross profit** = extended sales − extended cost. **Rekomendasi: simpan secara fisik** hasil hitungnya, agar dihitung konsisten di ETL (sekali) dan menghindari kesalahan hitung pengguna; biaya penyimpanan kecil dibanding risiko salah. Alternatif: hitung di *view* (asalkan semua akses lewat view) atau di tool BI (asal semua pakai tool yang sama).
- **Non-aditif** — tidak bisa dijumlahkan di dimensi apa pun:
  - **Gross margin** (= gross profit ÷ revenue) dan rasio/persentase lain → simpan **pembilang & penyebut** di fact table, lalu hitung **rasio dari jumlah, bukan jumlah dari rasio** di tool BI.
  - **Unit price** → menjumlahkan harga satuan menghasilkan angka tak bermakna. Rata-rata tertimbang = total dollar ÷ total quantity.

---

## 4. Transaction Fact Table

Proses transaksional paling umum. Ciri-ciri:
- Grain diekspresikan ringkas: "satu baris per transaksi" atau "per baris transaksi".
- **Sparse** (jarang) — tidak semua produk terjual di tiap keranjang.
- Bisa **sangat besar** — mayoritas tabel bermiliar/triliun baris adalah transaction fact table.
- **Sangat dimensional** (banyak dimensi).
- Metriknya umumnya aditif.

---

## 5. Detail Dimensi Kunci

**Date Dimension** — dimensi khusus yang ada di hampir setiap fact table. Dibuat sekali, diisi di muka.
- Punya **banyak atribut**: full date description, day of week, nomor hari/minggu/bulan/kuartal, nama bulan, tahun, plus **hierarki kalender & fiskal** paralel.
- **Flag & indikator sebagai teks** — ganti nilai kriptik dengan label deskriptif (mis. "Holiday"/"Non-Holiday", "Weekday"/"Weekend") agar langsung terbaca & bisa dipakai untuk grup/filter di laporan.
- **Atribut tanggal kini & relatif** (mis. "IsCurrentMonth") — untuk memudahkan laporan bergulir; di-update berkala oleh ETL.
- **Time-of-day** ditangani terpisah — sebagai **fakta** (mis. detik sejak tengah malam) atau **dimensi jam-menit** tersendiri, **jangan** digabung ke dimensi tanggal (akan meledakkan jumlah baris).

**Product Dimension** — contoh dimensi lebar & kaya atribut.
- **Ratakan hierarki many-to-one** (product → brand → subcategory → category → department) menjadi kolom-kolom terpisah dalam satu tabel denormalized — **bukan** snowflake.
- **Drilling down** = sekadar menambah atribut header (mis. tambah kolom "brand") ke query; tidak butuh struktur khusus.
- **Numeric value sebagai atribut atau fakta** — mis. berat/ukuran bisa jadi atribut (untuk filter/grup) atau fakta (untuk agregasi), tergantung pemakaian.
- **Multiple hierarchies** boleh ada dalam satu dimensi (mis. hierarki merchandising & hierarki lain).

**Store Dimension** — memuat **multiple hierarchies** (geografis: city → county → state → region; dan organisasi: district → region) diratakan berdampingan. Bisa memuat atribut tanggal (mis. first open date, last remodel date).

**Promotion Dimension** — contoh **dimensi kausal** ("mengapa" produk terjual): jenis potongan harga, media promosi (iklan/display/kupon), biaya promosi, tanggal mulai/selesai. Menghubungkan penjualan dengan sebab (kondisi promosi).

**Payment Method Dimension** — cara pembayaran (tunai/kartu/dll.), dengan pengelompokan.

---

## 6. Null & Degenerate Dimension

- **Null:** FK **tidak boleh null** (pakai baris dimensi khusus seperti "No Promotion"/"Unknown"). Fakta null boleh (agregasi SQL mengabaikannya). Atribut dimensi null sebaiknya diganti label deskriptif ("Not Applicable").
- **Degenerate Dimension (DD):** **nomor transaksi POS** disimpan langsung di fact table tanpa tabel dimensi tersendiri (tidak punya atribut deskriptif lain). Berguna untuk mengelompokkan baris dalam satu transaksi (mis. analisis *market basket*).

---

## 7. Factless Fact Table: "Apa yang Tidak Terjual?"

Skema penjualan hanya mencatat produk yang **terjual**, jadi tak bisa menjawab: *produk apa yang sedang promosi tapi tidak terjual?* Solusinya **promotion coverage factless fact table**:
- Grain berbeda: **satu baris per produk yang dipromosikan per toko per hari**, terlepas terjual atau tidak.
- Tidak punya fakta terukur — hanya menangkap **relasi antar key** (date, product, store, promotion). Boleh tambah *dummy fact* `Promotion Count = 1` untuk memudahkan penghitungan.
- **Jawabannya** = selisih himpunan: (produk yang dipromosikan) − (produk yang terjual). Di OLAP cube ini lebih mudah karena cube punya sel eksplisit untuk "non-perilaku".

---

## 8. Kunci: Surrogate, Natural, Durable

**Surrogate key** — PK dimensi harus berupa **integer buatan** yang di-assign berurutan (1, 2, 3, …), **bukan** natural key dari sistem operasional. Semua join fact↔dimensi memakai surrogate key. Keuntungan:
- **Melindungi DW dari perubahan operasional** — kode operasional bisa didaur ulang/dipakai ulang; surrogate key membedakan dua instans kode yang sama.
- **Mengintegrasikan banyak sumber** — via tabel cross-reference (natural key → surrogate).
- **Meningkatkan performa** — integer 4-byte kecil (menampung ~2 miliar nilai) → fact table & indeks lebih kecil, lebih banyak baris per operasi I/O.
- **Menangani kondisi null/unknown** — nilai surrogate khusus untuk "No Promotion", pelanggan anonim, "Date TBD", dll.
- **Mendukung pelacakan perubahan atribut (SCD)** — satu natural key bisa punya banyak baris/profil (dibahas Bab 5). Hindari "pseudo surrogate" (natural key + timestamp) yang memicu *double-barreled join*.

**Natural key (NK)** — kode dari sistem sumber, disimpan sebagai **atribut** di dimensi (bukan PK). Dari banyak sumber bisa diberi prefiks (mis. `SAP|43251`).

**Durable supernatural key** — pengenal stabil yang tak pernah berubah sepanjang hidup entitas (penting saat SCD Type 2 membuat banyak baris untuk satu entitas).

**Date dimension smart key** — pengecualian: date dimension boleh pakai **kunci bermakna** berformat `yyyymmdd` (memudahkan partisi & query rentang tanggal).

**Fact table surrogate key** — fact table sendiri boleh punya surrogate key (memudahkan operasi ETL seperti update/rollback baris tunggal).

---

## 9. Snowflake vs Flattened & Centipede Fact Table

- **Hindari snowflake** — jangan menormalisasi hierarki dimensi jadi tabel-tabel terpisah. Dimensi denormalized yang datar lebih sederhana & cepat.
- **Centipede fact table (anti-pola)** — kesalahan menaruh terlalu banyak FK di fact table (mis. FK terpisah untuk week, month, quarter, year, brand, category, department…) sehingga fact table "berkaki seratus" dan join ke puluhan tabel. Akibatnya: fact table (tabel terbesar) membengkak, indeks tak efektif, performa & usability buruk.
- **Aturan praktis:** kebanyakan proses bisnis cukup **< 20 dimensi**. Bila ≥ 25, gabungkan dimensi yang berkorelasi (mis. level hierarki yang sama) menjadi satu. **Elemen dari satu hierarki tidak boleh jadi dimensi terpisah.**

---

## Ringkasan Bab

Bab 3 memperkenalkan pemodelan dimensional lewat kasus **penjualan ritel POS** dan menegaskan **proses 4 langkah** (proses bisnis → grain → dimensi → fakta), dengan penekanan kuat pada **deklarasi grain** dan **data atomik** demi fleksibilitas maksimum. Konsep baru yang diperkenalkan: fakta **turunan** (gross profit, disimpan) & **non-aditif** (gross margin, unit price), **dimensi kausal** (promotion), **degenerate dimension** (nomor transaksi), **factless fact table** (menjawab "apa yang tidak terjadi"), dan pentingnya **surrogate key**. Bab juga memperingatkan anti-pola **centipede fact table** & snowflake. Bab 4 tetap di industri ritel — membahas proses bisnis **kedua** (Inventory) sambil memastikan usaha sebelumnya dapat dipakai ulang (menghindari *stovepipe*).
