# Bab 2 — Ikhtisar Teknik Pemodelan Dimensional Kimball

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab ini adalah **katalog/kamus resmi seluruh teknik pemodelan dimensional** yang dipakai di sepanjang buku. Setiap teknik diberi definisi ringkas plus rujukan silang ke bab tempat ia dibahas mendalam. Fungsinya sebagai **referensi cepat** — Anda bisa kembali ke sini kapan pun bertemu istilah teknik di bab-bab studi kasus. Teknik dikelompokkan menjadi: konsep fundamental, teknik fact table (dasar & lanjutan), teknik dimension table (dasar & lanjutan), integrasi via conformed dimension, penanganan SCD, penanganan hierarki, dan skema khusus.

---

## 1. Konsep Fundamental

Teknik di bagian ini **wajib dipertimbangkan di setiap desain dimensional**.

- **Kumpulkan kebutuhan bisnis & realita data** — pahami tujuan bisnis (KPI, isu, proses keputusan) lewat sesi dengan perwakilan bisnis, sekaligus telaah realita data sumber lewat *data profiling*.
- **Workshop pemodelan kolaboratif** — model dirancang bersama *subject matter expert* & perwakilan data governance. Data modeler memimpin, tapi model berkembang lewat serangkaian workshop interaktif. **Jangan merancang model dalam isolasi.**

### Proses Desain 4 Langkah (inti bab, dipakai di semua studi kasus)

1. **Pilih proses bisnis** — aktivitas operasional (mis. menerima order, memproses klaim). Setiap proses = satu baris di *bus matrix*.
2. **Deklarasikan grain** — tetapkan **apa arti satu baris fact table**. Ini **kontrak mengikat**; harus dinyatakan **sebelum** memilih dimensi/fakta karena semua kandidat harus konsisten dengan grain. Mulailah dari **grain atomik** (level terendah) agar tahan terhadap query tak terduga. Grain berbeda = tabel fisik berbeda; jangan dicampur.
3. **Identifikasi dimensi** — konteks "siapa, apa, di mana, kapan, mengapa, bagaimana". Sebisa mungkin bernilai tunggal untuk satu baris fakta.
4. **Identifikasi fakta** — pengukuran (hampir selalu numerik) yang konsisten dengan grain. Contoh valid: kuantitas & harga produk terjual; tidak valid: gaji manajer toko.

### Perluasan Model yang "Anggun" (Graceful Extensions)

Model dimensional tahan perubahan — hal berikut bisa dilakukan **tanpa mengubah query/aplikasi BI yang sudah ada**: menambah **fakta baru** (kolom baru sesuai grain), menambah **dimensi baru** (FK baru tanpa mengubah grain), menambah **atribut** ke dimensi, dan **menurunkan grain** jadi lebih atomik (dengan menyatakan ulang fact table di grain lebih rendah).

---

## 2. Teknik Fact Table Dasar

**Struktur fact table:** berisi FK ke tiap dimensi + fakta numerik sesuai grain; fakta yang tidak sesuai grain dilarang.

- **Aditif / semi-aditif / non-aditif** — aditif bisa dijumlahkan di semua dimensi; semi-aditif tidak bisa lintas waktu (mis. saldo); non-aditif tak pernah bisa dijumlahkan (mis. harga satuan, rasio).
- **Null di fact table** — fakta null boleh (agregasi SQL mengabaikannya), tapi **FK tidak boleh null** (pakai baris dimensi "Tidak Diketahui").
- **Conformed facts** — jika fakta yang sama muncul di beberapa fact table, definisi & rumusnya harus identik agar bisa dibandingkan.

**Tiga jenis grain fact table (fundamental):**

| Jenis | Satu baris mewakili | Karakter |
|---|---|---|
| **Transaction** | satu event pengukuran pada titik ruang & waktu | Paling detail & ekspresif; padat/jarang; hanya ada jika transaksi terjadi |
| **Periodic Snapshot** | rangkuman banyak event selama periode standar (hari/minggu/bulan) | Padat di FK (selalu ada baris tiap periode meski nol); banyak fakta |
| **Accumulating Snapshot** | satu proses/pipeline dari awal–akhir (mis. pemenuhan order) | Baris **di-update berulang** saat milestone tercapai; punya banyak FK tanggal per milestone + fakta *lag* |

**Jenis fact table lain:**
- **Factless fact table** — event yang tak punya fakta numerik (mis. kehadiran siswa); hanya kumpulan FK. Berguna menganalisis "apa yang **tidak** terjadi" (coverage − activity).
- **Aggregate fact table / OLAP cube** — rollup numerik data atomik semata-mata untuk **percepatan query** (seperti indeks); dipakai lewat *aggregate navigation* yang transparan.
- **Consolidated fact table** — menggabungkan fakta dari beberapa proses pada grain sama (mis. aktual vs forecast) agar analisis lintas-proses cepat; menambah beban ETL tapi meringankan BI.

---

## 3. Teknik Dimension Table Dasar

Dimensi disebut **"jiwa" data warehouse** — sumber label untuk filter & grouping; upaya governance terbesar ada di sini.

