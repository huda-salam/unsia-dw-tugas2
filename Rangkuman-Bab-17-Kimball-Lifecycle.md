# Bab 17 — Kimball DW/BI Lifecycle Overview

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab ini beralih dari **teknik pemodelan** ke **metodologi proyek** — peta jalan **Kimball Lifecycle** untuk membangun sistem DW/BI dari awal hingga pemeliharaan & pertumbuhan. Prinsip inti (sejak 1998, dulu "Business Dimensional Lifecycle"): **fokus pada kebutuhan bisnis, sajikan data terstruktur dimensional, kerjakan proyek iteratif yang terkelola**. Bab memandu tiap tahap: perencanaan, definisi kebutuhan bisnis, tiga jalur paralel (teknologi, data, aplikasi BI), deployment, pemeliharaan, dan **10 pitfall** yang harus dihindari.

---

## 1. Peta Jalan Lifecycle

Diagram Kimball Lifecycle menggambarkan **urutan tugas, dependensi, dan konkurensi** (bukan timeline absolut — kotak berukuran sama tapi effort sangat berbeda). Alurnya:

1. **Program/Project Planning** — menilai kesiapan, menetapkan scope & justifikasi awal, mendapatkan sumber daya, meluncurkan. Project management berjalan sebagai fondasi sepanjang lifecycle.
2. **Business Requirements Definition** — ada **panah dua arah** dengan planning (saling memengaruhi). Menyelaraskan DW/BI dengan kebutuhan bisnis **mutlak krusial** — teknologi terbaik pun tak menyelamatkan lingkungan yang gagal fokus ke bisnis.
3. **Tiga jalur paralel** yang lahir dari requirements:
   - **Teknologi** — technical architecture design → product selection & installation. ⚠️ Product selection **bukan** kotak pertama; kesalahan pemula = pilih produk sebelum paham tujuan.
   - **Data** — dimensional modeling → physical design → ETL design & development (effort ETL jauh lebih besar dari physical design).
   - **Aplikasi BI** — BI application design → development. Proyek **belum selesai saat data terkirim**; aplikasi BI (template parameter-driven) memenuhi sebagian besar kebutuhan analitik.
4. **Deployment** — ketiga jalur konvergen (+ edukasi & support).
5. **Maintenance** & **Growth** — pertumbuhan memulai proyek berikutnya, kembali ke awal lifecycle.

**Hubungan dengan agile:** Lifecycle & agile berbagi doktrin (fokus nilai bisnis, kolaborasi, inkremental), tapi Kimball menekankan **fondasi arsitektur data & governance** (bus architecture) dan sering **membundel beberapa deliverable agile** jadi rilis fungsional penuh sebelum deploy luas.

---

## 2. Program/Project Planning

**Menilai kesiapan (3 faktor, urut kepentingan):**
1. **Sponsor bisnis kuat** — punya visi dampak, rekam jejak sukses, cerdas politik. Berisiko bila CIO yang jadi sponsor; lebih baik mitra bisnis.
2. **Motivasi bisnis yang kuat & mendesak** — menyelesaikan masalah kritis (dari eksternal seperti kompetisi, atau internal seperti ketakmampuan analisis lintas organisasi pasca-akuisisi).
3. **Feasibility** — terutama **data feasibility**: apakah sudah mengumpulkan data operasional nyata yang bersih pada granularitas tepat? Tak ada solusi jangka pendek bila belum.

**Scoping & justifikasi** — scope harus **bermakna bagi bisnis & terkelola bagi IT**; mulai dari **satu proses bisnis** (simpan lintas-proses untuk fase lanjut). Hindari **"Law of Too"** (terlalu banyak sumber/user/lokasi/kebutuhan dalam waktu terlalu singkat). Justifikasi berbasis **peningkatan revenue/profit**, bukan sekadar "single version of the truth". Bila sulit dijustifikasi → gejala sponsor/masalah yang salah.

**Staffing** — tim lintas-fungsi (bisnis & IT); satu orang bisa mengisi banyak peran. Peran bisnis: sponsor, driver, lead, users. Straddler: business analyst, data steward, BI application designer. Peran IT: project manager, technical architect, data architect/modeler, DBA, metadata coordinator, ETL architect, ETL developer.

