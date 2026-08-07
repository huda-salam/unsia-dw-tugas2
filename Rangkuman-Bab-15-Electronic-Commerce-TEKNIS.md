# Bab 15 — Electronic Commerce (Versi Mendalam & Teknis)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · **kedalaman teknis** — struktur skema, SQL, implikasi ETL, analisis trade-off

Fokus teknis: metode **session tracking** (5 opsi + trade-off), **dual role date/time** (universal vs lokal), struktur **dua fact table clickstream**, **step dimension 3 role**, **aggregate + shrunken conformed dimension**, dan **P&L profitability** (kolom terhitung, alokasi biaya).

---

## 1. Session Tracking — 5 metode & trade-off

HTTP stateless → session ID harus dibangun:

| # | Metode | Kekuatan | Kelemahan |
|---|---|---|---|
| 1 | Kolasi entri log kontigu per host/IP | Tanpa cookie/fitur khusus | IP dinamis dipakai ulang; IP beda dalam 1 sesi; firewall |
| 2 | Session-level cookie | Temporary session ID untuk browser & app | Tak tahu kunjungan berikutnya |
| 3 | SSL (login + key exchange) | Terikat login | Overhead SSL tinggi; security advisory; tiap host butuh cert |
| 4 | Session ID di hidden field (halaman dinamis) | — | Butuh kontrol ketat; rapuh bila link tak dukung; multi-vendor pecah |
| 5 | **Persistent cookie** | **Paling andal**; grup situs bisa "super session" via ID bersama | Bisa ditolak/dihapus manual |

**Rekomendasi:** persistent cookie (terbaik); atau session cookie + kolasi host kontigu (butuh algoritma post-processor robust).

**Session ID** di-generate oleh **aplikasi operasional** (mis. order entry), **bukan** web server.

---

## 2. Session-Grained Fact Table

```
Clickstream Session Fact            grain = 1 baris / sesi customer selesai
  Universal Date Key (FK)           -- role 1: GMT/UTC (sinkronisasi lintas server)
  Universal Date/Time
  Local Date Key (FK)               -- role 2: jam dinding customer
  Local Date/Time
  Customer Key (FK)
  Entry Page Key (FK)               -- halaman AWAL sesi
  Session Key (FK), Referrer Key (FK)
  Session ID (DD)
  Session Seconds, Pages Visited,
  Orders Placed, Order Quantity, Order Dollar Amount
```

**Keputusan teknis:**
- **Grain sengaja tinggi** — menyimpang dari "grain terendah" agar terkelola (100 juta page fetch → 20 juta sesi @ 5 halaman). Session fact = agregasi page event (tak bisa jawab perilaku per halaman).
- **Dual role date/time via view:**
```sql
CREATE VIEW universal_date AS SELECT date_key, date AS universal_date, ... FROM date_dim;
CREATE VIEW local_date     AS SELECT date_key, date AS local_date, ...     FROM date_dim;
```
Zona waktu **disimpan di DB** (selisih kota bisa berubah 2 jam karena DST → jangan bebankan ke BI).
- **Date/time stamp penuh**, bukan time-of-day dimension (minim atribut; raksasa bila grain detik; stamp memudahkan aritmetika selisih lintas hari).
- **Causal dimension produk TIDAK cocok** (multivalued bila sesi banyak produk) → taruh di page-event fact. Market conditions (semua produk) cocok.
- **ETL session seconds:** bila akhir tak terdeteksi → assign nilai kecil nominal (hindari distorsi).

---

## 3. Page-Event-Grained Fact Table

```
Clickstream Page Event Fact         grain = 1 baris / page event / sesi
  Universal/Local Date Key + Date/Time (2 role)
  Customer Key, Page Key, Event Key, Session Key
  Session ID (DD)
  Session Step Key (FK)             ┐
  Purchase Step Key (FK)            ├─ Step Dimension, 3 role (view)
  Abandonment Step Key (FK)         ┘
  Product Key (FK), Referrer Key (FK), Promotion Key (FK)
  Page Seconds                      -- conformed fact (≠ Session Seconds!)
  Order Quantity, Order Dollar Amount   -- 0/null bila bukan event order
```

