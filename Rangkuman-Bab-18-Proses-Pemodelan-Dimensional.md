# Bab 18 — Dimensional Modeling Process & Tasks

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab ini memerinci **proses kolaboratif merancang model dimensional** bersama perwakilan bisnis — bukan lagi *apa* yang dimodelkan, tapi *bagaimana* menjalankan design session-nya. Mencakup: **persiapan** (peserta, tools, naming convention, jadwal), alur **desain** (bubble chart high-level → detail tabel-per-tabel → SCD → dokumentasi worksheet → issues log → bus matrix), lalu **review & validasi** (IT, core user, broader business), dan **finalisasi dokumentasi**.

---

## 1. Gambaran Proses

Membuat model dimensional adalah proses **sangat iteratif & dinamis**. Setelah persiapan, dimulai dari **model grafis awal dari bus matrix** (menetapkan scope & memperjelas grain), lalu menyelam ke **dimension tables** (definisi atribut, domain values, sumber, relasi, kualitas data, transformasi), baru **fact tables**, dan terakhir **review & validasi** bersama perwakilan bisnis.

**Tujuan utama:** model yang (1) memenuhi kebutuhan bisnis, (2) terverifikasi datanya tersedia untuk populasi, (3) memberi tim ETL **source-to-target mapping** yang solid.

**Input kunci:** preliminary bus matrix + detailed business requirements.
**Deliverable kunci:** high-level dimensional model, detailed dimension & fact table design, **issues log**.

Alur (`Preparation → High Level Model → Detailed Development (Iterate & Test) → Review & Validation → Final Documentation`) tampak linear tapi sangat iteratif — beberapa pass dari high-level ke detail tiap kolom. Durasi tipikal **3-4 minggu** untuk satu proses bisnis (bervariasi menurut pengalaman tim, ketersediaan requirement, keterlibatan steward, kompleksitas sumber, dan **ketersediaan conformed dimension** yang sudah ada).

> ⚠️ Bila memakai expert luar, **haruskan** mereka memfasilitasi bersama tim, **bukan** menghilang lalu kembali dengan desain jadi — agar tim paham desain & trade-off-nya, dan bisa mengerjakan model berikutnya secara mandiri.

---

## 2. Get Organized (Persiapan)

**Identifikasi peserta, terutama perwakilan bisnis.** Model terbaik = hasil **kolaborasi tim**. Data modeler memfasilitasi & bertanggung jawab atas deliverable, tapi **SME bisnis** wajib terlibat (merekalah yang historis tahu cara mengeluarkan data dari sumber). Sertakan orang yang paham realita source system, plus perwakilan DBA/ETL (agar belajar dari insight & **menahan godaan** menerapkan 3NF atau menunda kompleksitas ke aplikasi BI). Ingat tujuan: **tukar kompleksitas ETL demi kesederhanaan & prediktabilitas di lapisan presentasi BI**.

**Data stewardship** — komitmen dimensional = komitmen **conformed dimension**. Program stewardship aktif membantu mencapainya. Tantangan conformed dimension **lebih ke komunikasi & konsensus organisasi** daripada teknis. Kelompok berbeda punya business rule sendiri; steward harus mengembangkan aturan & definisi bersama, lalu membujuk organisasi menerimanya. Kritik "conformed dimension terlalu sulit" itu benar — tapi **itulah inti data terpadu**; bila semua menuntut label sendiri, tak ada "single version of the truth".

**Review business requirements** — familiarkan dengan dokumentasi requirement dulu. Tugas tim modeling: menerjemahkan requirement jadi model dimensional **fleksibel** (mendukung banyak analisis, bukan laporan spesifik). Melewati review = model yang **eksklusif digerakkan source data** tanpa nilai tambah bisnis.

**Tools:**
- **Modeling tool** — mulai dengan **spreadsheet** (mudah diubah saat iterasi); konversi ke modeling tool organisasi saat desain mengeras (bisa forward-engineer ke DB: tabel, index, partisi, view).
- **Data profiling tool** — eksplorasi konten & relasi source **nyata** (bukan dokumentasi usang), dari SQL sederhana hingga tool khusus. Verifikasi data ada dalam kondisi usable (atau cacatnya bisa dikelola).

**Naming convention** — label harus **deskriptif & konsisten** dari perspektif bisnis (nama tabel/kolom = elemen antarmuka BI; "Description" jelas di model tapi tak berarti di laporan). Kompleks karena grup berbeda punya makna berbeda untuk nama sama. Pendekatan umum 3 bagian: **prime word, qualifier, class word**. Manfaatkan fungsi IT bila ada; bila tidak, tetapkan saat modeling.

**Jadwal & fasilitas** — jadwalkan sesi **2-3 jam pagi & sore, 3-4 hari/minggu** (bukan hari penuh — anggota punya tanggung jawab lain; waktu luang untuk riset sumber & update dokumentasi). Ruang konferensi dedikasi + whiteboard besar + flip chart + proyektor (wajib untuk review).

---

## 3. Design the Dimensional Model

**Empat keputusan kunci (dari Bab 3):** (1) identifikasi proses bisnis, (2) deklarasikan grain, (3) identifikasi dimensi, (4) identifikasi fakta. Proses bisnis sudah ditentukan di akhir requirements (prioritisasi Bab 17 memilih baris bus matrix).

### 3.1 Konsensus bubble chart high-level

