# Bab 11 — Telecommunications (Telekomunikasi)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab ini **berbeda formatnya**: alih-alih membangun skema dari nol, ia mengajak melakukan **design review** atas draf skema billing telekomunikasi. Isinya menjadi **checklist kesalahan desain dimensional yang umum** (rekap seluruh prinsip buku) + **panduan praktis menjalankan design review** + teknik **geographic location dimension**. Sangat berguna sebagai daftar periksa untuk menilai skema Anda sendiri.

---

## 1. Konteks: Studi Kasus & Latihan Review

Perusahaan telekomunikasi punya banyak proses (channel inventory, service activation, product sales, promotion, call detail traffic, customer billing, support calls, repair work orders). Bab menampilkan draf skema **billing** yang "cacat" dan mengajak pembaca menemukan kesalahannya sebelum membaca pembahasan — sebuah alat belajar efektif.

---

## 2. Checklist Kesalahan Desain Dimensional (Inti Bab)

**Seimbangkan kebutuhan bisnis & realita sumber.** Model yang **hanya** dari requirement memuat elemen yang tak bisa di-source; model yang **hanya** dari analisis data sumber melewatkan elemen kritis bagi analitik. Rancang dari **paduan** keduanya (profil data sumber sambil kumpulkan requirement).

**Fokus pada proses bisnis, bukan laporan.** Model harus mencerminkan **event proses bisnis**, bukan dirancang untuk satu laporan/pertanyaan spesifik (yang cepat usang saat format berubah). Model process-centric lebih tahan perubahan & melayani banyak departemen. Skema komplementer (agregat, accumulating snapshot, consolidated, subset) datang **setelah** proses inti.

**Granularity.** Pertanyaan **pertama** di review: "Apa grain fact table?" — sering dijawab tak konsisten oleh tim. Deklarasikan grain jelas & disepakati. Bangun di **level terendah** (fleksibilitas maksimum). "Terendah" = paling detail untuk **proses yang dipilih**, bukan sekadar yang tersedia di enterprise.

**Single granularity untuk fakta.** Semua fakta aditif harus pada grain **seragam**. ⚠️ Hindari total "to-date" (YTD) di fact row — **tidak aditif sempurna**, mengundang double counting saat >1 tanggal muncul di hasil. Larang juga **field teks/flag kriptik** di fact table (boros ruang + pengguna ingin query/filter atasnya) → taruh di dimensi.

**Dimension granularity & hierarki.** Tiap dimensi & atribut **bernilai tunggal** untuk satu baris. Relasi many-to-one → **kolapskan/denormalisasi** dalam satu dimensi. Normalisasi cocok untuk **transaksi** (OLTP) & boleh di **ETL**, tapi merusak tujuan kembar (understandability + performa) di area presentasi. Hindari:
- **Centipede fact table** — FK terpisah untuk elemen hierarki (brand, category) → puluhan join. Bila FK > ~20, gabungkan/kolapskan.
- **Snowflake** — normalisasi hierarki jadi tabel terpisah; hemat disk tak signifikan, merugikan usability/performa. **Outrigger** hanya pengecualian (jangan disalahgunakan).
- **Dimensi campuran atomic+rollup** (baris produk & brand di satu tabel dengan atribut "level") — pola lama pra-aggregate-navigation; berisiko **overcounting** bila level indicator tak di-constrain tiap query.

**Date dimension.** Setiap fact table butuh ≥1 FK ke **date dimension eksplisit** yang bermakna (jangan generic yang tak jelas merujuk apa). ⚠️ **Jangan** pakai *fixed time series bucket* (12 kolom bulan dalam satu baris) — tak fleksibel (penuh → pilihan buruk), memindah atribut tanggal ke aplikasi, boros null. Gunakan **baris terpisah** di fact table.

**Degenerate dimension.** Nomor transaksi (invoice/order) = **degenerate dimension** (di fact table). ⚠️ Jangan buat tabel dimensi untuk nomor transaksi (memuat tanggal/customer dari header) — tanggal & customer harus jadi **FK di fakta**. **Tanda bahaya:** dimensi dengan baris (hampir) sebanyak fact table = ada degenerate dimension tersembunyi.

