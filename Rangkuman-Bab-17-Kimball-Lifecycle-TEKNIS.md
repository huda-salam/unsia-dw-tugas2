# Bab 17 — Kimball DW/BI Lifecycle (Versi Mendalam & Teknis)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · **kedalaman teknis** — detail proses, artefak, keputusan fisik, praktik implementasi

Bab 17 adalah metodologi (bukan skema), jadi "versi teknis" ini menekankan **detail implementasi yang dapat ditindaklanjuti**: proses langkah-demi-langkah, artefak/deliverable konkret, keputusan physical design (index, agregasi, partisi), dan checklist praktik per tahap.

---

## 1. Struktur Lifecycle & Track Paralel

```
                    ┌─ Technical Architecture Design → Product Selection & Installation ─┐
Program/Project     │                                                                    │
Planning ──┬─ Business ─┼─ Dimensional Modeling → Physical Design → ETL Design & Dev ────┼─ Deployment ─ Maintenance
           │  Req.      │                                                                    │              └─ Growth ─┐
           └────────────┼─ BI Application Design → BI Application Development ──────────────┘                          │
             (2-arah)   └────────────────────── Program/Project Management ──────────────────────────────────────────┘
                                                                                                         (loop kembali) ┘
```
- Kotak sama lebar ≠ effort sama (ETL ≫ physical design).
- Panah 2-arah planning ↔ requirements.
- **Product selection bukan langkah pertama** (anti-pola pemula).

---

## 2. Program/Project Planning — artefak & keputusan

**Readiness assessment (leading indicators, rank order):**
| # | Faktor | Sinyal bahaya |
|---|---|---|
| 1 | Executive business sponsor kuat | CIO sebagai sponsor (bukan mitra bisnis) |
| 2 | Motivasi bisnis mendesak | Tak ada sense of urgency |
| 3 | **Data feasibility** | Belum capture data bersih pada granularitas tepat (no short-term fix) |

**Scoping — "Law of Too":** hindari timeline terlalu singkat × terlalu banyak source × user × lokasi × kebutuhan analitik beragam. Mulai **single business process**.

**Justifikasi:** benefit ≫ cost; IT hitung expense (sisakan ruang growth — support DW/BI **tak** menurun seperti operasional); bisnis hitung benefit (revenue/profit, bukan "single version of truth").

**Staffing (peran, bukan orang):**
- Bisnis: sponsor, driver, lead, users.
- Straddler: business analyst, **data steward** (SME, politically challenging), BI application designer/developer.
- IT: project manager (komunikasi stellar), technical architect, data architect/modeler (**harus** rangkul dimensional, bukan sekadar hemat ruang), DBA (**lepaskan** dogma "satu index per tabel"), metadata coordinator, ETL architect/designer, ETL developer.

**Scope creep — 3 opsi:** (1) tambah scope (time/resource/budget), (2) zero-sum (tukar), (3) tolak-sebagai-enhancement. Jangan di "vakum IT".

---

## 3. Business Requirements — proses & artefak

**Forum:** interview vs facilitated; **survei ditolak** (datar, self-selected, tak menggali, tak membangun bond). Kimball **hybrid** (interview→detail, facilitation→konsensus).

**Interview flow (pemetaan ke model dimensional):**
```
Job responsibility (lob-ball)
  → Key performance metrics    ⟹ langsung = proses bisnis + fakta (tanpa tanya langsung)
  → "Bagaimana membedakan/mengategorikan produk?"  ⟹ atribut dimensi + hierarki
  → Jenis analisis (ad hoc vs standar)  ⟹ requirement BI tool
  → (eksekutif) visi leverage informasi  ⟹ selaraskan deliverable vs ekspektasi
Wrap-up: kriteria sukses TERUKUR + disclaimer fase 1
```
⚠️ Jangan desain berbasis "top 5 report" (pertanyaan user pasti berubah).

**Data-centric interview:** cek feasibility (minta initial **data profiling** — domain values, count field kritis) sebelum momentum requirements.

