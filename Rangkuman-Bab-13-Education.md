# Bab 13 — Education (Pendidikan)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Studi kasus universitas yang berfokus pada **dua konsep utama**: **accumulating snapshot fact table** (melacak pipeline aplikasi/hibah dengan milestone) dan **factless fact table** (event + coverage). Ditambah penanganan **multiple course instructor** (bridge), **term dimension** yang conform ke date, serta perlakuan **"apa yang tidak terjadi"** (explicit rows vs OLAP).

---

## 1. Konteks: Bus Matrix Universitas

Universitas punya banyak proses lintas *student lifecycle* (applicant pipeline, financial aid, enrollment, housing, course registration, evaluasi, aktivitas, career placement, advancement/pledges), *financial* (budgeting, endowment, GL, payroll, procurement), *employee* (headcount, hiring, benefits, faculty appointment, research pipeline & expenditure, publikasi), dan *facilities* (utilization, energy, work orders). Bab memilih menyoroti pola yang paling khas pendidikan.

---

## 2. Accumulating Snapshot: Applicant Pipeline

Pipeline penerimaan mahasiswa cocok untuk **accumulating snapshot** — proses **berumur pendek** dengan **awal-akhir jelas** dan **milestone standar**.

- **Grain = satu baris per pelamar** (di-update seiring bergerak lewat milestone).
- **Banyak FK tanggal** (inquiry, campus visit, application submitted/completed, admit, enroll, dst.) sebagai **role-playing date**, dengan surrogate key default untuk tanggal yang belum diketahui (baris baru/in-process). Menganalisis **pace** & mengidentifikasi **bottleneck** (penting bila ada lag pada kandidat yang diincar).
- **Applicant dimension** kaya: geografi, kredensial (GPA, skor SAT/ACT, AP credit, high school), gender, DOB, ethnicity, intended major, application source, dll. → menyesuaikan strategi admissions per tahap.
- **Fakta** = aneka count (inquiry, visit, submitted, admit early/regular, waitlist, defer, deny, enroll, withdraw); bila ada, tambahkan **estimasi probabilitas** apply & enroll (memprediksi yield).

**Keterbatasan & alternatif:** karena baris accumulating snapshot **di-update**, ia **tidak melestarikan** count/status pada titik-titik kritikal (mis. tanggal notifikasi early decision). Bila angka itu penting: **retain snapshot** di beberapa cut-off date, atau bangun **admission transaction fact table** (satu baris per transaksi per pelamar) untuk perbandingan antar-periode.

**Research grant proposal pipeline** — contoh accumulating snapshot lain: melacak lifecycle proposal hibah (preliminary → approval → award), menganalisis jumlah proposal per tahap per faculty/department/topik/funding source, dan success rate.

---

## 3. Factless Fact Table

Sebagian proses bisnis punya fact table **tanpa fakta terukur** — hanya perpotongan key dimensi. Dua tipe:

### 3.1 Event tracking

**Admissions event attendance** — melacak kehadiran calon mahasiswa di event (kunjungan SMA, college fair, alumni interview, campus overnight). Dimensi: event date, planned enroll term, applicant, application status, admissions officer, admission event. Fakta: `Attendance Count = 1`.

**Course registration** — **grain = satu baris per course terdaftar per student per term**. Dimensi: term, student, course, instructor, dst.

**Term dimension** — data di level **term** (bukan hari/minggu/bulan). Term dimension **harus conform** ke calendar date: tiap tanggal di date dimension mengidentifikasi term (Fall), term+academic year (Fall 2013), academic year (2013-2014) — label & nilai kolom identik.

**Menghitung di factless table** — bisa `COUNT(term_key)` (atau key apa pun; SQL menghitung jumlah key yang lewat, bukan nilai distinct — pakai `COUNT DISTINCT` untuk unik). Untuk kejelasan, sering ditambahkan **fakta buatan `registration_count = 1`** (bukan "dummy"): SQL lebih terbaca (`SUM(registration_count)`), memudahkan BI tool, **diperlukan** bila membangun aggregate table di atasnya, dan lazim di **OLAP cube** (count eksplisit karena join key tak terlihat di cube). Bila kelak muncul fakta terukur (tuition revenue, earned credit hours, grade) yang **konsisten dengan grain** → tambahkan (tabel tak lagi factless).

### 3.2 Coverage tracking

