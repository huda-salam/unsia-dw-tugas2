# Bab 9 — Human Resources Management (Manajemen SDM)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab ini menunjukkan bahwa kadang **dimensi karyawan yang kaya sudah mendukung banyak analisis pada dirinya sendiri** — sebuah pergeseran dari asumsi "semua metrik ada di fact table". Fokus: pelacakan **perubahan profil karyawan** (SCD Type 2 intensif dengan timespan presисi & change reason), **headcount periodic snapshot**, **recursive employee/manager hierarchy**, **bridge table untuk skill multivalued**, serta penanganan **survey data** dan **text comment**.

---

## 1. Employee Profile Tracking

Karyawan punya profil HR dengan ≥100 atribut (hire date, job grade, salary, review, cuti, organisasi, pendidikan, alamat, asuransi, dll.) yang **terus berubah** (hire, transfer, promosi, penyesuaian). Kebutuhan utama: **melacak & menganalisis perubahan profil secara akurat.**

**Draf awal (yang dihindari):** factless fact table dengan grain "satu baris per transaksi profil karyawan" + Type 2 di employee dimension. Masalahnya: fact table & dimension punya **jumlah baris sama** dan hampir selalu di-join — alarm pemodelan berbunyi (jangan sampai fact table sebanyak dimensinya).

**Desain yang benar:** cukup **perkaya employee dimension** dan buang fact table transaksi profil. Employee dimension memuat **snapshot profil setelah tiap perubahan**; deskripsi transaction type menjadi atribut **change reason**. Surrogate key = PK; natural employee ID = atribut durable.

---

## 2. Timespan Presisi & Change Reason

- **Effective & expiration date/time** — dua kolom yang mendefinisikan rentang saat profil akurat (seperti SCD Type 2 di Bab 5). Gunakan **date/time stamp** (bukan sekadar date) bila data dimuat lebih sering dari harian (profil bisa beda antara jam 9 pagi & 9 malam). Baris terkini di-set expiration ke tanggal jauh di masa depan; saat di-expire, di-set "tepat sebelum" efektif baris baru. **Current row indicator** untuk mengambil status terkini cepat.
- Type 2 employee dimension **sendirian** sudah bisa menjawab "berapa karyawan & profilnya pada titik waktu X" (constrain `effective ≤ waktu < expiration`).
- **Change reason tracking** — menempelkan metadata ETL ke data: atribut change reason (mis. `Last Name`, `ZIP`). Karena beberapa atribut bisa berubah bersamaan, change reason **multivalued** → tangani sebagai text string (`|Last Name|ZIP|`) atau bridge table. Query "berapa orang ganti ZIP tahun lalu?" pakai `LIKE '%ZIP%'`.
- **Super transaction** — sumber kadang mencatat perubahan sebagai banyak *micro-transaction* per atribut. Bungkus jadi satu "super transaction" (mis. promosi) → satu baris Type 2 baru yang mencerminkan semua perubahan sekaligus (idealnya aplikasi HR mencatat aksi level tinggi).

**Profile change: Type 2 attribute atau fact event?** Jangan berlebihan — dimensi bisa membengkak (ratusan atribut, jutaan baris untuk 100.000 karyawan). Banyak event HR (review, benefit, professional development) melibatkan dimensi lain (reviewer, approver, separation reason) → **modelkan sebagai fact table process-centric terpisah** (banyak factless). Outcome event (mis. job grade hasil promosi) boleh jadi atribut employee dimension, tapi **jangan** menjejalkan banyak FK outrigger (reviewer, benefit, dll.) ke employee dimension.

---

## 3. Headcount Periodic Snapshot

Untuk melaporkan status karyawan reguler (count, statistik, total). **Grain = satu baris per karyawan per bulan.** Dimensi: Month, Employee, Organization (employee key = baris yang berlaku di akhir bulan). Fakta: employee count, new hire/transfer/promotion count, salary/overtime/retirement paid, vacation accrued/taken/balance.

- Metrik bulanan ini **sulit dihitung dari employee dimension saja** → butuh snapshot.
- Sebagian besar **aditif**; kecuali yang berlabel **balance** (mis. vacation days balance) yang **semi-aditif** (harus dirata-rata lintas bulan setelah dijumlahkan lintas dimensi lain).

**Bus matrix HR** — banyak proses: requisition pipeline (accumulating), hiring (transaction), benefits eligibility/application/participation, headcount, compensation, review pipeline, professional development, disciplinary action, separations. Employee dimension + headcount snapshot hanya permukaan.

---

## 4. Recursive Employee/Manager Hierarchy

Atribut umum: nama manajer karyawan. Bila butuh lebih dari sekadar nama:

- **Manager key sebagai FK di fact table** → join ke **role-playing employee dimension** (semua atribut berlabel "manager"). Akses simetris employee & manager, performa sama. Kelemahan: dual FK harus ada di **setiap** fact table.
- **Manager key sebagai atribut employee dimension** → join ke **outrigger** role-play employee dimension.