- **Struktur** — satu primary key; tabel **lebar, datar, denormalized**, banyak atribut teks berkardinalitas rendah.
- **Surrogate key** — kunci integer buatan (bukan natural key sumber) sebagai PK dimensi; wajib untuk fleksibilitas & SCD.
- **Natural, durable, & supernatural key** — natural key dari sumber; **durable key** stabil sepanjang hidup entitas (tak berubah saat SCD Type 2); *supernatural/durable* key untuk melacak identitas.
- **Drilling down** — menambah atribut header baris ke query = menggali detail; inti analisis BI.
- **Degenerate dimension** — "dimensi" berupa pengenal transaksi (mis. nomor order/invoice) yang disimpan **di fact table** tanpa tabel dimensi.
- **Denormalized flattened dimension** — hierarki & atribut diratakan ke satu tabel (bukan snowflake) demi kesederhanaan & performa.
- **Multiple hierarchies** — satu dimensi bisa memuat beberapa hierarki (mis. kalender fiskal & kalender masehi).
- **Flags & indicator sebagai teks** — ganti kode kriptik (Y/N, 1/0) dengan label deskriptif yang bisa dibaca di laporan.
- **Calendar date dimension** — dimensi khusus tanggal dengan banyak atribut (hari kerja, bulan, kuartal, hari libur); ada di hampir setiap fact table.
- **Role-playing dimension** — satu dimensi fisik dipakai berkali-kali dengan peran berbeda via *view/alias* (mis. Date sebagai order date, ship date, due date).
- **Junk dimension** — menggabungkan aneka flag/indikator berkardinalitas rendah jadi satu dimensi agar FK di fact table tidak membengkak.
- **Outrigger dimension** — dimensi yang direferensikan oleh dimensi lain (dipakai hemat & hati-hati).

---

## 4. Integrasi via Conformed Dimension

Ini kunci membangun DW terintegrasi berskala enterprise.

- **Conformed dimension** — dimensi (mis. Date, Product, Customer) yang **dibangun & dipelihara sekali** sebagai master data, lalu **dipakai ulang** lintas fact table → memungkinkan integrasi & konsistensi semantik.
- **Shrunken dimension** — versi *conformed* yang lebih kecil (subset baris/atribut), mis. untuk agregat bulanan atau rollup wilayah.
- **Drilling across** — menggabungkan fakta dari beberapa fact table dengan menyatukannya lewat conformed dimension yang sama (bukan join fact-to-fact).
- **Value chain** — rangkaian proses bisnis berurutan (mis. purchasing → inventory → sales); tiap tahap jadi baris di bus matrix.
- **Enterprise Data Warehouse Bus Architecture** — kerangka yang merekatkan seluruh model lewat conformed dimension bersama; fondasi DW terdistribusi & agile.
- **Bus Matrix** — matriks **proses bisnis (baris) × dimensi (kolom)**; alat perencanaan induk. Turunannya: *detailed implementation bus matrix* (lebih rinci per fact table) dan *opportunity/stakeholder matrix* (proses bisnis × grup pemangku kepentingan).

---

## 5. Menangani Slowly Changing Dimension (SCD)

Atribut dimensi bisa berubah perlahan seiring waktu. Beberapa teknik boleh dipakai bersamaan dalam satu dimensi.

| Type | Nama | Perlakuan | Efek terhadap histori |
|---|---|---|---|
| **0** | Retain Original | Nilai tidak pernah diubah (mis. skor kredit awal, atribut date) | Selalu nilai asli |
| **1** | Overwrite | Nilai lama **ditimpa** nilai baru | Histori **hilang**; agregat/cube harus dihitung ulang |
| **2** | Add New Row | Tambah **baris baru** dengan surrogate key baru + kolom *effective date*, *expiration date*, *current flag* | Histori terjaga akurat (profil kontemporer) |
| **3** | Add New Attribute | Tambah **kolom** untuk menyimpan nilai lama (mis. "kategori sebelumnya") | Menyimpan "realita alternatif"; dipakai jarang |
| **4** | Add Mini-Dimension | Atribut yang **cepat berubah** dipecah ke *mini-dimension* tersendiri (untuk *monster dimension*) | FK mini-dim & base-dim keduanya masuk fact table |
| **5** | Mini-Dim + Type 1 Outrigger | Type 4 + referensi Type 1 (nilai kini) ke mini-dim yang ditanam di base dimension | Histori terjaga **dan** bisa lapor pakai nilai kini |
| **6** | Type 1 attr di Type 2 dim | Type 2 + kolom Type 1 (nilai kini) yang ditimpa di semua baris durable key | Bisa filter/grup pakai nilai saat kejadian **atau** nilai kini |
| **7** | Dual Type 1 & Type 2 | Fact diakses lewat dimensi yang sama dalam dua sudut pandang (via durable key = *as-is*; via surrogate key = *as-was*), dipapar sebagai dua view | Mendukung pelaporan *as-was* & *as-is* sekaligus |

---

## 6. Menangani Hierarki Dimensi