**Rencana** — anggota kunci membuat estimasi effort sendiri; tetapkan **acceptance checkpoint** tiap milestone; strategi komunikasi luas; tangani **scope creep** (tambah scope, zero-sum, atau tolak-sebagai-enhancement — jangan diputuskan dalam "vakum IT").

---

## 3. Business Requirements Definition

**Preplanning:**
- **Pilih forum** — wawancara (partisipasi individual, mudah dijadwalkan) vs facilitated session (waktu lebih singkat, komitmen lebih besar). **Survei tidak disarankan** (datar, tak bisa menggali). Kimball pakai **hybrid**: wawancara untuk detail, fasilitasi untuk konsensus. **Jangan** tanya user soal granularitas/dimensionalitas — tanya **apa yang mereka lakukan, mengapa, bagaimana mengambil keputusan**.
- **Siapkan tim** — lead interviewer (pertanyaan terbuka) + scribe (catat rinci); tanpa perekam (mengubah dinamika); riset dulu (annual report, org chart); siapkan **kuesioner** (alat bantu, bukan skrip).
- **Pilih & jadwalkan perwakilan** — cakup organisasi **horizontal** (untuk bus matrix) & **vertikal** (jangan hanya power analyst; sertakan eksekutif & middle management). Individu untuk eksekutif; grup 2-3 untuk level bawah. 1 jam individu, 1,5 jam grup, jeda 0,5 jam; maks 3-4 sesi/hari. Minta bawa laporan kunci.

**Mengumpulkan requirements** — alur: launch (kickoff business-centric) → **interview flow** (mulai dari tanggung jawab, lalu metrik kinerja kunci → langsung ke proses bisnis & fakta; probe dimensionalitas; tanya jenis analisis) → **wrap-up** (kriteria sukses **terukur**; disclaimer bahwa tak semua masuk fase 1). **Data-centric interview** diselingi untuk cek feasibility data sebelum momentum terbentuk.

**Dokumentasi** — debrief segera; tulis temuan. Dokumen paling penting: **consolidated findings** terorganisasi per **proses bisnis** (executive summary, overview, lalu per proses: mengapa dianalisis, kapabilitas diinginkan, keterbatasan kini, manfaat, feasibility). Bisa pakai **opportunity/stakeholder matrix** (baris = proses, kolom = grup organisasi).

**Prioritisasi** — pakai **prioritization grid** (sumbu vertikal = **business impact**, horizontal = **feasibility**). Mulai dari **kanan-atas** (high impact + high feasibility). Hindari **kiri-bawah** (mission impossible). **Kanan-bawah** (feasible tapi tak krusial) tak layak segera. **Kiri-atas** (peluang besar tapi belum feasible) → tim IT lain atasi keterbatasan feasibility-nya. **Jangan** putuskan dalam vakum — libatkan bisnis.

---

## 4. Tiga Jalur Paralel

**Technology Track.** *Technical architecture design* = blueprint layanan & infrastruktur (seperti cetak biru rumah: tangkap masalah di atas kertas, koordinasi paralel, reuse modular, alat **komunikasi**). Dua ekstrem yang tak sehat: melewati arsitektur (rebuild kemudian) vs 2 tahun mendesain arsitektur (lupa tujuan = selesaikan masalah bisnis). Proses **8 langkah**: task force → kumpulkan requirements arsitektur → dokumentasi → buat model → tentukan fase → desain subsystem → buat plan → review & finalisasi. *Product selection* (6 langkah): pahami proses purchasing → matriks evaluasi berbobot → riset pasar (hindari RFP bila bisa) → shortlist → prototype bila perlu (maks 2 produk) → pilih, trial, negosiasi (komitmen privat dulu untuk jaga posisi tawar).

**Data Track.** *Dimensional modeling* (Bab 18 detail). *Physical design*: logical & physical mirip (jangan biarkan DBA menormalisasi skema); standar penamaan; model fisik (+ staging/audit/security tables); **index plan** (dimension: bitmap pada atribut filter; fact: B-tree/clustered pada PK, date FK di posisi terdepan); **agregasi** (lebih hemat dari menambah hardware; berbasis pola akses & distribusi data); **partisi** (fact besar per activity date/bulan). *ETL design & development* (Bab 19-20; 34 subsystem).

