# Bab 6 — Order Management (Manajemen Pesanan)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Order management adalah rangkaian proses yang paling sering menjadi target awal proyek DW/BI: quoting → ordering → shipping → invoicing → payments → returns. Bab ini padat teknik: **role-playing dimension**, **junk dimension**, **degenerate dimension**, penanganan **multiple currency** & **multiple unit of measure**, fakta pada **granularitas berbeda** (alokasi), **P&L invoice**, **audit dimension**, dan **accumulating snapshot** untuk pipeline pemenuhan order — plus beberapa **pola yang harus dihindari**.

---

## 1. Skema Order Transaction & Normalisasi Fakta

**Grain = satu baris per baris item pesanan (order line).** Dimensi: Order Date, Requested Ship Date, Customer, Product, Sales Rep, Deal; degenerate: Order Number & Order Line Number. Fakta: order quantity, extended gross/discount/net dollar amount.

**Fact normalization (umumnya dihindari):** godaan menyimpan satu fakta generik + "measurement type dimension" (grain jadi satu baris per pengukuran). **Hindari** kecuali fakta sangat panjang & jarang terisi tanpa perhitungan antar-fakta. Alasannya: menormalisasi fakta **melipatgandakan jumlah baris** (mis. 10 juta baris × 4 fakta → 40 juta baris) dan menyulitkan perhitungan antar-fakta (SQL sulit menghitung rasio/selisih antar baris). Lebih pas bila platform BI-nya OLAP cube (lihat contoh di Bab 14 Healthcare).

---

## 2. Role-Playing Dimension

Sebuah **dimensi tunggal muncul beberapa kali** di fact table yang sama dengan **peran berbeda** — mis. Order Date & Requested Ship Date. Kedua FK **tidak boleh** join langsung ke satu tabel date yang sama (SQL akan menuntut kedua tanggal identik).

- Solusi: **bangun & kelola satu tabel date fisik**, lalu buat ilusi dua dimensi independen lewat **view atau alias**. **Beri nama kolom yang unik** per peran (mis. `Order Month` vs `Requested Ship Month`) agar tak tertukar di laporan.
- **Jangan** membuat satu tabel date berisi kombinasi (order date × ship date) — akan membengkak dan tak lagi *conform* dengan dimensi date harian/mingguan/bulanan lain.
- **Di bus matrix:** tulis banyak peran dalam satu sel, atau buat sub-kolom.
- Catatan: sebagian produk OLAP tak mendukung multiple role → perlu dimensi fisik terpisah.

---

## 3. Dimensi-Dimensi Order

- **Customer** — memuat hierarki ship-to (geografis) & bill-to, nama organisasi/korporat induk, credit rating. Keputusan **satu vs banyak dimensi** (ship-to & bill-to): tergantung apakah atribut berkorelasi dan cara pengguna menganalisis.
- **Sales Rep–Customer assignment** bisa ditangani lewat **factless fact table** (dengan effective/expiration date) bila penugasan berubah sepanjang waktu.
- **Deal Dimension** — mirip promotion dimension (Bab 3): term, allowance, incentive yang ditawarkan. Bila faktor-faktornya berkorelasi → satu dimensi; bila tidak (menghasilkan *Cartesian product*) → pecah. Bukan soal informasi (sama saja), melainkan kenyamanan pengguna & kompleksitas admin.

---

## 4. Degenerate Dimension untuk Order Number

Order number disimpan sebagai **degenerate dimension** (di fact table, tanpa tabel dimensi) — berbeda dari database operasional header/line. Semua detail menarik dari order header di-*triage* ke dimensi lain (order date, customer ship-to, dll.). Order number tetap berguna untuk: mengelompokkan baris satu order (mis. "rata-rata jumlah baris per order"), menautkan ke sistem operasional, dan sebagai bagian primary key.
> Degenerate dimension diperuntukkan bagi **pengenal transaksi operasional** — bukan dalih menaruh kode kriptik di fact table tanpa dimensi decode.

---

## 5. Junk Dimension

Untuk aneka **flag & indikator berkardinalitas rendah**. Opsi buruk: mengabaikannya, membiarkannya (kode kriptik/teks besar) di fact row, atau menjadikan tiap flag dimensi terpisah (menambah FK). Solusi rapi: **bungkus jadi satu (atau beberapa) junk dimension** — seperti "laci serba-guna" di dapur.

