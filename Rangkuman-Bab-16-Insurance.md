# Bab 16 — Insurance (Asuransi)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab penutup studi kasus yang berfungsi sebagai **rekapitulasi** — menganyam hampir semua pola pemodelan dimensional dari seluruh buku ke dalam satu kasus asuransi properti & kecelakaan (auto, homeowner, personal property). Mencakup: **value chain & bus matrix**, tiga jenis fact table komplementer (**transaction, periodic snapshot, accumulating snapshot**), **role-playing**, **SCD Type 1-3**, **mini-dimension**, **multivalued (bridge + weighting)**, **numeric attribute sebagai fakta vs dimensi**, **degenerate & junk dimension**, **audit dimension**, **supertype/subtype**, **pay-in-advance (written vs earned)**, **consolidated fact table**, **factless fact table**, **timespan accumulating snapshot**, dan **kesalahan umum yang dihindari**.

---

## 1. Value Chain, Bus Matrix & Tiga Jenis Fact Table

Value chain asuransi: **rate → quote → policy (underwriting) → premium billing → claim**. Bus matrix awal memuat proses policy transaction, premium snapshot, dan claim (kelak diperluas). Pola desain fundamental: **satu proses bisa menghasilkan tiga fact table komplementer** — transaction (detail tiap kejadian), periodic snapshot (ringkasan berkala), accumulating snapshot (pipeline). Bab ini menampilkan ketiganya untuk policy & claim.

---

## 2. Policy Transaction Fact Table

**Grain = satu baris per transaksi polis** (create/alter/cancel dengan reason). Ciri **classic transaction fact table**: hampir seluruhnya berisi **key** (data atomik granular → dimensionalitas tumbuh alami), dengan **satu fakta numerik** (`Policy Transaction Dollar Amount`) yang **maknanya bergantung pada transaction type** (tak bisa diberi label spesifik karena bercampur jenis).

Teknik yang diterapkan:
- **Role-playing date** — policy transaction date (masuk sistem) vs policy effective date (berlaku hukum); dua FK bernama unik, satu tabel date fisik, dipapar via view.
- **Degenerate dimension** — policy number & policy transaction number (setelah semua atribut header di-triage ke dimensi lain). ⚠️ Bila underwriter menetapkan **risk grade** menyeluruh untuk polis → itu milik **policy dimension**, dan policy number **tak lagi** degenerate.
- **Low cardinality dimension** — transaction type dimension (<50 baris). Meski sempit & dangkal, atribut tekstual **tetap** di dimensi (untuk filter & label).
- **Audit dimension** — key ke baris audit (data lineage: waktu ekstrak, source table, versi software ETL).

---

## 3. SCD, Mini-Dimension, Multivalued, Numeric

**SCD Type 1-3 pada policyholder** — Type 1 (overwrite; mis. koreksi date of birth), Type 2 (baris baru untuk ZIP code yang penting bagi pricing/risk — jangan revisit fakta lama), Type 3 (kolom "Historical" untuk realignment segmen customer massal). Type 3 jadi terlalu kompleks bila perlu >1 versi map historis.

**Mini-dimension (Type 4)** — policyholder dimension besar (>1 juta baris); pecah atribut yang cepat berubah & sering di-browse ke mini-dimension. **Covered item** (rumah/mobil yang diasuransikan) biasanya lebih besar dari policyholder → kandidat mini-dimension juga. ⚠️ Deskripsi objek tertutup bersifat **tekstual** → taruh di **dimensi** (target constraint & label), **bukan** fakta.

**Multivalued (bridge + weighting)** — commercial customer punya banyak kode **SIC/NAICS**. Seperti diagnosis group (Bab 14): **industry classification bridge** (join ke fakta atau outrigger customer dimension). Bila breakdown proporsional (50% agri, 30% dairy, 20% oil) → **weighting factor** di tiap baris bridge. Customer tanpa kode valid → baris bridge khusus "Unknown".

**Numeric attribute: fakta vs dimensi** — appraised value (kontinu, berubah seiring waktu) → **fakta**; boleh juga simpan **value range** ("$250.000–299.999") di dimensi untuk grouping. Coverage limit (terstandar, tak kontinu; "Replacement Value"/"Up to $250.000") → **atribut dimensi**.

---

## 4. Supertype/Subtype & Accumulating Snapshot Policy

**Heterogeneous supertype/subtype** (Bab 10) — lini bisnis sangat berbeda (homeowner vs auto vs personal property). Buat **subtype dimension** covered item & coverage per lini untuk atribut khas. **Tak perlu** fact table terpisah per lini (metrik tak bervariasi) — cukup **view** pada supertype fact table per subtype. Tak ada key baru; hanya **memperluas** baris dimensi.

**Complementary policy accumulating snapshot** — untuk efek kumulatif transaksi. Grain = satu baris per coverage & covered item per polis; date policy-centric (quoted, rated, underwritten, effective, renewed, expired), beberapa employee role (agent, underwriter). Tanpa transaction type dimension; fact set diperluas.

---

## 5. Premium Periodic Snapshot & Pay-in-Advance

**Grain = satu baris per coverage & covered item pada polis per bulan.** Menggunakan **conformed dimension** & **conformed facts** (subset dari transaction schema).