**Deliverable dokumentasi:**
- (opsional) write-up per interview (harus dipahami non-peserta).
- **Consolidated findings** (kritis) — per **proses bisnis**: exec summary → overview → per proses (why analyze, kapabilitas, keterbatasan, benefit/impact, feasibility).
- **Opportunity/stakeholder matrix** (baris = proses seperti bus matrix; kolom = grup organisasi; biasanya padat).

**Prioritization grid:**
```
Impact ▲  [kiri-atas: peluang besar,   ] [kanan-atas: KERJAKAN DULU  ]
tinggi    [ belum feasible → IT lain   ] [ high impact + feasible    ]
          [kiri-bawah: HINDARI         ] [kanan-bawah: doable tapi   ]
rendah    [ mission impossible         ] [ tak krusial → tunda       ]
          └──────────────── Feasibility ──────────────►
```

---

## 4. Technology Track — proses detail

**Technical architecture design (8 langkah):**
1. Bentuk architecture task force (2-3 org: technical architect + ETL + BI architect → back room & front room).
2. Kumpulkan requirement arsitektur (filter requirement bisnis untuk **timing/availability/performance**; + interview IT: standar, arah, boundary; lesson learned; kesediaan ubah operasional mis. identifikasi transaksi ter-update).
3. Dokumentasi (tabular: requirement bisnis → implikasi arsitektur; mis. "sales global tiap malam" → 24/7 availability, data mirroring, robust metadata, bandwidth, ETL horsepower).
4. Buat architecture model (grup: ETL/BI/metadata/infrastruktur; model high-level).
5. Tentukan fase implementasi (mandatory vs nice-to-have; minimal untuk proyek awal).
6. Desain & spesifikasi subsystem (yang tak ada di produk off-the-shelf — detail cukup untuk build/evaluasi).
7. Buat architecture plan (detail cukup untuk konstruksi; tak sebut produk spesifik kecuali in-house).
8. Review & finalisasi (komunikasi berlapis; langsung dipakai di product selection).

**Product selection (6 langkah):** pahami corporate purchasing → **evaluation matrix** berbobot (kriteria spesifik; vague = semua vendor "bisa") → market research (**hindari RFP** bila mungkin — time-consuming, beauty contest) → shortlist (diskualifikasi dari matrix; libatkan bisnis untuk BI tool; **Anda** yang mengarahkan, bukan vendor; cek reference) → prototype bila perlu (**maks 2 produk**, business case realistis) → pilih + trial + negosiasi (**komitmen privat**, trial hanya dengan vendor yang benar-benar diniatkan).

---

## 5. Data Track — physical design detail

**Naming/DB standards** — nama tabel/kolom **bermakna bagi bisnis**; standar key declaration & null.

**Physical model** — bangun di dev server; + **staging tables** (ETL), **auditing tables** (ETL processing & data quality), **security structures**.

**Index plan (keputusan konkret):**
```
Dimension: unique index pada single-column PK (surrogate);
           bitmap index pada atribut yang sering FILTER/GROUP,
           terutama yang di-constrain bersama; jika tak ada bitmap → evaluasi B-tree.
Fact:      B-tree/clustered index pada PK;
           date FK di POSISI TERDEPAN index (percepat load & query, date sering di-constrain);
           high-cardinality bitmap pada FK individual (lebih "agnostik" dari clustered
           saat user constrain dimensi tak terduga).
OLAP:      punya index & optimizer sendiri, tapi DBA sedikit kontrol.
```

**Agregasi** — lebih cost-effective dari menambah hardware. Saat agregasi: **eliminasi dimensi** atau kaitkan ke **shrunken rollup dimension** yang conform ke atomic base. Dua faktor evaluasi: (1) pola akses user (dari requirement + monitoring aktual), (2) distribusi statistik data (titik agregasi yang "bang for the buck"). Tak mungkin bangun semua agregasi teoretis. (Detail Bab 19-20.)