- **Grain terendah praktis** (micro-event grafis dibuang). ⚠️ Jangan agregasi lebih kasar (buang dimensi).
- **Conformed facts:** `Page Seconds` ≠ `Session Seconds` — **beri nama beda** agar tak salah dijumlah; `Σ page_seconds` per sesi = `session_seconds`.
- **Null facts** (order pada baris non-order) efisien (≈0 ruang di banyak DBMS), mengikat revenue langsung ke perilaku.

**Step dimension 3 role:**
```
Step Dimension (Step Number, Steps Until End)
```
```sql
-- Halaman AWAL pembelian sukses:
WHERE purchase_step.step_number = 1
-- Halaman TERAKHIR sesi gagal (abandonment):
WHERE abandonment_step.steps_until_end = 0
```

---

## 4. Aggregate + Shrunken Conformed Dimension

```
Session Aggregate Fact              -- <1% ukuran asli → ≥100× lebih cepat
  Universal Month Key (FK)          ──► Month Dim (shrunken conform dari calendar day)
  Demographic Key (FK)              ──► Demographic Dim (shrunken conform dari customer)
  Entry Page Key (FK), Session Outcome Key (FK)
  Number of Sessions, Session Seconds, Pages Visited,
  Orders Placed, Order Quantity, Order Dollar Amount
```
Dibangun dari session fact: `GROUP BY month, demographic_type, entry_page, session_outcome`; `COUNT(sessions)`, `SUM(fakta aditif)`. **Disiplin:** dimensi rollup harus **conformed subset** (bukan tabel sembarangan).

---

## 5. Profitability Lintas Channel (P&L)

**Keputusan arsitektur:** bangun di atas **sales transaction** (alokasi biaya ke transaksi), **bukan** clickstream (alokasi ke sesi tanpa produk/penjualan = terlalu kontroversial). Manfaat: profitabilitas **semua channel** (store/telesales/web).

```
Profitability Fact                  grain = tiap line item terjual (= sales transaction)
  Universal/Local Date Key + Time of Day Key (2 role each)
  Customer Key, Channel Key, Product Key, Promotion Key
  Ticket Number (DD)
  Units Sold
  Gross Revenue
  (−) Manufacturing Allowance, Marketing Promotion, Sales Markdown
  = Net Revenue                     ← kolom terhitung
  (−) Manufacturing Cost, Storage Cost
  = Gross Profit                    ← kolom terhitung
  (−) Freight Cost, Special Deal Cost, Other Overhead Cost
  = Net Profit                      ← kolom terhitung
```

**Kolom terhitung (net revenue/gross profit/net profit):**
- Akses via **view** → cukup dihitung (tak perlu simpan fisik).
- Akses tabel fisik langsung → **simpan fisik** (hindari salah hitung P&L kompleks).

**Alokasi biaya (isu ETL/bisnis):**
- Tiap biaya di-source/estimasi terpisah; entri baris = fraksi biaya total ke grain.
- Kualitas bervariasi: rata-rata nasional tahunan (pro forma) → kuartal/region → activity-based dinamis.
- **Biaya website** (infrastruktur, sulit dialokasi langsung): per jumlah halaman produk / per pages visited / per pembelian web aktual.
- Tim DW/BI **bukan** pelaksana ABC organisasi — ambil data terbaik, publikasikan, perbaiki seiring waktu (notifikasi saat business rule membaik).

**Kekuatan:** P&L dalam kerangka dimensional simetris → jawab **apa** & **mengapa** (slice per channel/segment/product line/promosi/waktu, termasuk gabungan constraint).

---

## Ringkasan Teknis

**Session tracking:** 5 metode (persistent cookie terbaik); session ID dari app operasional bukan web server. **Session-grained fact** (grain sengaja tinggi; dual role date universal/lokal via view, zona waktu di DB; date/time stamp penuh; causal produk ke page-event fact). **Page-event-grained fact** (grain terendah; step dimension 3 role → query halaman awal-sukses/akhir-gagal; conformed facts page vs session seconds; null order efisien). **Aggregate** dengan shrunken conformed dimension (<1% ukuran, ≥100×). **Profitability** = ekstensi sales transaction (bukan clickstream), grain line item, **P&L** (net revenue/gross profit/net profit — view vs simpan fisik), alokasi biaya (website: per halaman/visit/purchase; DW/BI bukan pelaksana ABC).