Buat diagram model dimensional high-level ("**bubble chart**") dari bus matrix. **Jangan** biarkan satu desainer membuatnya sendiri lalu presentasi — libatkan **seluruh tim**. Diagram entity-level ini memperjelas **grain fact table & dimensinya** ke audiens non-teknis.

- **Deklarasi grain** mempertimbangkan yang **dibutuhkan bisnis** & yang **mungkin dari sumber**; harus berakar pada realita data fisik. Satu baris bus matrix bisa jadi **beberapa bubble chart** (tiap fact table beda granularitas).
- Sebagian besar dimensi **muncul alami** setelah grain jelas. Bila dimensi usulan **tak cocok grain** → dimensi dihilangkan, grain diubah, atau pakai **solusi multivalued**.
- Jaga **konsistensi urutan dimensi** antar bubble chart satu proses.

### 3.2 Model detail (tabel-per-tabel)

Rapat rutin mendefinisikan model detail **kolom demi kolom**; perwakilan bisnis tetap terlibat (feedback atribut, filter, grouping, label, metrik). Mulai dari **dimension** lalu **fact**; awali dengan dimensi mudah (**date dimension** favorit) untuk sukses awal & membangun tim.

- **Identifikasi dimensi & atribut** — definisikan **conformed dimension** (harus diterima seantero enterprise). Steward & business analyst kunci untuk konsensus penamaan. Ini **tugas bisnis** menyepakati definisi & nama standar; komite governance mungkin harus turun tangan. Di sini tim juga menimbang **junk/mini-dimension** (sering baru tampak perlu saat mendalami desain).
- **Identifikasi fakta** — deklarasi grain mengkristalkan diskusi (fakta harus **true to grain**). Data profiling mengidentifikasi count & amount dari sumber; boleh tambah **derived facts** yang diinginkan bisnis.
- **Identifikasi teknik SCD** — untuk **tiap atribut dimensi**, definisikan bagaimana perubahan source direfleksikan (Type 0-7). Input steward kritis; tanya source expert apakah perubahan = koreksi data.

### 3.3 Dokumentasi, issues log, bus matrix

- **Design worksheet** (deliverable utama detail) — satu worksheet per tabel: nama atribut/fakta, deskripsi, sample values, indikator **SCD type** tiap atribut dimensi; untuk fakta: **FK relationship**, degenerate dimension, dan aturan **additive/semi-additive/non-additive**. Ini **langkah pertama** menuju source-to-target mapping (tim physical design melengkapi nama fisik, datatype, key).
- **Issues log** — tangkap semua isu/definisi/aturan transformasi/tantangan kualitas data; tunjuk penanggung jawab (sering project manager); review di akhir tiap sesi.
- **Bus matrix ter-update** — penemuan baru sering memunculkan fact/dimension baru atau pemecahan/penggabungan dimensi. Jaga bus matrix tetap mutakhir (alat komunikasi & perencanaan kunci; versi detail menangkap granularitas & metrik tiap fakta).

---

## 4. Review, Validasi & Finalisasi

Setelah tim percaya diri, masuk fase **review & validasi** dengan pihak lain:

**IT review** — biasanya pertama; peserta akrab dengan proses bisnis target (mereka menulis/mengelola sistemnya). ⚠️ Menantang karena banyak yang **3NF modeler** & cenderung menerapkan aturan OLTP. **Berikan edukasi dimensional dulu** (jangan habiskan waktu berdebat disiplin). Mulai dari **bus matrix** (scope, conformed dimension, prioritas) → high-level diagram → detail worksheet → open issues.

**Core user review** — sering **tak perlu** bila core user sudah anggota tim modeling. Bila perlu, mirip IR review (core user lebih teknis). Di organisasi kecil, sering digabung dengan IT review.

**Broader business user review** — **sama banyak edukasi & review desain**. Edukasi tanpa membanjiri, sambil menunjukkan bagaimana model mendukung requirement mereka. Mulai bus matrix → bubble chart → dimensi kritis (customer, product); bisa dilengkapi diagram **drill path hierarki**. Alokasikan waktu menunjukkan model menjawab beragam pertanyaan (ambil contoh dari requirement document).

**Finalisasi dokumentasi** — kompilasi dari working papers: deskripsi proyek, high-level data model diagram, detailed design worksheet tiap tabel, open issues.

---

## Ringkasan Bab

Bab 18 memerinci **cara menjalankan** proyek pemodelan dimensional: proses **iteratif kolaboratif** (3-4 minggu per proses) dengan perwakilan bisnis & data steward. **Persiapan** mencakup peserta yang tepat (SME bisnis + DBA/ETL yang menahan godaan 3NF), komitmen **conformed dimension/stewardship**, review requirement, tools (spreadsheet → modeling tool, **data profiling**), **naming convention** (prime-qualifier-class), dan jadwal 2-3 jam. **Desain** mengikuti empat langkah: konsensus **bubble chart** high-level (grain berakar realita sumber) → model detail tabel-per-tabel (dimensi dulu, date favorit; conformed dimension = tugas bisnis; SCD per atribut) → **design worksheet** (source-to-target awal), **issues log**, **bus matrix ter-update**. Ditutup **review berlapis** (IT dengan edukasi dimensional dulu, core user, broader business dengan drill path) dan finalisasi dokumentasi. Bab 19 masuk ke **34 subsystem ETL**.
