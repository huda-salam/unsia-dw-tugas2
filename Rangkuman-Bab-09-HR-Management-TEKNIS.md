# Bab 9 — Human Resources Management (Versi Mendalam & Teknis)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · **kedalaman teknis** — struktur skema, SQL, implikasi ETL, analisis trade-off

Versi ini membedah Bab 9 ke tingkat implementasi. Fokus teknis: **anti-pola factless profil vs Type 2 dimension**, **jebakan ripple manager key Type 2**, **management hierarchy bridge**, dan **dilema AND/OR skill bridge** dengan solusi SQL (UNION/INTERSECTION vs text-string LIKE).

---

## 1. Anti-Pola: Factless Profil vs Type 2 Dimension

**Draf naif (Figure 9-1):**
```
Employee Transaction Fact (factless)      grain = 1 baris / transaksi profil
  Transaction Date Key (FK)
  Transaction Date/Time
  Employee Key (FK)               ──► Employee Dimension (Type 2, sangat lebar)
  Employee Transaction Type Key (FK) ──► reason code (promosi, ganti alamat, ...)
```
**Mengapa buruk:** setiap transaksi profil → satu baris Type 2 baru di employee dimension. Akibatnya fact table & dimension **punya jumlah baris identik** dan **hampir selalu di-join**. Ini melanggar prinsip: fact table tak boleh sebanyak dimensinya.

**Solusi (Figure 9-2):** buang fact table; **perkaya employee dimension**:
```
Employee Dimension
  Employee Key (PK surrogate)
  Employee ID (NK durable)
  Employee Name/Address, Job Grade, Salary, Education,
  Original Hire Date (FK), Last Review Date (FK), Appraisal Rating,
  Health Insurance Plan, Vacation Plan,
  Change Reason Code, Change Reason Description,
  Row Effective Date/Time, Row Expiration Date/Time, Current Row Indicator
```
Reason code menjadi **atribut change reason**. Atribut numerik yang **diringkas** (bukan sekadar difilter) → milik fact table.

---

## 2. Timespan Presisi Type 2 — mekanika ETL

- **Date vs date/time:** saat latency harian → cukup date. Saat load lebih sering → **date/time stamp** (profil bisa beda 9 pagi vs 9 malam) agar event operasional dikaitkan ke baris profil yang tepat.
- **Expire:** baris terkini expiration = tanggal fiktif masa depan; saat di-expire → set "just before" effective baris baru (prior day/minute/second).
- **Revert:** bila profil kembali ke karakteristik lama → **insert baris baru** (jangan sekadar revisi expiration baris lama → dua baris efektif bersamaan).
- **Query point-in-time:**
```sql
SELECT COUNT(*) FROM employee_dim
WHERE effective_datetime <= @t AND @t < expiration_datetime;  -- profil pada instant @t
```

### 2.1 Change reason (metadata dalam data)

Multivalued (beberapa atribut berubah sekaligus) → text string `|Last Name|ZIP|` atau bridge.
```sql
-- "Berapa orang ganti ZIP tahun lalu?"
... WHERE ChangeReason LIKE '%ZIP%' AND <rentang tahun>;
```

### 2.2 Super transaction

Sumber mencatat perubahan sebagai banyak *micro-transaction* per atribut. **Bungkus** jadi satu super transaction (mis. "Promotion") → satu baris Type 2 mencerminkan semua atribut berubah. Idealnya aplikasi HR mencatat aksi level tinggi; jika tidak, deteksi super transaction di ETL bisa rumit.

### 2.3 Type 2 attribute vs fact event — batas

Jangan pakai employee dimension untuk melacak **setiap** event (review, benefit, prof-dev). Banyak melibatkan dimensi lain (reviewer, approver, exit interviewer, separation reason) → **fact table process-centric terpisah** (sering factless, tapi memungkinkan count/trend). Outcome (job grade hasil promosi) boleh jadi atribut; **jangan** overload dengan banyak FK outrigger.

---

## 3. Headcount Periodic Snapshot

