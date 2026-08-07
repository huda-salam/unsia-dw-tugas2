# Bab 4 — Inventory (Persediaan)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Masih di industri ritel, bab ini membahas **proses bisnis kedua** (persediaan) untuk menunjukkan bagaimana model baru memanfaatkan dimensi yang sudah ada — bukan membangun *stovepipe*. Dua kontribusi besar bab ini: (1) melengkapi **tiga jenis fact table** (transaction, periodic snapshot, accumulating snapshot) dengan konsep **fakta semi-aditif**; dan (2) memaparkan secara mendalam **enterprise data warehouse bus architecture, bus matrix, conformed dimension**, dan pentingnya **data governance** — fondasi integrasi seluruh DW enterprise.

---

## 1. Value Chain & Tiga Model Persediaan

**Value chain** = rangkaian proses bisnis berurutan (mis. terbitkan PO → terima di gudang → simpan → kirim ke toko → jual). Tiap proses menghasilkan metrik unik pada interval, granularitas, dan dimensionalitas berbeda → tiap proses melahirkan satu atau lebih fact table. Value chain memberi gambaran arsitektur data enterprise secara keseluruhan.

Persediaan bisa dimodelkan dengan **tiga cara komplementer** (kadang perlu dua atau ketiganya sekaligus):

| Model | Cara kerja | Cocok untuk |
|---|---|---|
| **Periodic snapshot** | Ukur level stok pada interval reguler; tiap snapshot jadi baris baru (menumpuk seperti lapisan sedimen) | Persediaan berjalan terus yang diisi ulang |
| **Transaction** | Catat **setiap transaksi** yang memengaruhi stok | Frekuensi & timing jenis transaksi tertentu |
| **Accumulating snapshot** | Satu baris per penerimaan produk, **di-update** saat produk bergerak lewat milestone | Pipeline stok berhingga (awal–akhir jelas) |

---

## 2. Inventory Periodic Snapshot & Fakta Semi-Aditif

**Desain (4 langkah):** proses = snapshot harian stok toko; **grain = satu baris per produk per toko per hari**; dimensi = Date, Product, Store; fakta paling sederhana = **Quantity on Hand**. (Catatan: pada periodic snapshot, grain sering tak bisa dinyatakan lewat "transaksi" → dinyatakan lewat daftar dimensi. Promosi tidak relevan di sini karena promosi terkait pergerakan produk, bukan level stok.)

- **Tabel padat (dense)** — beda dengan sales yang sparse. Karena peritel ingin menghindari *out-of-stock*, ada baris untuk **setiap produk di setiap toko setiap hari** (mis. 60.000 × 100 = 6 juta baris/hari, tapi hanya ~84 MB karena baris sempit). Kompromi bila terlalu padat: kurangi frekuensi snapshot seiring waktu (mis. 60 hari terakhir harian, sisanya mingguan — di **dua fact table terpisah** karena periodisitasnya beda).

- **Fakta semi-aditif** — konsep kunci bab ini. Quantity on hand **bisa** dijumlahkan lintas produk & toko, tapi **tidak bisa** dijumlahkan lintas **tanggal** (ia snapshot level/saldo pada satu titik waktu). Sama seperti saldo rekening bank: saldo Senin–Jumat tidak dijumlahkan. Cara agregasi lintas waktu yang benar: **rata-rata** (mis. average daily balance).
  > Semua ukuran "level statis" (stok, saldo finansial, ukuran intensitas seperti suhu) **non-aditif lintas tanggal**; agregasikan dengan merata-ratakan.
  - **Hati-hati:** fungsi SQL `AVG` **salah** untuk ini — ia membagi dengan jumlah **semua baris**, bukan jumlah **periode waktu**. (Contoh: 3 produk × 4 toko × 7 hari → `AVG` membagi dengan 84, padahal seharusnya dibagi 7.) OLAP cube bisa mendefinisikan aturan agregasi sehingga semi-aditif tidak bermasalah.

