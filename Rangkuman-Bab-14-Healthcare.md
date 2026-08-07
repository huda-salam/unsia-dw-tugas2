# Bab 14 — Healthcare (Layanan Kesehatan)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Layanan kesehatan menawarkan banyak contoh desain dimensional yang kaya. Bab ini menautkan data **administratif & klinis**, dengan sorotan pada **accumulating snapshot** (workflow klaim billing/pembayaran) yang punya **role-playing date masif**, **multivalued diagnosis** via **bridge table** (tanpa weighting → impact report), **supertype/subtype** untuk charges, **measure type dimension** untuk fakta sparse (lab), serta penanganan **text comment**, **image**, **inventory fasilitas**, dan **perubahan retroaktif** (late arriving data).

---

## 1. Konteks: Bus Matrix Kesehatan

Konsorsium kesehatan punya proses klinis (physician orders, medications, lab test, disease management, patient outcomes/satisfaction), billing/revenue (inpatient/outpatient charges, claims billing, claims payment, collections/write-offs), dan operasional (bed/facility utilization, supply procurement/utilization, workforce scheduling). Dimensi bersama: date, responsible party, employer, health plan, payer (primary & secondary), physician, procedure, equipment, lab test, medication, diagnosis, facility.

---

## 2. Accumulating Snapshot: Claims Billing & Payment Workflow

Proses billing klaim **berumur pendek** dengan milestone standar → **accumulating snapshot** (bukan periodic snapshot yang untuk time series panjang, atau transaction yang untuk detail).

- **Grain = satu baris per baris (line) pada tagihan medis** — merepresentasikan **histori terakumulasi** line item dari pembuatan hingga status kini; baris **di-update** saat apa pun berubah.
- **Delapan FK tanggal** milestone: treatment, primary/secondary insurance billing, responsible party billing, last primary/secondary/responsible party payment, zero balance. Semua **role-playing date**; tanggal yang belum diketahui menunjuk baris date **"To Be Determined"** (surrogate key tak boleh null).
- **Dimensi:** patient, physician, physician organization, procedure, facility, primary diagnosis, primary & secondary insurance organization (2 role), responsible party, employer, master bill ID (DD).
- **Fakta:** billed amount, primary/secondary insurance paid, responsible party paid, total paid (calculated), sent to collections, written off, unpaid balance (calculated), length of stay, dan **lag** (bill→initial payment tiap pihak, bill→zero balance).
- **Alur:** baris dibuat saat charge diterima; awalnya 7 tanggal terakhir = TBD. Seiring pembayaran masuk, **baris sama di-update destruktif**. ⚠️ Update destruktif menantang DBA — bila baris menstabil, **reorganisasi fisik** memulihkan storage; **partisi pada treatment date key** terjaga (treatment date tak berubah). Skenario "berantakan" (banyak pembayaran per line, atau satu pembayaran untuk banyak klaim) butuh **companion transaction schema** — accumulating snapshot hanya membingkai alur standar.

**Role-playing** — ciri khas accumulating snapshot: 8 FK tanggal join ke **8 view** dari satu date dimension (label kolom dibedakan). Dimensi lain juga berperan (payer; physician sebagai referring/attending/consulting/assisting).

---

## 3. Multivalued Diagnosis (Bridge Table)

Saat prosedur/lab, pasien punya **satu atau lebih diagnosis** (rekam medis elektronik memungkinkan banyak diagnosis; pasien lansia rawat inap bisa punya **20 diagnosis** sekaligus).

**Mengapa bukan FK jamak?** Diagnosis **tidak berperilaku seperti role independen**, jumlahnya sering >3, dan FK jamak membuat BI tak efisien (query tak tahu slot mana untuk di-constrain).

**Solusi:** ganti diagnosis FK dengan **Diagnosis Group Key** yang di-join many-to-many ke **Diagnosis Group Bridge** (satu baris per diagnosis dalam grup). Pasien dengan 3 diagnosis → satu diagnosis group + 3 baris bridge.

- **Tanpa weighting factor** — berbeda dari Bab 10, bobot dampak tiap diagnosis pada treatment/tagihan **nyaris mustahil** ditetapkan. Tanpa bobot, analisis diagnosis fokus pada **impact question** ("total billed untuk prosedur yang melibatkan diagnosis gagal jantung kongestif?") — analis paham ini bisa **overcounting**.
> Weighting factor itu **elegan tapi tidak wajib**. Bila komunitas bisnis tak sepakat/antusias → tinggalkan. Dengan >1 dimensi multivalued, jangan repot memutuskan interaksi banyak bobot.
- **Varian PK-relationship** — bila modeling tool menuntut relasi FK→PK proper, sisipkan **Diagnosis Group Dimension** (PK = diagnosis group) di antara fakta & bridge (biasanya tanpa info baru, kecuali ada label cluster diagnosis) → semua join jadi many-to-one.
- **Portofolio grup** — jangan buat grup unik per encounter (baris meledak, banyak identik); pakai **master diagnosis group** yang di-*lookup* saat ETL (temukan → pakai; tidak → buat baru).
- **Inpatient evolving** — bila diagnosis berkembang selama rawat inap → grup unik per pasien + **dua date stamp** (begin/end) untuk change tracking.