```
Employee Headcount Snapshot Fact          grain = 1 baris / karyawan / bulan
  Month Key (FK), Organization Key (FK), Employee Key (FK)
  Employee Count, New Hire Count, Transfer Count, Promotion Count   (additive)
  Salary Paid, Overtime Paid, Retirement Fund Paid,
  Retirement Fund Employee Contribution                            (additive)
  Vacation Days Accrued, Vacation Days Taken                       (additive)
  Vacation Days Balance                                            (SEMI-additive)
```
- `Employee Key` = baris employee dimension yang berlaku di **akhir hari terakhir bulan** → menjamin laporan month-end akurat.
- Metrik bulanan sulit dihitung dari employee dimension saja. **Balance** semi-aditif: rata-rata lintas Month setelah `SUM` lintas dimensi lain.

**Bus matrix HR** (contoh proses & jenis fact table): requisition pipeline (accumulating), hiring (transaction), benefits eligibility/participation (periodic), headcount (periodic), compensation (transaction), review pipeline (accumulating), prof-dev (transaction), disciplinary pipeline (accumulating), separations (transaction).

---

## 4. Recursive Manager Hierarchy — dua desain & jebakan ripple

### 4.1 Dual role-playing (manager key di fact table)
```
Employee Separation Fact
  Employee Key (FK)  ──► Employee Dimension
  Manager Key (FK)   ──► Manager Dimension (role-play employee, label "Manager ...")
```
Akses simetris, performa setara. **Kelemahan:** dual FK di **setiap** fact table.

### 4.2 Outrigger (manager key di employee dimension)
```
Employee Dimension
  ... Manager Key (FK) ──► Manager Dimension (role-play, current-only ideal)
```

### 4.3 Jebakan ripple Type 2 (analisis kritis)

Abby manajer Hayden. Jika **manager key = Type 2** DAN atribut Abby (mis. alamat rumah) = Type 2:
1. Alamat Abby berubah → baris employee baru untuk Abby (employee key baru).
2. Karena manager key Hayden = Type 2 → employee key Abby baru memicu **baris baru untuk Hayden**.
3. Bila **Abby = CEO** → satu perubahan profil merambat **ke seluruh karyawan** (replikasi baris untuk semua).

**Mitigasi:**
- Manager key = **durable natural key** manajer → link ke role-play dimension **current-row-only** (perubahan profil manajer tak merambat).
- Atau manager key = **Type 1** → selalu manajer kini (histori hilang, sering cukup).

### 4.4 Management hierarchy bridge (variable depth)
```
Management Hierarchy Bridge
  Manager Key (FK)
  Employee Key (FK)     -- tiap bawahan langsung/tidak + manajer-ke-diri
  # Levels from Top
  Bottom Flag, Top Flag
```
- **OLAP** menangani parent/child variable-depth mulus; **relational** butuh `CONNECT BY`/recursive CTE (tak praktis untuk pengguna BI) → pakai bridge.
- **Kelemahan:** sulit dibangun, banyak baris (performa), UX ad hoc rumit; agregasi **ke atas** perlu membalik arah join.
- **Type 2 + bridge = pertumbuhan cepat** (perubahan profil manajer senior merambat). **Solusi:** bridge pakai **durable key** + `Begin/End Effective Date` → baris bertambah hanya saat **relasi pelaporan** berubah, bukan tiap atribut Type 2. Lebih mudah dikelola tapi lebih sulit dinavigasi → kubur di aplikasi BI siap-pakai. (Alternatif pathstring dari Bab 7 juga relevan; tak ada silver bullet.)

---

## 5. Multivalued Skill — bridge vs text string

### 5.1 Skill keyword bridge
```
Employee Dimension ──► Employee Skill Group Bridge ──► Skills Dimension
                       (Skill Group Key FK,             (Skill Key PK,
                        Skill Key FK)                     Skill Description,
                                                          Skill Category)
```
Oracle+Unix+SQL → satu skill group key, tiga baris bridge.