- Baris junk dimension = **kombinasi nilai yang benar-benar muncul** dalam data (bukan seluruh Cartesian product bila banyak yang tak terpakai).
- Mengurangi jumlah FK di fact table & merapikan flag yang berserakan.

---

## 6. Pola Header/Line yang Harus Dihindari

**Pola #1 — memperlakukan order header sebagai dimensi.** Mereplikasi header operasional jadi satu "order header dimension" (natural key = order number, memuat semua atribut header). Meski relasinya benar, cacatnya jelas: dimensi header jadi **sangat besar relatif terhadap fact table** (mis. 5 baris/order → dimensi = 20% ukuran fakta; padahal seharusnya beda orde besar), tumbuh secepat fact table, dan analisis apa pun (customer/rep/deal) harus melintasi dimensi raksasa itu.
→ **Yang benar:** pecah atribut header ke dimensi analitik masing-masing (customer, deal, dll.) yang dikaitkan ke fact table grain order line; order number cukup jadi degenerate dimension.

---

## 7. Multiple Currency

Perusahaan multinasional menerima order dalam belasan mata uang. Jangan buat kolom per mata uang. Solusi: **setiap fakta digandakan** — satu dalam **mata uang lokal**, satu dalam **mata uang korporat standar** (mis. USD).

- Kurs konversi per baris sesuai aturan bisnis (saat order, akhir hari, dst.). Metrik **standar** fully additive; metrik **lokal** hanya aditif untuk satu mata uang yang sama.
- Tambahkan **Currency Dimension** untuk menandai mata uang lokal (lokasi transaksi tak menjamin mata uang yang dipakai).
- Konversi antar-baris (rate × amount) sebaiknya di **view**, bukan dibebankan ke pengguna.
- Untuk kebutuhan kompleks (manajer mana pun ingin lihat volume dalam mata uang mana pun): tambah **currency conversion fact table** harian. Dimensinya = mata uang (bukan negara; relasi mata uang↔negara bukan 1:1). Simpan kurs **dua arah** (rate asimetris). Tak perlu Cartesian product penuh (~100 mata uang ≠ 10.000 baris/hari) — hanya pasangan yang bermakna.

---

## 8. Fakta pada Granularitas Berbeda & Alokasi

Data header/line kerap punya fakta beda granularitas (mis. ongkos kirim berlaku untuk **seluruh order**, bukan per baris). **Respons pertama: alokasikan (*allocate*)** fakta level-header ke level baris agar semua fakta bisa di-slice/dice & di-roll up per semua dimensi (termasuk product).
- Alokasi sering jadi "pertarungan politik" — idealnya ditangani **departemen finance** (via *activity-based costing*), bukan tim DW/BI, agar proyek tak terhambat negosiasi konsensus.

---

## 9. Invoice Transactions & P&L

Skema **invoice line item** termasuk yang paling **powerful**: dalam satu skema menampilkan customer, product, revenue, cost, hingga **contribution (bottom-line profit)** — sebuah **P&L view** lengkap di dalam kerangka dimensional yang kaya.

- Fakta P&L (per baris invoice): gross → dikurangi allowance & discount → **net** → dikurangi rangkaian biaya (fixed & variable manufacturing cost, storage cost, distribution cost) → **contribution amount**. Semua biaya idealnya **dialokasikan ke level baris** (tanpa product dimension, P&L kehilangan makna).
- **Peringatan profitabilitas:** menyediakan P&L granular sering **lebih sulit dari yang terlihat** — fakta biaya kerap tak tersedia di level atomik (butuh pemetaan/alokasi rumit, tiap jenis biaya = program ekstraksi terpisah). Lakukan asesmen kelayakan sumber dulu. Profitabilitas sering paling baik dikerjakan sebagai **consolidated model** setelah komponen revenue & cost tersedia terpisah.
- **Service level performance** — tanggal-tanggal kritis pengiriman dipakai sebagai role-playing date; bisa ditambah *on-time counter* (0/1), *lag metric* (hari lebih cepat/lambat), dan **penilaian kualitatif** (mis. "On-time", "1 day late") sebagai atribut junk/dimensi tersendiri — jadi bisa jadi **fakta, dimensi, atau keduanya**.

