# Bab 15 — Electronic Commerce (Perdagangan Elektronik)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Studi kasus **web retailer** yang berfokus pada **clickstream** — data unik dengan tantangan identifikasi tersendiri. Bab ini membahas **sumber & tantangan clickstream** (mengidentifikasi asal pengunjung, sesi, identitas), **dimensi khas clickstream** (page, event, session, referral, step), **dua fact table** (grain session & grain page event), **aggregate table**, integrasi clickstream ke bisnis retail (conformed dimension), dan **profitability lintas channel** (P&L).

---

## 1. Sumber & Tantangan Clickstream

Clickstream **bukan** sekadar sumber data lain — ia kumpulan sumber yang berevolusi (aneka format server log). Tantangan struktural:
- **Terdistribusi** — data dikumpulkan banyak server fisik serentak; sinkronisasi log **sulit** (server sibuk = ratusan page event/detik, jam antar-server tak sinkron ke 1/100 detik).
- **Banyak pihak** — log sendiri, mitra referral, ISP, atau spesifikasi pencarian dari search engine.
- **Stateless** — HTTP tak punya konsep sesi; log menunjukkan event halaman terisolasi tanpa kaitan jelas.
- **Anonim** — sulit tahu siapa pengunjung, atau apakah pernah datang.

**Mengidentifikasi asal pengunjung** — bisa dari home page default, referral search engine (index berbayar/pencarian), bookmark, atau **clickthrough** (klik link dari situs lain — sering berbayar via banner). Situs perujuk biasanya teridentifikasi sebagai field di web event (penting untuk verifikasi efektivitas marketing & audit tagihan iklan).

**Mengidentifikasi sesi** — tiap kunjungan butuh **session ID** unik (seperti nomor struk). Karena HTTP stateless, sesi ditetapkan lewat: (1) kolasi entri log kontigu dari host/IP sama (rapuh: IP dinamis dipakai ulang, IP beda dalam satu sesi, firewall); (2) **session-level cookie** (bertahan selama browser buka — tak tahu kunjungan berikutnya); (3) SSL (overhead tinggi); (4) session ID di hidden field halaman dinamis (butuh kontrol ketat, rapuh bila link tak dukung); (5) **persistent cookie** (paling andal — tapi bisa ditolak/dihapus; grup situs bisa sepakati ID bersama untuk "super session"). **Paling andal: persistent cookie**; alternatif baik: session cookie + kolasi host kontigu (butuh algoritma post-processor kuat).

**Mengidentifikasi pengunjung** — paling sulit: pengunjung ingin **anonim** & mungkin memberi info tak akurat; cookie mengidentifikasi **komputer, bukan individu** (anggota keluarga berbeda; satu orang pakai banyak device).

---

## 2. Dimensi Khas Clickstream

Selain dimensi umum (date, customer, product, promotion, dll.), clickstream punya **empat dimensi khas**:
- **Page** — atribut halaman: page source, page function, page template, item/graphics/animation/sound type, file name.
- **Event** — jenis event dalam halaman: open page, refresh, click link (field XML-driven).
- **Session** — hasil/konteks sesi: session type, local & session context (page-derived vs trajectory-derived), action sequence, **success status** (misi tercapai?), customer status. Dimaksud mendeskripsikan **kelas/kategori** sesi (bukan tiap sesi individual).
- **Referral** — asal rujukan: referral type, referring URL/site/domain, search type, specification, target (di mana pencarian menemukan match).
- **Step** — posisi event dalam urutan sesi (dari Bab 8): step number, steps until end.

---

## 3. Dua Fact Table Clickstream

### 3.1 Session-grained (agar tabel tak meledak)

**Grain = satu baris per sesi customer yang selesai** — sengaja **menyimpang** dari prinsip "grain terendah" agar terkelola (mis. 100 juta page fetch/hari → 20 juta sesi bila rata-rata 5 halaman/sesi). Dimensi: calendar date, time of day, customer, **entry page** (halaman awal sesi — "bagaimana customer naik ke bus Anda?"), session, referrer. Fakta: session seconds, pages visited, orders placed, order quantity, order dollars.

- **Dua role date/time:** waktu **universal** (GMT/UTC — sinkronisasi lintas server/zona waktu) & waktu **lokal** (jam dinding customer). Zona waktu **disimpan di database** (bukan dihitung BI — selisih kota bisa berubah 2 jam karena DST). Keduanya = view dari satu date dimension.
- Modelkan instant dengan **date/time stamp penuh**, bukan time-of-day dimension (yang minim atribut bermakna & bisa raksasa; stamp memudahkan aritmetika selisih waktu lintas hari).
- **Causal dimension** berbasis produk **tidak cocok** di sini (multivalued bila sesi menyentuh banyak produk) — taruh di fact table grain page event. Market conditions dimension (semua produk) cocok di grain session.
- **Session seconds** — bila akhir sesi tak terdeteksi (customer ketik URL baru/tutup browser), ETL beri nilai kecil nominal agar analisis tak terdistorsi.

### 3.2 Page-event-grained (grain terendah praktis)

