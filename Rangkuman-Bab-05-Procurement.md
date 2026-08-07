# Bab 5 — Procurement (Pengadaan)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab ini punya dua fokus. Pertama, keputusan desain klasik: **satu fact table gabungan vs banyak fact table terpisah** untuk proses transaksional yang bertahap. Kedua — dan ini kontribusi terpentingnya di seluruh buku — penanganan **Slowly Changing Dimension (SCD) Type 0–7**, yaitu strategi menghadapi atribut dimensi yang berubah perlahan seiring waktu.

---

## 1. Studi Kasus Procurement

Pengadaan (procurement) mencakup rantai proses: purchase requisition → purchase order → shipping notification → warehouse receipt → vendor invoice → vendor payment. Mengelola performa pengadaan (harga vendor, diskon, ketepatan) berdampak langsung pada laba. Muncul dimensi baru **Vendor** (dengan atribut seperti status, minority ownership flag, corporate parent) dan berbagai *control number* (nomor PO, nomor cek pembayaran) yang jadi kandidat **degenerate dimension**.

---

## 2. Satu vs Banyak Transaction Fact Table

Pertanyaan desain: buat **satu** fact table gabungan (dengan dimensi "transaction type") atau **beberapa** fact table terpisah per jenis transaksi? Tidak ada rumus pasti — putuskan berdasar kebutuhan bisnis + realita data sumber. Pertimbangan yang membantu:

- **Kebutuhan analitik pengguna** — bagaimana bisnis paling sering menganalisis? Pilih bentuk yang paling sederhana bagi mereka.
- **Apakah benar-benar proses bisnis berbeda?** — di procurement, "membeli" (PO) jelas beda dari "menerima" (receipt). **Adanya control number terpisah tiap langkah = petunjuk proses terpisah** → condong ke fact table terpisah. (Bandingkan Bab 4: aneka transaksi persediaan adalah **satu** proses → satu fact table.)
- **Apakah banyak sistem sumber dengan granularitas unik?** — di sini ada 3 sistem (purchasing, warehousing, accounts payable) → condong ke fact table terpisah.
- **Bagaimana dimensionalitas fakta?** — beberapa dimensi hanya berlaku untuk sebagian jenis transaksi (mis. diskon hanya untuk pembayaran; penerima barang hanya untuk receipt) → condong ke fact table terpisah.

Hasilnya: procurement dimodelkan sebagai **beberapa transaction fact table** (satu per proses), masing-masing baris di bus matrix.

**Complementary snapshot** — untuk memantau pergerakan produk melintasi seluruh pipeline (termasuk durasi tiap tahap), tambahkan **accumulating snapshot** (procurement pipeline fact) dengan banyak FK tanggal per milestone + fakta lag. Ingat: accumulating snapshot hanya untuk proses dengan **milestone jelas**; aliran kontinu tanpa akhir bukan kandidat.

---

## 3. Dasar Slowly Changing Dimension (SCD)

Selama ini kita berpura-pura dimensi tidak berubah. Kenyataannya, nilai atribut dimensi **berubah perlahan** seiring waktu. Untuk **tiap atribut**, kita harus menetapkan strategi: saat nilai berubah di dunia operasional, bagaimana model dimensional merespons?

> **Data governance/steward bisnis harus terlibat** menentukan strategi SCD — IT tidak boleh memutuskan sendiri. Jangan menyimpulkan "bisnis tidak peduli perubahan" hanya karena tak disebut saat requirement.

Beberapa teknik bisa dipakai **bersamaan** dalam satu dimensi. Type 1, 2, 3 sudah lama dikenal; Kimball memberi nomor pada teknik lain (0, 4–7) demi komunikasi yang lebih jelas.

---

## 4. Teknik SCD Dasar (Type 0–3)

Contoh berjalan: software *IntelliKidz* dipindah dari departemen **Education** ke **Strategy** pada 1 Feb 2013.

**Type 0 — Retain Original.** Nilai atribut **tidak pernah berubah**; fakta selalu dikelompokkan berdasar nilai asli. Cocok untuk atribut berlabel "original" (mis. skor kredit awal) dan mayoritas atribut date dimension. **Durable supernatural key selalu Type 0.**