**Pay-in-advance facts (written vs earned)** — manajemen ingin tahu premium yang **written** (terjual) vs **earned** (terealisasi saat layanan diberikan). Revenue polis di-*earn* bulan demi bulan. Menghitung earned premium berarti mereplikasi seluruh aturan revenue recognition (kompleks, apalagi ada upgrade/downgrade tengah bulan) → lebih baik **di-source dari sistem operasional terpisah**. Contoh: polis tahunan $600 written 1 Jan → written Jan = $600, earned Jan = $50 ($600/12); Feb written = 0, earned = $50; cancel 31 Mar → earned Mar = $50, **written Mar = −$450**.

**Supertype/subtype revisited** — snapshot facts bervariasi per lini (fakta custom inkompatibel → banyak null). Solusi: **pisahkan snapshot fact table secara fisik per lini** (satu supertype + serangkaian subtype; subtype = salinan segmen supertype untuk coverage/covered item lini tertentu, plus fakta supertype sebagai kemudahan).

**Multivalued revisited** — auto insurance: banyak **insured driver** per polis → **policy-insured driver bridge** dengan **weighting factor** (berdasar share premium tiap driver). Karena relasi berubah → tambah **effective/expiration date** di bridge → menjadi **factless fact table** yang menangkap relasi polis–policyholder–covered item–driver yang berevolusi.

---

## 6. Claim: Transaction, Junk, Accumulating Snapshot

**Claim transaction fact table** — **grain = satu baris per transaksi tugas klaim** (open/close/reopen claim, set/reset reserve, salvage, adjuster, lawsuit, payment, subrogate). Dimensi: policyholder, coverage, covered item, claimant, 3rd party payee, employee (2 role), claim, claim profile, dst.

**Transaction vs profile junk dimension** — banyak indikator/deskripsi klaim. High-cardinality (alamat lokasi kerugian, narasi) → **claim dimension**. Low-cardinality codified (metode lapor, indikator catastrophe) → **claim profile dimension** (junk, satu baris per kombinasi unik profil). ⚠️ **Hindari** dimensi dengan baris sebanyak fact table.

**Claim accumulating snapshot** — pertanyaan "claim-to-date" sulit dijawab dari transaksi mentah. **Grain = satu baris per klaim** (dibuat saat dibuka, di-update sepanjang hidup klaim hingga tutup). **Tujuh role-playing date** (open, loss, estimate, 1st payment, most recent payment, subrogation, close), **status dimension** (open/closed/reopened), fakta lag & to-date. Dimensi transaksi-spesifik (employee, payee, transaction type) **ditekan**.

**Accumulating snapshot untuk workflow kompleks** — bila milestone **banyak & tak stabil**: tetapkan hanya **date kunci** sebagai role-playing (start, end, milestone kritis). Untuk milestone berlebih, pengguna lebih peduli **lag** daripada tanggal. 20 milestone → 190 kemungkinan lag; simpan cukup **19 lag dari anchor (event A/begin date)**, sisanya dihitung (B-to-C = A-to-C − A-to-B; null bila salah satu event tak terjadi — ditangani anggun saat count/average).

**Timespan accumulating snapshot** — accumulating snapshot klasik menyajikan **status kini** tapi **menghapus status antara**. Untuk merekonstruksi workflow pada tanggal arbitrer masa lalu: tambah **snapshot start/end date + current flag** (seperti Type 2); alih-alih update destruktif, **insert baris baru** yang melestarikan status untuk rentang waktu. View berbasis current flag melayani mayoritas pengguna; minoritas memfilter start/end date.

**Periodic instead of accumulating** — untuk klaim **berumur panjang** (disabilitas jangka panjang/cedera badan bertahun): pakai **periodic snapshot** (grain = satu baris per klaim aktif per interval, mis. bulanan; fakta aditif periode: amount claimed/paid, change in reserve).

---

## 7. Consolidated Fact Table & Factless Accident Events

**Policy/claim consolidated periodic snapshot** — untuk metrik profit (premium revenue vs claim loss). Alih-alih drill-across dua fact table berulang, buat **consolidated fact table** (Bab 7) dengan **dimensi tereduksi** ke granularitas terendah yang **sama** di kedua proses. ⚠️ Kembangkan consolidated **setelah** base metric tersedia di model atomik terpisah.

**Factless accident events** — merekam korelasi many-to-many pihak & objek dalam kecelakaan ("tabrakan literal"). Dimensi baru: **loss party** (individu terlibat) & **loss party role**. Fact table mendaftar korelasi orang & kendaraan dalam satu kecelakaan (fakta = count).

---

## Ringkasan Bab

Bab 16 (asuransi P&C) adalah **rekapitulasi** hampir seluruh teknik buku. Tiga fact table komplementer (transaction/periodic snapshot/accumulating snapshot) untuk **policy** & **claim**; **role-playing date**, **SCD Type 1-3**, **mini-dimension** (policyholder & covered item besar), **multivalued** via **bridge + weighting** (SIC/NAICS, insured driver), **numeric sebagai fakta vs dimensi** (appraised value vs coverage limit), **degenerate** (policy number) & **junk/profile dimension** (claim profile), **audit dimension**, **supertype/subtype** (lini bisnis, dipisah fisik saat fakta bervariasi), **pay-in-advance** (written vs earned premium, di-source operasional), **consolidated fact table** (policy/claim profit), **factless** (accident events, evolving relationship), **timespan accumulating snapshot** (rekonstruksi status historis via start/end + current flag), dan pilihan **periodic vs accumulating** untuk klaim berumur panjang. Ditekankan pula **kesalahan umum yang dihindari**: dimensi header berukuran fact table, teks kriptik di fakta, dan sejenisnya. Ini menutup bagian studi kasus; Bab 17 beralih ke **Kimball DW/BI Lifecycle**.