**Facility utilization** — memahami fasilitas mana dipakai untuk apa tiap jam. **Insert satu baris per fasilitas per blok jam per hari per term — terlepas dipakai atau tidak.** Dimensi: term, day of week, time-of-day hour, facility (building, type, sqft, capacity, amenities), **owner department** & **assigned department** (2 role-playing), utilization status (Available/Utilized). Fakta: `Facility Count = 1`. Coverage table menyediakan denominatornya (semua slot yang mungkin) untuk menghitung occupancy rate.

**Student attendance** — **grain = satu baris per student yang masuk kelas per hari** (level **calendar date**, bukan term). Menjawab: kursus mana paling ramai, atrisi kehadiran terkecil, dll.

---

## 4. Multiple Course Instructor

Bila kursus **co-taught**, instructor jadi atribut multivalued pada grain. Opsi:
- **Ubah grain** jadi per instructor per registrasi — grain tak natural, rawan **overstatement** registration count. (Kurang disarankan.)
- **Bridge table** dengan instructor group key (di fakta atau outrigger course dimension): satu baris per instructor solo, dua baris per tim. Bisa diberi **weighting factor** (Bab 10) bila alokasi beban jelas — tapi rawan isu overstatement bridge.
- **Konkatenasi nama** instructor jadi satu atribut terdelimit (Bab 9) — mudah untuk label laporan, tapi **tak** mendukung analisis per karakteristik instructor.
- **Primary instructor** sebagai satu FK (atribut berprefiks "primary") bila ada yang ditetapkan utama.

**Course registration periodic snapshot** (opsional) — bila pengguna ingin status pada tanggal kunci (preregistration, awal term, drop/add deadline, akhir term): grain = satu baris per course terdaftar per student per term **per tanggal snapshot**.

---

## 5. "Apa yang Tidak Terjadi"

**Explicit rows untuk non-event** — untuk memantau mahasiswa terdaftar yang **tidak hadir**: tambahkan baris eksplisit dengan `Attendance Count = 1 atau 0` (tabel tak lagi factless). Layak di sini karena non-event punya **dimensionalitas sama** dan no-show diasumsikan **porsi kecil**. ⚠️ Tapi menambah baris untuk event yang tak terjadi **konyol** di banyak konteks lain (mis. baris untuk produk promosi yang **tak** dibeli tiap transaksi customer).

**OLAP untuk non-event** — database multidimensional (OLAP) unggul memahami "apa yang tidak terjadi": menangani **sparsity** tanpa menyimpan nol eksplisit → data event & non-event tersedia sekaligus, mengurangi kerumitan star schema relasional (untuk cube yang tak terlalu sparse).

---

## 6. Peluang Analitik Pendidikan Lain

Banyak proses dari bab sebelumnya berlaku di universitas: **procurement** & **HR** (mengelola biaya). **Research grant** = variasi analisis keuangan (Bab 7) di level lebih detail (seperti subledger) dengan dimensi tambahan (funding source, research topic, grant duration, investigator) — mengelola budget vs actual agar tak surplus/defisit. **Alumni** = seperti memahami basis customer (Bab 8): karakteristik geografis/demografis/employment/interest/behavioral + data semasa mahasiswa → menargetkan pesan, alokasi sumber daya, recruiting, job placement, riset. Butuh **CRM operasional** yang melacak semua touch point alumni.

---

## Ringkasan Bab

Bab 13 (universitas) menyoroti dua pola khas. **Accumulating snapshot** untuk **pipeline** (applicant admissions, research grant proposal): grain per entitas yang di-update lewat milestone, banyak role-playing date, analisis pace & bottleneck — dengan keterbatasan (baris di-update tak melestarikan status historis → lengkapi dengan snapshot cut-off atau transaction fact table). **Factless fact table** dua tipe: **event** (admissions attendance, course registration — grain per registrasi per term, term conform ke date, sering pakai fakta buatan `count=1`) dan **coverage** (facility utilization — satu baris per slot terlepas dipakai/tidak, untuk occupancy rate). Ditambah **multiple instructor** (bridge/konkatenasi/primary), **course registration periodic snapshot**, penanganan **"apa yang tidak terjadi"** (explicit rows vs OLAP sparsity), dan peluang analitik lain (research grant, alumni). Bab 14 (Healthcare) akan memperdalam bridge table & pola data kesehatan yang kompleks.