**Surrogate key.** PK dimensi = surrogate key; satu-satunya pengecualian = date dimension (stabil/prediktabel).

**Decode & deskripsi.** Semua kode/ID di dimensi harus disertai **decode deskriptif** (nilai yang ingin dilihat pengguna di laporan & menu). Pengguna **tidak** suka kode (lihat daftar decode yang menempel di monitor mereka). Simpan decode sebagai **data** (bukan di semantic layer BI) → konsisten lintas tool, data-driven.

**Conformity commitment.** Tim **wajib** berkomitmen memakai **conformed dimension** lintas model process-centric — kritis untuk konsistensi & integrasi; tanpanya, stovepipe terus berulang. Berlaku baik untuk arsitektur Kimball maupun hub-and-spoke.

---

## 3. Panduan Menjalankan Design Review

**Persiapan:**
- **Undang pemain yang tepat** — tim modeling + perwakilan BI + orang yang **sangat paham bisnis**; tapi jangan 25 orang.
- **Tunjuk fasilitator** (netral atau terlibat, sesuai dinamika) untuk menjaga fokus.
- **Sepakati scope**, **blok waktu** (biasanya 2 hari penuh, semua hadir penuh — jangan keluar-masuk), **ruang tepat** (whiteboard besar), **beri PR** (tiap orang kirim 5 kekhawatiran utama sebelumnya → cegah *group think*).

**Saat review:**
- **Tinggalkan sikap defensif** & teknologi (laptop/HP) di pintu.
- Pastikan **pemahaman bersama** atas model saat ini (dedikasikan jam pertama).
- Tunjuk **scribe**. **Mulai dari big picture** (bus matrix → proses → grain → dimensi; "kupas bawang"). Jangan tunda hal sulit ke sore hari kedua.
- Ingatkan **penerimaan bisnis** = ukuran sukses akhir. **Sketsa baris contoh dengan nilai data**. **Tutup dengan rekap** (tugas & tenggat jelas).

**Setelah review:** tetapkan penanggung jawab isu terbuka, evaluasi cost/benefit tiap perbaikan (buat action plan), dan **rencanakan review ulang tiap 12–24 bulan** (perubahan desain = tanda sukses, bukan gagal).

---

## 4. Geographic Location Dimension

Industri telko/utilitas punya notasi lokasi presisi (street, city, state, ZIP, **latitude/longitude** untuk geospatial & visualisasi peta).

- Godaan: **satu master location table** yang di-outrigger ke semua entitas (service line, equipment, network, real estate, customer, dst.). **Standardisasi alamat memang berharga — tapi itu tugas ETL back room**, tak perlu dipapar ke pengguna.
- **Informasi geografis lebih baik jadi atribut di dalam banyak dimensi**, bukan satu location dimension/outrigger standalone. Biasanya **sedikit tumpang tindih** antar-lokasi di berbagai dimensi; konsolidasi ke satu dimensi justru membayar **harga performa**.
- **Hindari dimensi abstrak generik** (mis. "generalized location") di area presentasi — merusak usability & performa. Abstraksi lebih pas di **ETL back room**.

---

## Ringkasan Bab

Bab 11 memakai kasus **billing telekomunikasi** sebagai latihan **design review**, menghasilkan **checklist kesalahan desain dimensional** yang merangkum seluruh prinsip buku: seimbangkan requirement vs sumber, fokus proses bisnis (bukan laporan), deklarasi **grain** yang jelas & seragam, larang YTD/teks di fakta, kolapskan hierarki (hindari **centipede**, **snowflake**, dimensi campuran atomic+rollup), pakai **date dimension eksplisit** (bukan fixed bucket), **degenerate dimension** untuk nomor transaksi, **surrogate key**, **decode deskriptif** (bukan kode), dan **komitmen conformed dimension**. Ditambah **panduan praktis menjalankan design review** (persiapan, taktik, tindak lanjut) dan penanganan **geographic location** (atribut di banyak dimensi, standardisasi di ETL, hindari dimensi abstrak generik). Bab 12 (Transportation) akan membahas fact table dengan multiple role-playing & lebih banyak pola perjalanan.