**Dilema AND/OR (inti teknis):** SQL buruk untuk constraint lintas baris.
```sql
-- Unix DAN Linux (INTERSECTION) / Unix ATAU Linux (UNION):
(SELECT employee_id, employee_name
 FROM Employee e JOIN SkillBridge b ON e.SkillGroupKey=b.SkillGroupKey
                 JOIN Skills s      ON b.SkillKey=s.SkillKey
 WHERE s.Skill='UNIX')
INTERSECT   -- atau UNION untuk OR
(SELECT employee_id, employee_name
 FROM Employee e JOIN SkillBridge b ON ... JOIN Skills s ON ...
 WHERE s.Skill='LINUX');
```
Sembunyikan kompleksitas di interface kustom.

### 5.2 Skill keyword text string (menghilangkan bridge)
```
Employee Dimension
  ... Employee Skill Group List   -- mis. "|UNIX|C++|LINUX|"
```
AND/OR dalam **satu** SELECT, delimiter `|` mencegah salah cocok, `UCase` mengatasi case:
```sql
-- OR:
WHERE UCase(skill_list) LIKE '%|UNIX|%' OR UCase(skill_list) LIKE '%|LINUX|%';
-- AND (struktur sama, operator beda):
WHERE UCase(skill_list) LIKE '%|UNIX|%' AND UCase(skill_list) LIKE '%|LINUX|%';
```
**Trade-off:** standar SQL (jalan di DB apa pun), AND/OR mudah — **tapi tidak mendukung** count per skill keyword (untuk itu butuh bridge).

---

## 6. Survey & Text Comment — struktur

### 6.1 Survey
```
Employee Evaluation Survey Fact          grain = 1 baris / pertanyaan / responden
  Survey Sent Date Key (FK), Survey Received Date Key (FK)  -- role-playing date
  Survey Key (FK)
  Responding Employee Key (FK), Reviewed Employee Key (FK)  -- role-playing employee
  Question Key (FK)             ──► Question Dim (label, category, objective)
  Response Category Key (FK)    ──► Response Category Dim (favorable/hostile)
  Survey Number (DD)
  Response
```
Question & Survey dimension memudahkan pencarian topik lintas kuesioner. Pola berlaku untuk customer satisfaction/product usage.

### 6.2 Text comment — keputusan penempatan

| Kardinalitas komentar | Penempatan |
|---|---|
| < jumlah transaksi (ada "No Comment" berulang) | **Comment dimension** terpisah (FK di fakta) |
| Unik per event | Atribut **transaction-grained dimension** |

- **JANGAN** di fact table — bukan degenerate dimension; freeform text menambah *bulk* yang ikut terseret di tiap operasi metrik fakta.
- Bila memungkinkan, parse ke atribut (compliment/complaint), tapi verbatim biasanya tetap perlu.
- Join dimensi komentar besar lambat, tapi pengguna sudah memfilter ketat saat membaca komentar; analisis metrik umum tak terbebani.

---

## Ringkasan Teknis

Employee dimension kaya menggantikan factless profil fact table (yang punya baris = dimensi). **Type 2** dengan **date/time effective/expiration** (query `effective ≤ @t < expiration`), **change reason** (`LIKE '%ZIP%'`), dan **super transaction** (bungkus micro-transaction). **Headcount snapshot** grain per karyawan/bulan (balance semi-aditif). **Manager hierarchy** via dual role-play atau outrigger; **jebakan ripple**: manager key Type 2 + profil manajer Type 2 → perubahan CEO merambat ke semua → pakai **durable key/Type 1**. **Variable-depth** via **bridge** (durable key + effective date agar tumbuh saat relasi berubah, bukan tiap atribut). **Skill multivalued**: bridge (dilema AND/OR → UNION/INTERSECT) vs **text string** (`LIKE` + `UCase`, tak bisa count per skill). **Survey** grain per pertanyaan/responden (dual role-play date & employee). **Text comment** di **comment/transaction dimension** (bukan fact table), penempatan bergantung kardinalitas.