**Type 1 — Overwrite.** Nilai lama **ditimpa** nilai baru; atribut selalu mencerminkan nilai terkini. Mudah diterapkan, key tak berubah, tapi **histori hilang total** — semua fakta (lama & baru) tampak seolah IntelliKidz selalu di Strategy.
- ⚠️ **Efek samping penting:** laporan yang sama bisa memberi hasil berbeda sebelum vs sesudah overwrite; **agregat & OLAP cube yang meringkas atribut itu harus dibangun/diproses ulang** agar tetap cocok dengan data atomik.
- Cocok hanya bila perubahan berupa **koreksi tak signifikan** atau nilai lama tak bernilai. Jangan jadikan default.

**Type 2 — Add New Row.** Teknik **utama** & "workhorse" untuk merepresentasikan histori secara akurat. Saat atribut berubah, **sisipkan baris dimensi baru** dengan **surrogate key baru** untuk profil baru; baris lama tetap.
- Fakta **tidak disentuh**: baris fakta sebelum 1 Feb tetap menunjuk key lama (Education), sesudahnya menunjuk key baru (Strategy). Ini **mem-partisi histori** dengan sempurna — laporan periode lama tetap identik meski dijalankan setelah perubahan. (Contoh: $500 Education + $100 Strategy, bukan $600 Strategy seperti Type 1.)
- **Tidak perlu** membangun ulang agregat / memproses ulang cube (beda dari Type 1).
- **Kolom administratif wajib:** *row effective date*, *row expiration date* (baris terkini di-set `9999-12-31` agar bisa pakai `BETWEEN`, hindari null), dan *current row indicator*. Baris lama "di-expire" tepat sebelum efektif baris baru (tanpa celah).
- Butuh **surrogate key** (natural key tak bisa dipakai karena satu natural key kini punya banyak profil). Untuk menghitung jumlah produk yang benar, pakai **natural key** sebagai basis *distinct count* — ia "lem" yang menyatukan banyak baris Type 2 satu entitas.
- **Paling aman** bila aturan bisnis belum pasti — dari Type 2 bisa "berpura-pura" Type 1 (via Type 6/7), sebaliknya tidak (dari Type 1 ke Type 2 retroaktif sangat mahal).
- **Type 1 di dalam dimensi Type 2:** jika sebuah atribut Type 1 dikoreksi setelah ada perubahan Type 2 pada atribut lain, koreksi itu mungkin harus diterapkan ke **semua** baris profil entitas tersebut (steward yang memutuskan).

**Type 3 — Add New Attribute.** Tambah **kolom baru** untuk menyimpan nilai lama (mis. "Prior Department Name" = Education), sementara kolom utama di-overwrite ke nilai baru (Strategy, seperti Type 1). Memungkinkan melihat data **seolah perubahan tak pernah terjadi** ("alternate reality") — nilai kini & lama dianggap benar **bersamaan**.
- **Jarang dipakai** (nomor tinggi ≠ lebih baik). Cocok untuk **perubahan masif serentak** yang bisa diprediksi (mis. reorganisasi teritori sales, penataan ulang lini produk), bukan perubahan tak terduga per-baris (mis. pindah domisili pelanggan — itu Type 2).
- **Multiple Type 3** — untuk perubahan berirama teratur (mis. rekategorisasi tiap awal tahun): sediakan kolom "current" + kolom per tahun ("2012 department", "2011 department"), produk baru diberi "Not Applicable".

---

## 5. Type 4 — Mini-Dimension (Rapidly Changing "Monster")

Untuk dimensi **berjuta baris** yang sebagian atributnya **cepat berubah** (mis. customer). Type 2 tidak menarik di sini (akan menambah jutaan baris lagi). Solusi: **pecah atribut yang volatil/sering dianalisis** ke **mini-dimension** terpisah.

- Contoh: mini-dimension **demografi** (age band, purchase frequency score, income level). **Satu baris per kombinasi unik nilai**, bukan per pelanggan → jauh lebih kecil.
- Atribut kontinu (income) **di-band** jadi rentang diskret (mis. `$30,000–39,999`) untuk membatasi jumlah kombinasi — ini kompromi terbesar teknik ini. Jika pengguna butuh nilai mentah (mis. skor biro kredit), simpan juga **di fact table**.
- **Fact table memuat dua FK pelanggan:** customer key + demographics key **yang berlaku saat event**. Ini memberi manfaat performa (titik masuk kecil, hindari tabel customer raksasa) **dan** melacak perubahan profil (mis. John ulang tahun → key demografi baru pada muatan fakta berikutnya; baris lama tak diubah).
- **Batasi ukuran** mini-dimension (mis. 5 atribut × 10 nilai = 100.000 baris = batas atas wajar). Untuk kebutuhan lebih besar, pakai beberapa mini-dimension (Bab 10).