**Change tracking pada embedded manager key (jebakan penting):** contoh Abby manajer Hayden. Jika manager key = Type 2 **dan** atribut Abby (mis. alamat) juga Type 2 → perubahan alamat Abby membuat baris baru Abby, yang lalu memicu baris baru Hayden. Bila **Abby CEO**, satu perubahan profilnya **merambat ke seluruh karyawan**. Solusi: pakai **durable natural key** manajer (link ke baris manajer terkini saja), atau perlakukan manager key sebagai **Type 1** (selalu manajer kini, histori hilang — kadang cukup).

**Drilling naik/turun hierarki manajemen** (mis. seluruh rantai komando): atribut/FK sederhana hanya cukup untuk relasi employee→manager berkedalaman tetap. Untuk hierarki rekursif berkedalaman variabel:
- **OLAP** menangani parent/child variable-depth dengan mulus (kekuatan OLAP).
- **Relational** butuh `CONNECT BY`/recursive CTE — praktis tak terpakai oleh pengguna BI.
- → Pakai **bridge table** (seperti Bab 7): satu baris per manajer × tiap karyawan (langsung/tidak) di rantainya + baris manajer-ke-dirinya, dengan `# Levels from Top`, `Bottom Flag`, `Top Flag`. Kelemahan: sulit dibangun, banyak baris (performa), UX ad hoc rumit, agregasi ke atas perlu membalik join path. Bila dikombinasi Type 2 → **tumbuh cepat**; pakai **durable key + effective/expiration date** agar baris hanya bertambah saat relasi pelaporan berubah (bukan tiap perubahan atribut). Sebaiknya dikubur di aplikasi BI siap-pakai.

---

## 5. Multivalued Skill Keyword (Bridge Table)

Departemen IT ingin menandai karyawan dengan skill teknis (bahasa program, OS, database) — jumlahnya **variabel & open-ended**. Bila skill berjumlah tetap → atribut posisional (mis. kolom "Linux Skills"/"No Linux Skills"), mudah & cepat, tapi runtuh saat skill bertambah banyak.

**Skill keyword bridge:** `Employee Skill Group Bridge` menghubungkan employee ke `Skills Dimension` via *skill group key*. Karyawan yang menguasai Oracle+Unix+SQL diberi skill group key sama → tiga baris di bridge.

**Dilema AND/OR:** query **OR** (Unix *atau* Linux) mudah (`OR` di skill description). Query **AND** (Unix *dan* Linux) sulit — constraint melintasi **dua baris** (SQL buruk untuk ini). Solusi: `UNION` (untuk OR) / `INTERSECTION` (untuk AND) — disembunyikan di interface kustom.

**Alternatif — skill keyword text string:** hilangkan bridge dengan menyimpan **satu string terdelimit** (`|Unix|C++|`) sebagai atribut/outrigger employee. AND/OR bisa dalam satu SELECT: `UCase(skill_list) LIKE '%|UNIX|%' AND/OR ... '%|LINUX|%'` (delimiter `|` mencegah salah cocok; `UCase` mengatasi ambiguitas huruf besar/kecil). Kelemahan: **tidak mendukung** query yang menghitung per skill keyword.

---

## 6. Survey Data & Text Comments

**Survey questionnaire.** **Grain = satu baris per pertanyaan per responden.** Dimensi: dua **role-playing employee** (responding & reviewed employee), Survey (deskriptor instrumen), Question (pertanyaan + kategori; pertanyaan sama muncul di banyak survey), Response Category (mis. favorable/hostile). Mendukung slicing survey yang kaya; polanya berlaku untuk customer satisfaction, product usage, dll.

**Text comments.** Komentar bebas (mis. remark reviewer) tak rapi masuk kategori fakta/dimensi. Bila bisnis menolak membuangnya:
- Cek apakah bisa di-*parse* jadi atribut rapi (mis. compliment vs complaint); tapi verbatim biasanya tetap diperlukan.
- **Jangan** simpan di fact table (bukan degenerate dimension; menambah *bulk* yang ikut terseret di tiap operasi fakta).
- **Simpan di luar fakta:** sebagai **comment dimension** terpisah (FK di fakta) bila kardinalitas komentar < jumlah transaksi (mis. banyak "No Comment"); atau sebagai atribut **transaction-grained dimension** bila tiap event unik. Join dimensi besar ini lambat, tapi saat pengguna membaca komentar mereka biasanya sudah memfilter ketat — dan analisis metrik umum tak terbebani bobot teks.

---

## Ringkasan Bab

Bab 9 menekankan **employee dimension yang kaya** sebagai pusat analisis HR: perubahan profil ditangani **SCD Type 2** dengan **timespan presisi** (date/time effective/expiration) + **change reason** (metadata ETL), bukan sebagai fact table transaksi terpisah. **Headcount periodic snapshot** (grain per karyawan per bulan) menyediakan count & metrik bulanan (balance = semi-aditif). **Recursive manager hierarchy** ditangani via role-playing dimension (fixed depth) atau **bridge table** (variable depth) — dengan peringatan *ripple* Type 2 dari perubahan profil manajer senior (pakai durable key). **Skill multivalued** memakai **bridge table** (dengan dilema AND/OR → UNION/INTERSECTION) atau **text string terdelimit**. Ditutup dengan skema **survey** (grain per pertanyaan per responden, dual role-playing employee) dan penanganan **text comment** (di luar fact table). Bab 10 (Financial Services) akan membahas heterogeneous product, mini-dimension, dan skema supertype/subtype.