---

## 4. Supertype/Subtype untuk Charges

Charges kesehatan mengikuti pola **supertype/subtype** (Bab 10): facility charge (inpatient) beda dari professional charge (outpatient). Untuk fokus rawat inap, spesialisasikan skema:
- **Dua role physician:** admitting & attending physician (+ organisasi masing-masing, karena bisa beda organisasi). Untuk bedah kompleks (mis. transplantasi jantung) dengan tim spesialis: satu FK **primary responsible physician**, sisanya via **group key ke bridge multivalued**.
- **Dua diagnosis group multivalued:** **admitting** diagnosis group (ditentukan di awal, sama untuk semua baris treatment dalam satu rawat inap) & **discharge** diagnosis group (baru diketahui saat pulang).

**Electronic Medical Records (EMR)** — pergeseran dari kertas ke elektronik menantang DW karena **variabilitas & volume ekstrem** (numerik, teks bebas, gambar/foto). Kandidat klasik *big data*.

---

## 5. Measure Type Dimension untuk Fakta Sparse

Untuk variabilitas hasil lab: **measure type dimension** yang menjelaskan arti fakta generik (unit of measure, batasan additivity) — grain jadi **"satu baris per pengukuran per event"**.

- **Kelebihan:** sangat fleksibel (tambah tipe = tambah baris, bukan ubah struktur fakta); hilangkan null (baris ada hanya bila pengukuran ada).
- **Trade-off:** **banyak baris** (10 pengukuran lab = 10 baris, bukan 1). Cocok untuk situasi **sangat sparse** (lab klinis, uji manufaktur). ⚠️ Saat densitas naik → terlalu banyak baris → **kembali** ke desain klasik kolom tetap. Juga menyulitkan kalkulasi antar-pengukuran (SQL suka aritmetika **dalam** baris, bukan lintas baris; hati-hati mencampur amount inkompatibel di satu kolom). **OLAP cube lebih toleran** menghitung lintas measure type.

---

## 6. Text Comment, Image, Inventory, Retroaktif

**Freeform text comment** (mis. catatan klinis) — kurang bertenaga analitik kecuali di-parse, tapi pengguna enggan membuangnya. ⚠️ **Jangan** simpan di fact table (boros, jarang di-query; **bukan** degenerate dimension — itu hanya untuk control number). Simpan di **comment dimension** terpisah (bila banyak "No Comment" berulang → kardinalitas < transaksi) atau atribut **transaction event dimension** (bila hampir unik per event). Join lambat, tapi pengguna biasanya sudah memfilter ketat saat drill ke komentar.

**Image** — trade-off: simpan **nama file JPEG** di fakta (program lain bisa akses bebas, tapi harus sinkronkan database file grafis terpisah) vs **embed blob** langsung di database.

**Facility/equipment inventory** — seperti Bab 4: **periodic snapshot** (status tiap bed di titik reguler — factless, FK patient/physician/nurse), **transaction** (satu baris per gerakan masuk/keluar bed; operating room: pre-op/post-op/downtime + durasi), atau **timespan fact table** (Bab 8, effective/expiration, bila perubahan tak volatil — mis. rehab/eldercare).

**Perubahan retroaktif (late arriving data)** — di kesehatan (terutama sistem legacy) **lazim**: data prosedur beberapa minggu lalu, update profil pasien back-dated berbulan-bulan. Makin terlambat, makin menantang ETL. Ini bisa jadi **mode dominan**, bukan kasus outlier (ditangani di Bab 19).

---

## Ringkasan Bab

Bab 14 (kesehatan) menautkan data administratif & klinis. Sorotan: **accumulating snapshot** untuk **workflow claims billing/payment** (grain per line klaim, 8 role-playing date, update destruktif, fakta lag, TBD date) — dilengkapi companion transaction untuk skenario berantakan. **Multivalued diagnosis** ditangani **bridge table** (diagnosis group key; **tanpa weighting** → impact report yang bisa overcount; varian PK-relationship; portofolio grup via ETL lookup; twin date untuk inpatient evolving). **Supertype/subtype charges** (admitting/attending physician role, admitting/discharge diagnosis group). **Measure type dimension** untuk fakta **sparse** (fleksibel & bebas null, tapi banyak baris & sulit kalkulasi lintas baris — OLAP lebih toleran). Ditutup dengan **text comment** (comment/transaction dimension, bukan fakta), **image** (filename vs blob), **inventory fasilitas** (snapshot/transaction/timespan), dan **late arriving data** (mode dominan di kesehatan). Bab 15 (E-Commerce) akan membahas clickstream & pola data web.