- **Fixed depth positional hierarchy** — hierarki berkedalaman tetap (mis. produk → brand → kategori → departemen), diratakan jadi kolom-kolom terpisah.
- **Slightly ragged / variable depth** — kedalaman sedikit bervariasi; sering ditangani dengan mengisi level kosong.
- **Ragged / variable depth dengan pathstring attribute** — hierarki berkedalaman tak tentu (mis. bagan organisasi, bill of material) ditangani dengan atribut *path string* atau *bridge table* hierarki.

---

## 7. Teknik Fact Table Lanjutan

Ringkasan cepat (dibahas mendalam di bab studi kasus terkait):

- **Fact table surrogate key** — kunci buatan untuk baris fakta (mis. memudahkan update/rollback ETL).
- **Centipede fact table** — anti-pola: terlalu banyak dimensi "normalisasi" berkardinalitas rendah; sebaiknya digabung jadi junk dimension.
- **Numeric value sebagai atribut atau fakta** — nilai numerik bisa jadi fakta (diukur/di-agregasi) atau atribut dimensi (untuk filter/grup), tergantung penggunaan; kadang keduanya.
- **Lag / duration fact** — selisih waktu antar milestone (mis. lama pemenuhan order).
- **Header/line fact table** — pola order (header + baris); hindari menyimpan fakta header di grain baris tanpa alokasi.
- **Allocated fact** — mengalokasikan fakta level-header (mis. ongkos kirim) ke grain baris agar bisa dianalisis penuh.
- **Multiple currency fact** — simpan nilai dalam mata uang lokal **dan** mata uang standar.
- **Multiple unit of measure fact** — simpan fakta dasar + faktor konversi satuan, bukan banyak kolom terhitung.
- **Year-to-date fact** — hati-hati; umumnya YTD dihitung di BI, bukan disimpan.
- **Multipass SQL** — hindari join fact-to-fact; gunakan query terpisah lalu gabungkan hasil via conformed dimension.
- **Timespan tracking in fact table** — melacak rentang waktu berlaku suatu baris fakta.
- **Late arriving fact** — fakta yang datang terlambat; cocokkan ke versi dimensi yang berlaku saat event.

---

## 8. Teknik Dimensi Lanjutan

- **Dimension-to-dimension join** — dua dimensi berhubungan langsung (dipakai selektif).
- **Multivalued dimension & bridge table** — saat satu fakta berkaitan dengan banyak nilai dimensi (mis. satu pasien banyak diagnosis) → pakai *bridge table* (opsional dengan bobot alokasi).
- **Time-varying multivalued bridge** — bridge yang juga melacak perubahan sepanjang waktu.
- **Behavior tag time series** — deret tag perilaku pelanggan antar periode.
- **Behavior study group** — himpunan entitas hasil analisis, disimpan untuk query lanjutan.
- **Aggregated facts sebagai atribut dimensi** — mis. "total belanja pelanggan tahun lalu" ditaruh sebagai atribut untuk filter cepat.
- **Dynamic value band** — pengelompokan nilai dinamis (mis. band saldo) via tabel band.
- **Text comments dimension** — komentar bebas dipindah ke dimensi terpisah.
- **Multiple time zones** — simpan waktu lokal **dan** waktu standar/UTC.
- **Measure type dimension** — mengubah banyak fakta jadi pasangan (jenis ukuran, nilai) saat fakta sangat banyak/jarang.
- **Step dimension** — menganalisis langkah ke-n dalam suatu sesi/proses.
- **Hot swappable dimension** — dimensi yang bisa ditukar sesuai konteks pengguna.
- **Abstract generic dimension** — hati-hati; menggeneralisasi dimensi (mis. "lokasi" generik) hanya bila benar-benar konsisten.
- **Audit dimension** — melekatkan metadata ETL (kualitas, versi, timestamp) ke baris fakta untuk pelacakan.
- **Late arriving dimension** — anggota dimensi yang datang setelah faktanya; ETL harus menyisipkan baris & memperbaiki FK.

---

## 9. Skema Bertujuan Khusus

- **Supertype & subtype schema** — untuk produk heterogen (mis. beragam jenis akun bank): satu skema *supertype* dengan atribut umum + skema *subtype* per jenis dengan atribut spesifik.
- **Real-time fact table** — menampung data mendekati real-time (mis. partisi "hot").
- **Error event schema** — skema untuk mencatat pelanggaran aturan kualitas data selama ETL (dibahas di Bab 19).

---

## Ringkasan Bab

Bab 2 adalah **peta seluruh persenjataan teknik** Kimball. Yang paling penting untuk dikuasai lebih dulu: **proses desain 4 langkah** (proses bisnis → grain → dimensi → fakta), **tiga jenis grain fact table** (transaction, periodic snapshot, accumulating snapshot), **conformed dimension + bus matrix** sebagai fondasi integrasi enterprise, dan **SCD Type 0–7** untuk menangani perubahan atribut dimensi. Teknik lanjutan (bridge table, junk dimension, mini-dimension, allocated fact, dll.) akan muncul dan didemonstrasikan secara konkret pada studi kasus per industri mulai **Bab 3 (Retail Sales)**.