**BI Applications Track.** "Bagian yang menyenangkan." Meski ada yang ingin lingkungan ad hoc penuh, **template parameter-driven** memenuhi sebagian besar kebutuhan (bagi banyak user, "ad hoc" = ubah parameter laporan). *Specification*: identifikasi **10-15 starter report**; tetapkan standar (menu, look-and-feel); navigasi terstruktur; portal/dashboard. *Development*: standar (naming, kalkulasi, library); mulai setelah desain DB selesai, tool & metadata terpasang, subset data historis termuat; **mulai sebelum ETL "selesai"** (developer menemukan masalah data & menguji response time lebih awal); QA butuh data stabil.

---

## 5. Deployment, Maintenance & Growth

**Deployment** — konvergensi butuh **preplanning** & keberanian menilai kesiapan jujur. Analogi: **data = hidangan utama**, "dimasak" di dapur ETL (paling tak terprediksi). ⚠️ Menyajikan data "setengah matang" demi tenggat membuat user enggan kembali. Lakukan **end-to-end system testing** (data QA, operasi, performa, usability) + kemas dengan **edukasi & support** (tiered: self-service → power user → tim DW/BI).

**Maintenance & Growth** — pekerjaan **belum selesai** setelah deploy. Investasikan di: **support** (krusial segera; relokasi ke komunitas bisnis; jujur saat ada masalah), **edukasi** (refresher/advanced/introductory berulang), **technical support** (perlakukan sebagai produksi dengan SLA; monitor proaktif), **program support** (pasarkan sukses, checkpoint review). **Growth = tanda sukses, bukan gagal** — libatkan bisnis dalam prioritisasi (prioritization grid), bentuk komite sponsorship eksekutif, lalu kembali ke awal lifecycle.

---

## 6. 10 Pitfall yang Harus Dihindari

Sepuluh kesalahan mematikan (satu saja bisa menjatuhkan inisiatif):
10. Terlalu terpesona teknologi & data alih-alih fokus kebutuhan bisnis.
9. Gagal merekrut sponsor bisnis (visioner senior yang berpengaruh & wajar).
8. Menggarap proyek "galaktik" bertahun-tahun alih-alih iteratif terkelola.
7. Habiskan energi membangun struktur ternormalisasi, kehabisan budget sebelum area presentasi dimensional.
6. Perhatikan performa back room & kemudahan pengembangan lebih dari performa query & kemudahan pakai front room.
5. Buat area presentasi yang seharusnya queryable jadi terlalu kompleks.
4. Populasikan model dimensional standalone tanpa arsitektur data (conformed dimension).
3. Muat **hanya data teragregasi** ke area presentasi (bukan atomik).
2. Anggap bisnis, kebutuhan, data, dan teknologi bersifat **statis**.
1. Abaikan bahwa **sukses DW/BI terikat langsung pada penerimaan bisnis** — bila user tak menerima sistem sebagai fondasi keputusan, semua sia-sia.

---

## Ringkasan Bab

Bab 17 memberi tur **Kimball Lifecycle**: dimulai dari **program/project planning** (menilai kesiapan — sponsor, motivasi, data feasibility; scoping satu proses; hindari "Law of Too"), **business requirements definition** (wawancara hybrid, tanya *apa & mengapa*, consolidated findings per proses, **prioritization grid** impact×feasibility), lalu **tiga jalur paralel**: teknologi (arsitektur 8 langkah + product selection 6 langkah), data (dimensional modeling → physical design → ETL), dan aplikasi BI (10-15 starter template, mulai sebelum ETL "selesai"). Ketiganya konvergen di **deployment** (data = hidangan utama; jangan sajikan setengah matang), diikuti **maintenance & growth** (growth = tanda sukses). Ditutup **10 pitfall** — puncaknya: sukses DW/BI = **penerimaan bisnis**. Bab 18 memerinci proses kolaboratif merancang model dimensional.