**Grain = satu baris per page event per sesi** (micro-event elemen grafis dibuang). ⚠️ **Jangan** agregasi ke grain lebih kasar (menghilangkan dimensi). Dimensi tambahan vs session: **page** (halaman individual), **event**, **product** (+**promotion** untuk interpretasi causal), **step (3 role)**. Session ID = degenerate dimension ("parent key" penghubung). Fakta: **page seconds** (bukan "session seconds" — beri nama beda demi **conformed facts**, agar tak salah dijumlah; page seconds seharusnya menjumlah ke session seconds), order quantity, order dollars (nol/null bila bukan event pemesanan — tapi tetap disertakan untuk mengikat revenue ke perilaku; null efisien di banyak DBMS).

**Step dimension (3 role)** — overall session, purchase subsession (berakhir sukses), abandonment subsession (gagal). Query kuat: purchase step = 1 → halaman awal pembelian sukses; abandonment step until end = 0 → halaman terakhir sesi gagal. Berlaku untuk **proses sekuensial apa pun**.

---

## 4. Aggregate Clickstream Fact Table

Kedua fact table besar; pertanyaan bulanan (mis. total kunjungan & revenue per grup demografi per bulan) memaksa meringkas jutaan baris. Buat **aggregate table** (Session Aggregate Fact): group by month, demographic type, entry page, session outcome. Hasilnya **<1%** ukuran asli → **≥100× lebih cepat**.

**Disiplin agregasi:** aggregate table terhubung ke **shrunken rollup dimension** yang **conform** dengan dimensi asli — month = subset conform dari calendar day; demographic = subset conform dari customer. (Bukan tabel sembarangan.)

---

## 5. Integrasi Clickstream ke Bisnis Retail

Web visitor clickstream diintegrasikan ke keluarga proses web retailer (supply chain, CRM, product/service orders, shipments, billing, payments, web site operations) lewat **conformed dimension**. Empat dimensi khas clickstream tak menyulitkan — justru kemampuannya **menjembatani dunia web & brick-and-mortar** adalah keuntungan (mis. "pengalaman web seperti apa yang menghasilkan customer yang membeli service policy tertentu?"). **Bus matrix** menjadi alat komunikasi: satu kolom = daftar undangan rapat conforming dimensi.

---

## 6. Profitability Lintas Channel (Termasuk Web)

Subject area paling menantang. Bangun **web profitability sebagai ekstensi sales transaction** (alokasi biaya ke tiap transaksi penjualan), **bukan** di atas clickstream (alokasi biaya ke sesi tanpa produk/penjualan jelas terlalu kontroversial). Manfaat: pandangan profitabilitas **lintas semua channel** (store, telesales, web), bukan cuma web.

- **Grain = tiap line item terjual** (sama dengan sales transaction). Dimensi: date, time, customer, channel, product, promotion, ticket number (DD).
- **Fakta P&L** (seperti Bab 6): units sold → gross revenue → dikurangi manufacturing allowance/marketing promotion/markdown → **net revenue** → dikurangi manufacturing & storage cost → **gross profit** → dikurangi freight/special deal/other overhead → **net profit**.
- **Kolom terhitung** (net revenue, gross profit, net profit): bila akses via **view** → cukup dihitung; bila akses tabel fisik langsung → **simpan fisik** (hindari salah hitung P&L yang kompleks).
- **Alokasi biaya** — tiap biaya di-*source*/estimasi terpisah; entri di baris = fraksi biaya total dialokasikan ke grain. Kualitas biaya bervariasi (rata-rata nasional tahunan vs activity-based). **Biaya website** (infrastruktur) sulit dialokasi langsung — skema: per jumlah halaman produk, per pages visited, atau per pembelian web aktual. Tim DW/BI **tidak** bertanggung jawab mengimplementasi ABC di seluruh organisasi — ambil data biaya terbaik saat ini, publikasikan P&L, perbaiki seiring waktu.

**Kekuatan P&L dalam kerangka dimensional:** menjawab **apa** yang profit **dan "mengapa"** (semua komponen revenue/cost/profit bisa di-slice per channel, customer segment, product line, promosi, waktu — bahkan gabungan: "customer profitable di tiap channel? promosi mana yang jalan di web tapi tidak di channel lain?").

---

## Ringkasan Bab

Bab 15 (web retailer) berpusat pada **clickstream**: sumber terdistribusi/stateless/anonim dengan tantangan mengidentifikasi **asal pengunjung** (clickthrough/referral), **sesi** (persistent cookie paling andal), dan **pengunjung** (cookie = komputer, bukan individu). Diperkenalkan **dimensi khas** (page, event, session, referral, step) dan **dua fact table**: **session-grained** (sengaja menyimpang dari grain terendah agar terkelola; dual role date universal/lokal; entry page) dan **page-event-grained** (grain terendah; step dimension 3 role; conformed facts page vs session seconds). **Aggregate table** dengan shrunken conformed dimension (≥100× lebih cepat). Integrasi ke bisnis retail via **conformed dimension** (jembatan web ↔ brick-and-mortar; bus matrix sebagai alat komunikasi). Ditutup dengan **profitability lintas channel** — P&L sebagai ekstensi sales transaction (bukan clickstream), dengan alokasi biaya (termasuk biaya website) dan kekuatan menjawab "apa & mengapa". Bab 16 (Insurance) akan merekap banyak pola pemodelan dimensional dari seluruh buku.