---

## 6. Teknik SCD Hybrid (Type 5–7)

Dipakai saat perlu **melestarikan atribut historis** (yang berlaku saat fakta terjadi) **sekaligus** melaporkan fakta historis berdasar **nilai atribut kini**. Fleksibilitas tinggi, tapi **kompleksitas tinggi** — jangan pakai kecuali bisnis benar-benar membutuhkannya. (Penamaan: 4+1=5, 2×3×1=6, 7 sekadar nomor berikut.)

**Type 5 — Mini-Dimension + Type 1 Outrigger.** Type 4 + tanamkan **current mini-dimension key** sebagai atribut **Type 1** (di-overwrite) di dimensi utama. Memungkinkan roll-up fakta historis berdasar **profil terkini** pelanggan, atau hitung profil kini tanpa fakta. Dipapar sebagai satu tabel logis; beri prefiks "current" pada kolomnya. (Istilah: bila demographics key di *fact table* → *mini-dimension*; bila di *customer dimension* → *outrigger*.)

**Type 6 — Type 1 Attributes di Dimensi Type 2.** Dua kolom departemen per baris: **historic** (Type 2, akurat saat fakta terjadi) + **current** (Type 1, di-overwrite di **semua** baris entitas saat berubah). Satu dimensi bisa mengelompokkan fakta lewat nilai saat kejadian **atau** nilai kini. (Gabungan Type 1+2+3 dalam satu kolom-berpasangan; issue baris = Type 2, tambah kolom current = Type 3, update = Type 1.)

**Type 7 — Dual Type 1 & Type 2.** Fact table memuat **dua FK**: surrogate key (untuk Type 2, join ke dimensi historis) **dan** durable key (join ke tabel/view dimensi **Type 1 current-only**). Memberi fungsi sama seperti Type 6 tapi **ETL lebih ringan** (tabel current cukup dibuat lewat *view* baris terkini). Cocok bila perlu perspektif current & historic untuk **banyak atribut** dalam dimensi besar. Biaya: satu kolom ekstra di fact table.

---

## 7. Rekap SCD Type 0–7

| Type | Aksi pada Dimensi | Dampak pada Analisis Fakta |
|---|---|---|
| **0** | Nilai tak diubah | Fakta terkait nilai **asli** |
| **1** | Timpa nilai | Fakta terkait nilai **kini** (histori hilang) |
| **2** | Tambah baris baru untuk profil | Fakta terkait nilai **saat kejadian** |
| **3** | Tambah kolom (nilai kini & lama) | Fakta bisa dilihat via nilai **kini atau lama** (alternate reality) |
| **4** | Tambah mini-dimension atribut cepat-berubah | Fakta terkait atribut cepat-berubah **saat kejadian** |
| **5** | Type 4 + outrigger key Type 1 di base dim | **Saat kejadian** + nilai cepat-berubah **kini** |
| **6** | Kolom Type 1 di baris Type 2 (timpa semua baris lama) | **Saat kejadian** + nilai **kini** |
| **7** | Baris Type 2 + view current (dual FK di fakta) | **Saat kejadian** + nilai **kini** |

---

## Ringkasan Bab

Bab 5 memberi dua pelajaran besar. **Pertama**, keputusan **satu vs banyak fact table**: petunjuk ke arah fact table terpisah adalah proses bisnis yang benar-benar berbeda, control number terpisah, banyak sistem sumber, dan dimensionalitas yang bervariasi. **Kedua**, dan yang paling fundamental, **Slowly Changing Dimension Type 0–7** — dari tidak berubah (Type 0), menimpa (Type 1), menambah baris untuk mengawetkan histori (Type 2, teknik utama), menambah kolom untuk realitas alternatif (Type 3), memecah atribut volatil ke mini-dimension (Type 4), hingga hybrid (Type 5–7) yang menyajikan perspektif historis **dan** kini sekaligus dengan konsekuensi kompleksitas. Pilihan strategi harus melibatkan **data steward bisnis** dan menyeimbangkan fleksibilitas vs kompleksitas. Bab 6 (Order Management) akan memperluas ini dengan role-playing dimension, junk dimension, dan penanganan multiple currency/unit of measure.
