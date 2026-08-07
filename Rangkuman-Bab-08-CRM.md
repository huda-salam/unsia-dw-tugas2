# Bab 8 — Customer Relationship Management (CRM)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab ini seluruhnya berfokus pada **customer** — dimensi yang paling menantang di DW/BI (bisa berjuta baris, ratusan atribut, cepat berubah, gabungan banyak sumber). Inti CRM: **semakin baik Anda mengenal pelanggan, semakin baik hubungan jangka panjang yang bisa dijaga**, dan itu menuntut **satu pandangan pelanggan yang terintegrasi (360°)**. Bab ini membahas atribut customer dimension (parsing nama/alamat, tanggal, segmentasi, behavior tag), **outrigger**, **bridge table** untuk dimensi multivalued, dan perilaku pelanggan kompleks (**study group**, **step dimension**, **timespan fact table**).

---

## 1. CRM: Operasional & Analitik

CRM bertujuan **memaksimalkan hubungan dengan pelanggan sepanjang masa hidupnya** (customer lifetime), memfokuskan seluruh fungsi (marketing, sales, operations, service) pada relasi yang saling menguntungkan. CRM punya "kepribadian ganda":

- **CRM operasional** — sinkronisasi proses yang menyentuh pelanggan; metrik & karakteristik dikumpulkan di setiap *touch point* dan dibagikan lintas fungsi.
- **CRM analitik** — memanfaatkan data pelanggan yang akurat & terintegrasi di DW/BI untuk mengukur efektivitas keputusan, menemukan peluang up-sell/cross-sell, dan menghasilkan model/skor. **DW/BI adalah inti CRM** — fondasi pandangan 360°.
- **Closed loop:** Collect → Integrate (ETL) → Store → Analyze/Model → hasil model **didorong kembali** ke titik operasional (rep, call center, website) untuk tindakan berikutnya (mis. penawaran produk berikut, respons anti-attrition).

Tantangan terbesar bukan teknologi, melainkan **integrasi mendalam** — bisa sampai 100 sumber data pelanggan (kebanyakan eksternal) dengan granularitas & atribut berbeda. **Conformed customer dimension** menjadi sangat krusial. CRM juga menuntut komitmen bersama bisnis & IT (fokusnya lintas-fungsi).

---

## 2. Atribut Customer Dimension

Conformed customer dimension adalah **landasan** CRM. Beberapa jenis atribut penting:

**Parsing nama & alamat.** Penanganan operasional biasanya terlalu sederhana (kolom generik Name-1..3, Address-1..6) dan **nyaris tak berguna** untuk segmentasi — bahkan memicu masalah kualitas data. Solusi: **pecah ke elemen sekecil mungkin** (salutation, first/middle name, surname, suffix, title, street number/name/type/direction, city, district, state, region, country, postal code, telepon per komponen, email, dll.). ETL melakukan *parsing → standardisasi* (Rd→Road, Ste→Suite) → *verifikasi* (ZIP cocok dengan state). Tersedia tool *name/address cleansing*.

**Pertimbangan internasional.** Ini **masalah character set, bukan font**. ASCII (255 karakter) tak cukup untuk tulisan non-Latin. Gunakan **Unicode** — tapi diterapkan di lapisan fondasi (OS, database, aplikasi harus Unicode-compliant).

**Customer-centric dates** — tanggal seperti *first purchase*, *last purchase*, *date of birth*. Agar bisa diringkas per atribut kalender unik (season, kuartal, fiskal), ubah jadi **FK ke date dimension** (sebagai **outrigger** role-playing dengan label kolom unik, mis. "Date of 1st Purchase Month"). Pastikan semua tanggal berada dalam rentang date dimension korporat.

**Aggregated facts as dimension attributes** — mis. "total belanja tahun lalu" ditaruh sebagai **atribut** agar pengguna bisa memfilter customer ("high spender") tanpa query fakta terpisah. Hanya untuk **filter/label, bukan kalkulasi**. Bebannya ada di **ETL** (harus akurat, mutakhir, konsisten dengan fakta). Pilih yang **sering dipakai** & **jarang perlu update** (mis. "belanja tahun lalu" lebih ringan dari "year-to-date"). Sering diganti label deskriptif ("High Spender") demi konsistensi definisi.

**Segmentation attributes & scores** — atribut paling powerful: gender, ethnicity, life stage, income/lifestyle, status (new/active/inactive/closed), referring source, market segment. Banyak organisasi juga menempelkan **skor** dari model statistik (propensity to churn, probability to default, dll.).

---

## 3. Behavior Tag Time Series