**Partisi** — fact besar per **activity date** (segmen per bulan, tampak satu tabel) → keuntungan load, maintenance, query. Strategi index/agregasi/tuning **berevolusi** seiring pola pemakaian; tetap kirim data ter-index & teragregasi memadai sejak rollout awal.

**ETL** — Bab 19 (34 subsystem) & Bab 20 (proses).

---

## 6. BI Applications Track — detail

**Specification:** 10-15 starter report/analytic app (narrow focus, kelola ekspektasi); standar (pull-down menu, look-and-feel); spesifikasi template (layout, input variable, kalkulasi, break); navigasi terstruktur (portal/dashboard).

**Development:** standar (naming, kalkulasi, library, coding) untuk minimalkan rework; mulai saat **desain DB selesai + BI tool/metadata terpasang + subset historis termuat**; revisit spec (model pasti berubah); investasi edukasi tool-specific.
- **Mulai sebelum ETL "selesai"** → developer temukan masalah data (jarum di tumpukan jerami) meski ETL sudah QA, + uji response time nyata lebih awal.
- QA BI **tak bisa** selesai sebelum data stabil → sisakan waktu setelah ETL cutoff final.

---

## 7. Deployment, Maintenance & Growth — checklist

**Deployment:** butuh preplanning + keberanian menilai kesiapan. **Data = entrée** (ETL = tugas paling tak terprediksi). End-to-end testing: data QA, operations processing, performance, usability. Kemas dengan edukasi + **support berlapis**: (1) website/self-service, (2) power user area bisnis, (3) tim DW/BI terpusat.

**Maintenance & Growth (4 area investasi):**
- **Support** — krusial segera (relokasi ke komunitas bisnis; jujur saat ada masalah; deliverable buruk → beban rekonsiliasi/rework membludak).
- **Education** — refresher/advanced/introductory berulang; informal untuk developer/power user.
- **Technical support** — perlakukan sebagai **produksi + SLA**; monitor performa & kapasitas **proaktif**.
- **Program support** — pasarkan sukses; komunikasi berkelanjutan; **checkpoint review** berkala.

**Growth = sukses, bukan gagal** → prioritization grid + komite sponsorship eksekutif → loop ke awal lifecycle (bangun di atas fondasi teknis/data/BI).

---

## 8. 10 Pitfall (referensi cepat)

| # | Pitfall |
|---|---|
| 10 | Terpesona teknologi/data, bukan kebutuhan bisnis |
| 9 | Gagal rekrut sponsor bisnis visioner |
| 8 | Proyek galaktik bertahun, bukan iteratif |
| 7 | Habis budget di struktur ternormalisasi sebelum area presentasi dimensional |
| 6 | Utamakan back room > front room query performance & ease of use |
| 5 | Area presentasi terlalu kompleks |
| 4 | Model dimensional standalone tanpa conformed dimension |
| 3 | Muat hanya data teragregasi (bukan atomik) |
| 2 | Anggap bisnis/data/teknologi statis |
| 1 | Abaikan bahwa sukses = **penerimaan bisnis** |

---

## Ringkasan Teknis

Lifecycle: **planning** (readiness: sponsor > motivasi > **data feasibility**; scoping single-process; "Law of Too"; staffing peran; scope creep 3 opsi), **requirements** (hybrid interview; flow metrik→proses/fakta; **consolidated findings** per proses; **prioritization grid** impact×feasibility), **3 track**: teknologi (**arsitektur 8 langkah** + **product selection 6 langkah**, hindari RFP, komitmen privat), data (physical design: **index plan** dimension-bitmap/fact-B-tree-date-leading, **agregasi** shrunken conform, **partisi** activity date), BI (10-15 template, mulai **sebelum ETL selesai**, QA butuh data stabil). **Deployment** (data=entrée, testing end-to-end, support 3 tier), **maintenance & growth** (4 area investasi, growth=sukses→loop). **10 pitfall**, puncaknya penerimaan bisnis.