**Audit Dimension** — melekatkan **metadata ETL** ke baris fakta lewat satu FK: quality indicator, out-of-bounds indicator, amount-adjusted flag (dari *error event table*, Bab 19), plus versi logika alokasi biaya & konversi mata uang. Menjawab pertanyaan bisnis seperti "seberapa yakin saya pada angka ini?". Mulai dengan desain audit dimension yang **sederhana**.

---

## 10. Accumulating Snapshot untuk Order Fulfillment Pipeline

Pipeline: Order → Backlog → Release to Manufacturing → Finished Goods Inventory → Shipment → Invoicing. Selama ini tiap tahap = transaction fact table terpisah (bagus untuk detail & analisis per proses). Tapi untuk memahami **kecepatan produk (velocity)** melintasi **seluruh** pipeline → gunakan **accumulating snapshot**.

- **Grain = satu baris per order line**, tapi **baris di-update** saat order bergerak lewat milestone (pembeda utama accumulating snapshot — bukan sekadar "punya banyak tanggal").
- Punya **banyak FK tanggal** (mis. 9 peran) sebagai role-playing date; tanggal yang belum diketahui memakai baris "Unknown/TBD". Berisi banyak fakta kuantitas per milestone + **fakta lag** (order-to-manufacturing-release, order-to-shipment, dll.).
- **Melengkapi** transaksi (jumlah produk mengalir) & periodic snapshot (jumlah produk mengendap). Paling cocok untuk **proses berumur pendek** dengan awal-akhir jelas & item ter-identifikasi unik (VIN, serial number, lot number); proses berumur panjang (mis. rekening bank) lebih cocok periodic snapshot.
- **Dengan dimensi Type 2:** untuk pipeline aktif, fact table di-update ke surrogate key **terkini**; setelah baris selesai, umumnya tak lagi direvisi.
- **Lag calculation** — selisih antar tanggal (bisa dirata-rata lintas dimensi). ETL bisa menghitung lag yang lebih cerdas (mis. *workday* memperhitungkan akhir pekan & libur), disajikan lewat view.

---

## 11. Multiple Unit of Measure

Pihak berbeda ingin melihat kuantitas dalam satuan berbeda (pallet, shipping case, retail case, scan unit, equivalized consumer unit). **Jangan** menyembunyikan faktor konversi di product dimension (membebani pengguna & rawan salah, apalagi bila faktor berubah seiring waktu).

- **Simpan faktor konversi di fact table.** Bila ada 10 fakta kuantitas × 5 satuan, jangan simpan 50 kolom; simpan **10 fakta dasar + 4 faktor konversi** (= 14 kolom; cukup 4 karena fakta dasar sudah dalam satu satuan).
- Sajikan ke pengguna lewat **view** (perhitungan intra-baris sangat efisien); beri label unik per satuan.
- Bonus: menyimpan faktor di fakta mengurangi tekanan menerbitkan baris product baru hanya karena faktor konversi berubah — faktor ini berperilaku lebih seperti **fakta** ketimbang atribut dimensi.

---

## 12. Beyond the Rearview Mirror

Sebagian besar metrik di bab ini bersifat "kaca spion" (melihat ke belakang). Organisasi ingin melengkapinya dengan **indikator awal (leading indicator)** — mis. aktivitas prospecting/quoting yang bisa diekstrapolasi untuk memproyeksikan volume order. Ini cukup ditambahkan sebagai **baris baru di bus matrix** yang berbagi dimensi bersama.

---

## Ringkasan Bab

Bab 6 membahas banyak "multiple": **role-playing dimension** (satu dimensi, banyak peran via view/alias), **multiple currency** (fakta lokal + korporat, plus currency conversion fact table), dan **multiple unit of measure** (fakta dasar + faktor konversi di fact table). Diperkenalkan pula **junk dimension** (membungkus flag berkardinalitas rendah), **degenerate dimension** untuk order number, penanganan **fakta beda granularitas** lewat **alokasi**, skema **invoice P&L** yang kuat (dengan peringatan kesulitan data biaya), **audit dimension** (metadata ETL), dan **accumulating snapshot** untuk pipeline pemenuhan order. Ditutup dengan pola **header/line yang harus dihindari** dan gagasan menambah **leading indicator**. Bab 7 (Accounting) akan membahas general ledger, periodic snapshot, dan consolidated fact table.