Pendekatan populer men-skor pelanggan: **RFI/RFM** — *Recency* (berapa hari sejak terakhir order/kunjung), *Frequency* (berapa kali dalam setahun), *Intensity/Monetary* (berapa banyak uang). Tiap pelanggan = titik dalam **kubus RFI** (skala kuintil 1–5).

- Data mining mengelompokkan jutaan titik jadi **behavior tag** (mis. A="high volume, good credit, few returns", F="former good customer, not seen recently").
- Tiap periode, pelanggan ditandai cluster terdekat → **deret waktu tag tekstual**, mis. `John Doe: C C C D D A A A B B`.
- **Jangan simpan sebagai fakta** — tag bersifat tekstual (tak bisa dihitung/dirata-rata) tapi bisa di-query untuk pola kompleks. **Simpan sebagai deret atribut posisional di customer dimension** (satu kolom per periode) — interface BI sederhana, performa baik (bitmap index). Tambahkan juga **satu kolom konkatenasi** (`CCCDDAAABB`) untuk pencarian wildcard pola ("D diikuti B").

**Hubungan dengan data mining:** tim data mining adalah "klien" hebat DW, tapi ada *mismatch kecepatan* (tool bisa memproses ratusan record/detik, query drill-across besar tak bisa). Solusi: tulis hasil observasi ke **file** dan serahkan ke server data mining, jangan query ulang berulang.

---

## 4. Counts dengan Perubahan Type 2

Menghitung pelanggan berdasar atribut tanpa join ke fakta bisa **overcounting** bila customer dimension pakai SCD Type 2 (satu individu = banyak baris). Solusi: `COUNT DISTINCT` pada **durable customer identifier**, atau pakai **current row indicator** untuk hitung berdasar nilai terkini. Untuk hitung pada titik historis: constrain `row effective date <= tanggal <= row expiration date`.

---

## 5. Outrigger untuk Set Atribut Berkardinalitas Rendah

Snowflake umumnya dilarang, **tapi outrigger diperbolehkan** dalam kasus khusus. Contoh: blok 150 atribut demografi/sosio-ekonomi eksternal per *county* — identik untuk semua pelanggan di county sama. Daripada mengulang blok besar itu per pelanggan, modelkan sebagai **outrigger**. Alasan pengecualian: grain berbeda, nilai analitik lebih rendah, di-load pada waktu berbeda, hemat ruang signifikan (bisa disembunyikan di view).
> ⚠️ Outrigger adalah **pengecualian, bukan aturan**. Bila desain penuh outrigger, itu tanda over-normalisasi.

**Hierarki customer** — pelanggan komersial punya hierarki organisasi bertingkat (lokasi → kantor regional → HQ unit bisnis → parent). Teknik dari **Bab 7** langsung berlaku: fixed depth (jumlah level tetap), slightly variable (propagasi nilai level bawah ke atas), atau **ragged variable depth** (bridge table). Hindari label abstrak Level-1/Level-2.

---

## 6. Bridge Table untuk Dimensi Multivalued

Prinsip dasar: dimensi yang menempel ke fakta harus **bernilai tunggal**. Tapi beberapa dimensi "bermasalah" bernilai **jamak** pada grain transaksi — mis. demografi dari banyak sumber, alamat kontak pelanggan komersial, keahlian pelamar, hobi, diagnosis pasien, fitur opsional mobil, joint account holder.

Dua pilihan:
- **Positional** — kolom bernama per nilai (mis. kolom per hobi). Mudah di-query tapi **tak skalabel** (kehabisan kolom, banyak null). Cocok s/d ~100 kolom (columnar DB lebih toleran).
- **Bridge table** — baris hanya ada bila diperlukan → skalabel, tanpa null; tapi **query jadi kompleks** dan harus disembunyikan dari pengguna. ⚠️ Query bridge kadang di luar jangkauan SQL tool BI biasa.

**Bridge untuk atribut sparse (name-value pair).** Bila ada ratusan/ribuan atribut *open-ended* (mis. data aplikasi pinjaman berupa pasangan nama-nilai; nilai bisa numerik/teks/URL/pointer file), pakai bridge: `Application Disclosure Bridge` menghubungkan fakta ke `Disclosure Item Dimension` (item name, value type, value text string).

**Bridge untuk multiple contact.** Pelanggan komersial besar punya banyak kontak (decision maker, purchasing agent, dll.), tiap kontak punya **role**. Pakai `Contact Group Bridge` (contact group key, contact key, contact role). Jaga agar contact dimension tidak jadi "tempat pembuangan" semua orang — batasi pada konteks relasi pelanggan.

---

## 7. Perilaku Pelanggan Kompleks