- **Enhanced inventory facts** — quantity on hand saja tidak cukup; digabung dengan fakta lain untuk mengukur *velocity* pergerakan stok (mis. jumlah *turns*, *days' supply*).

---

## 3. Inventory Transaction

Model kedua: catat **setiap transaksi** yang memengaruhi stok (terima produk, tahan inspeksi, lepas inspeksi, kembalikan ke vendor, tempatkan di bin, ambil dari bin, kemas, kirim, terima retur, dst.). **Grain = satu baris per transaksi persediaan**; dimensi = Date, Product, Warehouse, (Vendor), Transaction Type; fakta = Inventory Transaction Dollar Amount; plus degenerate dimension nomor transaksi.

- Sangat detail — menjawab pertanyaan yang tak terjawab periodic snapshot.
- **Tapi tidak praktis dipakai sendirian** untuk analisis performa stok: merekonstruksi posisi stok dari seluruh transaksi terlalu rumit & lambat. → *"Ada lebih dari sekadar transaksi"; snapshot sering melengkapi transaction fact table.*
- **Aturan penting:** jika dimensionalitas transaksi **bervariasi** per jenis event (mis. shipper hanya untuk penerimaan/pengiriman, customer hanya untuk pengiriman/retur), buat **serangkaian fact table terpisah**, bukan menjejalkan semua ke satu.
  > Jika pengukuran punya granularitas/dimensionalitas natural berbeda, kemungkinan berasal dari proses berbeda → modelkan sebagai fact table terpisah.

---

## 4. Inventory Accumulating Snapshot

Model ketiga: untuk proses dengan **awal jelas, akhir jelas, dan milestone** di antaranya. **Satu baris per penerimaan lot produk**, di-update berulang saat lot bergerak lewat milestone (received → inspected → bin placement → shipped) hingga habis.

- Punya **banyak FK tanggal** — satu per milestone (di-handle sebagai *role-playing date dimension*, dibahas Bab 6). Tanggal yang belum diketahui memakai surrogate date key default.
- Berisi **banyak fakta kuantitas** per milestone (received, inspected, returned to vendor, placed in bin, shipped, damaged…) + **fakta lag** (durasi antar-milestone, mis. *receipt-to-inspected lag*) + **milestone counter** (0/1) + FK ke **status dimension** (mencerminkan status pipeline terkini).
- **Unik:** baris **di-update** (menimpa), hanya mencerminkan status **terkini** — beda dari periodic snapshot yang mengawetkan snapshot lama.
- **Bermasalah untuk OLAP cube** — karena update mengubah fakta **dan** FK, sebagian besar cube perlu diproses ulang (kecuali baris hanya dimuat setelah pipeline selesai).

---

## 5. Membandingkan Ketiga Jenis Fact Table

- **Periodic snapshot** — "foto" aktivitas di akhir tiap periode; satu-satunya cara mudah melihat tren performa longitudinal. Umumnya lebih sedikit dimensi tapi lebih banyak fakta dibanding transaction.
- **Accumulating snapshot** — untuk analisis *workflow/pipeline*; selalu punya banyak FK tanggal + fakta lag.
- **Complementary** — periodic & accumulating snapshot bisa bekerja sama (mis. membangun snapshot bulanan secara inkremental dari transaksi harian). Transaksi & snapshot adalah **"yin & yang"** desain dimensional — bersama memberi pandangan lengkap. Redundansi datanya bisa diterima (misi DW adalah mempublikasikan data agar bisa dianalisis).
  > Ketiga jenis fact table ini ternyata **cukup** untuk semua kasus di buku ini.

---

## 6. Enterprise Data Warehouse Bus Architecture

Membangun DW enterprise dalam satu upaya raksasa terlalu berisiko; membangunnya sebagai kepingan terisolasi mengalahkan tujuan konsistensi. Solusinya: **pendekatan terarsitektur & inkremental** — *bus architecture*.

- Istilah **"bus"** berasal dari industri kelistrikan/komputer: struktur bersama tempat semua perangkat terhubung & memperoleh daya (seperti bus pada komputer tempat berbagai perangkat "plug-in" dan bekerja bersama meski dibuat vendor/waktu berbeda).
- Dengan mendefinisikan **antarmuka bus standar**, model-model dimensional dari proses bisnis berbeda bisa dibangun tim berbeda pada waktu berbeda, lalu **"plug together"** dan hidup berdampingan — asalkan patuh pada standar (**conformed dimension** bersama).
- Memberi "yang terbaik dari dua dunia": kerangka arsitektur menyeluruh **plus** pembagian masalah menjadi potongan proses bisnis seukuran gigitan yang bisa dikerjakan tim independen & asinkron.
- **Independen dari teknologi** — model relasional maupun OLAP bisa jadi peserta penuh asalkan dirancang di sekitar conformed dimension & fact.

---

## 7. Enterprise Data Warehouse Bus Matrix

Alat untuk mendokumentasikan & mengomunikasikan bus architecture.

- **Baris = proses bisnis** (bukan departemen!), teridentifikasi dari sistem sumber operasional. **Mulai proyek dari satu baris** untuk meminimalkan risiko (risiko terbesar ada di desain/pengembangan ETL). Proses konsolidasi (mis. **profitability** yang menggabungkan revenue & cost lintas proses) sangat berharga tapi jangan dikerjakan pertama.
- **Kolom = conformed dimension** bersama lintas enterprise. Umumnya matriks "kotak" ~25–50 baris & kolom (industri asuransi cenderung lebih banyak kolom).
- **Sel di-"X"** untuk menandai dimensi mana terpakai di proses mana.

**Banyak kegunaan:** perencanaan arsitektur, desain database, koordinasi data governance, estimasi proyek, dan komunikasi organisasi (ke atas ke manajemen, ke samping antar-tim). Alat sederhana yang menyampaikan *master plan* secara visual.

**Kesalahan umum bus matrix:**
- **Baris departemental / terlalu luas** — baris jangan seperti kotak bagan organisasi.
- **Baris report-centric / terlalu sempit** — baris jangan seperti daftar laporan; satu proses mendukung banyak analisis.
- **Kolom terlalu umum** — jangan menggabungkan populasi tanpa irisan (mis. "person" untuk karyawan + pemasok + pelanggan) jadi satu dimensi generik.
- **Kolom terpisah per level hierarki** — kolom harus di **level paling granular**; jangan buat kolom terpisah untuk week/month/quarter/year — cukup satu kolom "Date" (level dinyatakan dalam sel).

**Retrofit stovepipe:** model terisolasi lebih buruk dari sekadar peluang hilang — ia menghalangi DW koheren. Kadang bisa dipetakan ke dimensi terstandardisasi, tapi sering lebih hemat **membongkar & membangun ulang** ketimbang me-retrofit.

**Opportunity/Stakeholder Matrix** — varian: baris proses bisnis sama, tapi **kolom diganti fungsi bisnis** (merchandising, marketing, store ops, finance). Menunjukkan fungsi mana tertarik pada proses mana → siapa yang perlu diundang ke sesi requirement.

---

## 8. Conformed Dimension & Drilling Across

**Conformed dimension** (alias: common/master/reference/shared dimension) = fondasi bus. **Dibangun sekali di ETL**, lalu direplikasi (logis/fisik) ke seluruh enterprise. Penggunaannya harus **diwajibkan** (idealnya oleh CIO).

**Drilling across** — conformed dimension memungkinkan menggabungkan metrik dari **beberapa fact table** dalam satu laporan: query tiap model terpisah (*multipass SQL*), lalu **outer-join** hasilnya berdasar atribut dimensi bersama (mis. product name). Kalkulasi lintas-fakta dilakukan di aplikasi BI setelah hasil digabung. (Alias vendor: multipass, multi-fact, stitch query.)

**Jenis-jenis conformity:**
- **Identical** — persis sama: key, nama kolom, definisi, **dan nilai** identik (mis. "July" ≠ "JULY"; "Month" ≠ "Month Name"). Idealnya tabel fisik yang sama atau diduplikasi sinkron. Boleh ada atribut tambahan yang tidak conform (tak bisa dipakai drill-across).
- **Shrunken rollup (subset atribut)** — dimensi level lebih tinggi yang atributnya **strict subset** dari dimensi atomik (mis. Brand adalah rollup dari Product; forecast di level brand, sales di level product). Atribut bersama harus dilabeli & bernilai identik; PK-nya terpisah.
- **Shrunken (subset baris)** — dimensi pada level detail sama tapi hanya **sebagian baris** (mis. dimensi produk korporat penuh vs subset untuk satu lini bisnis).
- **Limited conformity** — kadang tidak realistis/tidak perlu meng-conform (mis. konglomerat lintas industri tanpa pelanggan/produk bersama & tanpa minat *cross-sell*). Uji lakmus: kesediaan menyepakati definisi bersama. Jika tak bersedia, jangan bangun DW enterprise. Tetap disarankan mulai dari *least-common-denominator* (walau hanya 1–2 atribut bersama).

---

## 9. Data Governance & Conformed Facts

**Data governance** — tantangan sesungguhnya conformed dimension bukan teknis, melainkan **konsensus enterprise** atas nama, definisi, & nilai atribut. Tanpa governance, muncul silo departemen dengan "versi kebenaran" yang mirip-tapi-beda.

- **Harus dipimpin bisnis, bukan IT** — IT bisa memfasilitasi, tapi jarang sukses sebagai penggerak tunggal karena tak punya otoritas organisasi. Pemimpin governance perlu: dihormati, paham operasi enterprise, mampu menyeimbangkan kebutuhan organisasi vs departemen, berwibawa menantang status quo, komunikatif, dan piawai bernegosiasi.
- **Tujuan governance:** menyepakati definisi/label/domain value ("berbicara bahasa yang sama"), plus menetapkan kebijakan kualitas, akurasi, keamanan & akses data.
- **Teknologi (ERP, MDM) membantu tapi tidak menyelesaikan** — governance yang kuat adalah prasyarat.
- **Conformed & Agile** — conformed dimension **memungkinkan** agile (bukan menghalangi): dimensi dibangun sekali & dipakai ulang, sehingga *time-to-market* proses baru menyusut (tinggal menambah fact table). Tak perlu semua orang sepakat atas semua atribut — mulai dari **subset atribut** penting lintas enterprise, ekspansi iteratif per sprint.

**Conformed facts** — 95% upaya arsitektur data ada di conformed dimension; sisanya ~5% pada **conformed fact**. Fakta yang hidup di lebih dari satu model (revenue, profit, harga/biaya standar, KPI) harus punya **definisi & rumus identik** bila dinamai sama; bila tak bisa di-conform persis, **beri nama berbeda** agar pengguna tak menggabungkan fakta inkompatibel. Bila satuan ukur natural berbeda (mis. *shipping case* di gudang vs *scanned unit* di toko), simpan fakta dalam **kedua satuan** (dibahas lebih lanjut di Bab 6).

---

## Ringkasan Bab

Bab 4 melengkapi **tiga jenis fact table** — *periodic snapshot* (untuk persediaan berjalan; memperkenalkan **fakta semi-aditif** yang harus dirata-rata lintas waktu), *transaction* (detail per transaksi), dan *accumulating snapshot* (untuk pipeline berhingga; baris di-update per milestone dengan **fakta lag**) — dan menegaskan ketiganya saling melengkapi. Kontribusi konseptual terbesarnya: **enterprise data warehouse bus architecture** dan **bus matrix** (baris = proses bisnis, kolom = conformed dimension) sebagai kerangka membangun DW enterprise secara inkremental, ditopang **conformed dimension** (dengan varian identical/shrunken rollup/row subset), **drilling across**, **conformed facts**, dan **data governance yang dipimpin bisnis**. Bab 5 (Procurement) akan memperkenalkan penanganan perubahan atribut dimensi (**Slowly Changing Dimensions**).