**Behavior study group (cohort).** Pertanyaan pelanggan cepat jadi rumit (mis. "berapa pelanggan yang bulan ini belanja lebih dari rata-rata bulanan tahun lalu?") — terlalu kompleks untuk satu SQL. Solusi: jalankan query sekali untuk mengidentifikasi himpunan pelanggan (mis. top 100), lalu **tangkap durable key mereka sebagai tabel fisik satu kolom** (study group dimension). Karena pakai durable key, **kebal terhadap Type 2**. Di-join equijoin ke durable key customer dimension (bisa via view → terlihat seperti star biasa). Bisa dikombinasi dengan operasi **union/intersection/set difference** (mis. pelanggan bermasalah dua bulan berturut). Bisa ditambah kolom tanggal kejadian untuk *panel study*.

**Step dimension untuk perilaku sekuensial.** Untuk proses berurutan (mis. web session dari banyak page event). Step dimension **abstrak** yang dibuat di muka: tiap baris = (total steps, this step number, steps until end). Menempatkan satu langkah dalam konteks keseluruhan sesi. Bisa dipakai **beberapa peran** (mis. sesi keseluruhan, subsesi pembelian sukses, keranjang ditinggalkan) → bisa query "halaman pertama dari sesi sukses" atau "halaman terakhir keranjang ditinggalkan". Alternatif: kolom teks lebar berisi urutan kode produk (`11254|45882|…`) yang bisa dicari wildcard.

**Timespan fact table.** Untuk mengetahui status pelanggan pada **titik waktu arbitrer di masa lalu** ("apakah customer sedang fraud alert saat kredit ditolak?"). Kuncinya: **sepasang date/time stamp** per transaksi — *begin effective* (momen transaksi) & *end effective* (momen transaksi berikutnya) → rangkaian tanpa celah. Query cukup `begin <= waktu < end` untuk membekukan status pada instant tertentu, atau rentang tanggal untuk "semua yang pernah fraud alert di 2013". ETL mengelolanya dua langkah (baris baru pakai end date jauh di masa depan; baris sebelumnya di-update end date = begin baris baru). Hindari null (pakai tanggal fiktif jauh di masa depan).

---

## 8. Partial Conformity, Fact-to-Fact, & Low Latency

**Partial conformity multiple customer dimension.** Dengan puluhan sumber pelanggan beda granularitas/kualitas, membangun **satu** customer dimension komprehensif sering mustahil. Solusi ringan: dua dimensi cukup disebut *conformed* bila **berbagi beberapa atribut yang diadministrasi khusus** (nama kolom & nilai sama). Tanam atribut conformed (mis. "customer category", lalu geografi) secara **inkremental & agile** di tiap dimensi tanpa mengubah grain — cakupan integrasi tumbuh bertahap.

**Hindari fact-to-fact join.** Men-join **dua fact table** ke satu customer dimension bersamaan (many-to-one-to-many) **menghasilkan jawaban salah** karena perbedaan kardinalitas (mis. solicitation vs response — tak semua solicitation berujung response, dan sebaliknya). ⚠️ Tak ada kombinasi inner/outer/left/right join yang benar. Solusi: **drill-across** — query tiap fakta terpisah lalu outer-join hasilnya. Bila sering dikombinasi, buat **consolidated fact table** (dengan aturan bisnis untuk kardinalitas berbeda).

**Low latency reality check.** Data mendekati real-time berharga, **tapi kualitasnya turun** seiring makin cepat: batch harian → set transaksi lengkap + cek kualitas penuh; ekstrak beberapa kali/hari → set mungkin tak lengkap (belum lolos credit check), key mungkin belum resolve (pakai entri dimensi sementara); real-time instan → hanya fragmen, nyaris tanpa cek kualitas. Beri tahu pengguna trade-off ini. Pendekatan hybrid: low-latency siang hari, koreksi via batch malam.

---

## Ringkasan Bab

Bab 8 berfokus pada **customer** — dimensi tersulit sekaligus fondasi CRM (operasional + analitik, closed loop). Membangun customer dimension yang kaya menuntut **parsing nama/alamat** ke elemen dasar, **Unicode** untuk internasional, **customer-centric dates** (outrigger), **aggregated facts sebagai atribut** (untuk filter), **segmentation & behavior tag time series** (RFI/RFM, disimpan posisional bukan sebagai fakta). Diperkenalkan **outrigger** (pengecualian snowflake), **bridge table** untuk dimensi **multivalued/sparse** (name-value pair, multiple contact), serta perilaku kompleks: **behavior study group** (cohort via durable key), **step dimension** (sekuensial), dan **timespan fact table** (dual date/time stamp). Ditutup dengan **partial conformity** (integrasi inkremental), peringatan **fact-to-fact join**, dan **low latency trade-off**. Bab 9 (HR) akan membahas dimensi karyawan & bridge table lanjutan.
