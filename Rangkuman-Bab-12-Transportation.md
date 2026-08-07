# Bab 12 — Transportation (Transportasi)

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Studi kasus **maskapai penerbangan** yang mengangkat tema "voyage" (perjalanan bertahap). Konsep utama: **fact table pada beberapa level granularitas** (leg < segment < trip < itinerary), penggunaan **role-playing dimension** yang masif (date, time, airport), **menautkan segment jadi trip**, **menggabungkan dimensi berkorelasi** (class of service), penanganan **origin/destination**, serta isu **lokalisasi**: kalender per negara (outrigger), **date/time di banyak zona waktu**, multi-currency & multi-bahasa.

---

## 1. Multiple Fact Table Granularity

Data maskapai bisa dimodelkan di beberapa level (tiap level punya metrik & peminat berbeda):

- **Leg** — pesawat lepas landas di satu bandara & mendarat di bandara lain **tanpa henti**. Peminat: *capacity planning* & *flight scheduling* (load factor, durasi, keterlambatan menit). Level **paling granular**.
- **Segment** — satu **flight number** oleh satu pesawat; bisa 1+ leg. Contoh SFO→MSP transit DEN **tanpa ganti pesawat/nomor** = 1 segment, 2 leg. Segment = **baris di kupon tiket**; revenue & mileage ditentukan di sini. Peminat: marketing & revenue.
- **Trip** — gambaran **permintaan pelanggan sebenarnya** (SFO→MSP dengan ganti pesawat di DEN = 2 segment, tapi 1 trip; penumpang hanya ingin SFO→MSP). Peminat: sales & marketing.
- **Itinerary** — seluruh tiket / confirmation number reservasi.

**Keputusan:** mulai di **grain segment** — level terendah dengan **metrik revenue bermakna**. (Alternatif: alokasikan metrik segment ke leg berdasar mileage.) Leg-level akan digarap kemudian untuk capacity planner — **conformed dimension** dari iterasi ini akan dipakai ulang. **Grain = satu baris per boarding pass** yang dikumpulkan dari penumpang.

**Role-playing masif** — skema segment-level memakai banyak view dari satu tabel fisik date/time/airport: scheduled/actual departure date & time (role-playing), segment origin/destination airport, dll. (teknik dari Bab 6). Degenerate: confirmation number, ticket number, segment sequence, flight number.

---

## 2. Menautkan Segment ke Trip

Grain segment **menyembunyikan** sifat trip — sulit menjawab "ke mana frequent flyer pergi?". Mengambil semua segment lalu mengurutkannya pun sulit menemukan titik awal/akhir trip (kebanyakan itinerary lengkap mulai & berakhir di bandara sama).

**Solusi:** tambahkan **dua role-playing airport dimension** — *trip origin* & *trip destination* — sambil **tetap** di grain segment. Ditentukan saat ekstraksi dengan mencari **stopover** (henti >4 jam = definisi resmi maskapai). ⚠️ Hati-hati saat meringkas per trip: sebagian dimensi (fare basis, class of service) **tak berlaku** di level trip.

**Aggregate trip table** (opsional) — bila pengguna sering melihat level trip: buat fact table agregat di grain trip (fakta seperti trip total base fare, trip total taxes, jumlah segment). Hanya dibuat bila ada masalah performa/usability nyata — bila trip rata-rata 3 segment, percepatan hanya ~3× → mungkin tak sepadan.

**Related fact tables** — leg-grained flight activity (durasi aktual/blocked, delay, fuel weight), reservations, issued tickets, dan **revenue & availability snapshot** per flight (90 hari terakhir menjelang keberangkatan, dengan dimensi **"days prior to departure"** untuk membandingkan flight serupa di milestone standar mis. 60 hari sebelum).

---

## 3. Ekstensi ke Industri Lain

**Cargo shipper** — mirip skema maskapai. **Grain = container pada satu bill-of-lading pada satu leg** perjalanan. Dimensi: ship mode (jenis perusahaan & vessel), container (ukuran, butuh listrik/pendingin), **commodity** (via *harmonized commodity codes* — conformed dimension master yang dipakai bea cukai). Pihak (consignor, foreign/domestic transporter & consolidator, shipper, consignee) = **7 peran** dari satu master **business entity dimension**. Port = **4 peran** (voyage/leg × origin/destination). Bill-of-lading = degenerate. Fee/tarif berlaku per leg.

**Travel services** — lengkapi flight activity dengan fact table hotel & rental mobil (berbagi dimensi date & customer). **Grain hotel stay = seluruh menginap**; grain rental = seluruh episode sewa. (Untuk rantai hotel sendiri, skema jauh lebih kaya.)

---

## 4. Menggabungkan Dimensi Berkorelasi

Prinsip umum: relasi many-to-many antar dua grup atribut → dimensi terpisah. **Tapi kadang lebih baik digabung.**

**Class of service** — pengguna ingin analisis kelas **dibeli** (booking) & kelas **diterbangkan** (flown), plus indikator upgrade/downgrade. Reaksi awal: dua role-playing dimension + FK ketiga untuk upgrade indicator (agar BI tak perlu logika mengenali banyak skenario upgrade). **Tapi** class dimension hanya **4 baris** (first, business, premium economy, economy) & upgrade indicator hanya **3 baris**. Karena baris sangat sedikit → **gabung** jadi **satu** class of service dimension (~8 baris) berisi class purchased, class flown, purchased-flown group, dan class change indicator. Lebih sederhana daripada tiga FK.

---

## 5. Origin & Destination

Untuk **origin/destination airport**, volume data lebih besar → role-playing **terpisah** lebih praktis. Namun pengguna butuh atribut yang **bergantung pada kombinasi** origin+destination: jarak antar city-pair, tipe rute (domestic/trans-Atlantic), dan aktivitas non-arah SFO↔DEN (tak peduli arah). Ini sulit dengan dimensi terpisah, dan **label rute tak standar** (SFO-DEN vs DEN-SFO vs …).
> Jangan biarkan kode aplikasi BI membuat label tak konsisten — simpan label standar di **dimensi**.

Dua opsi: (a) tambahkan **city-pair route dimension** terpisah (directional/nondirectional route name, type, distance band, dom-intl indicator, transocean indicator); atau (b) gabungkan atribut origin+destination+route jadi satu dimensi. Cartesian product teoretis besar, tapi nyatanya jauh lebih kecil (maskapai tak terbang antar semua bandara); dengan puluhan atribut origin **dan** destination identik, biasanya lebih tergoda **memisah**. Catatan: **bridge table tidak diperlukan** di sini — relasi many-to-many origin/destination sudah terwakili bersih di fact table.

---

## 6. Isu Lokalisasi Date & Time

**Kalender per negara (outrigger).** Untuk multinasional, date dimension utama memuat atribut kalender **generik** (bila lintas Gregorian/Hebrew/Islamic/Chinese → empat set day/month/year). **Country-specific date outrigger** (key = date key + country code) menambah atribut spesifik negara (holiday/season name, civil/religious holiday flag). Bisa di-join sebagai outrigger atau langsung ke fakta; dengan interface yang meminta pengguna memilih negara, atribut ini seolah "ditempel" ke date table (melihat kalender lewat mata satu negara). Mirip penanganan multiple fiscal calendar di Bab 7. Rumit bila ada holiday lokal yang beda tanggal di bagian negara berbeda.

**Date/time di banyak zona waktu.** Tangkap **keduanya**: waktu **lokal** (memahami timing relatif waktu setempat) & waktu **standar** (GMT/UTC/Zulu — melihat sifat simultan transaksi lintas bisnis). ⚠️ Dunia punya **lebih dari 24 zona** (China 1 zona, India offset 5,5 jam, Nepal offset seperempat jam, plus DST). Offset **tak bisa** sekadar disimpan di fakta/time/airport dimension karena **bergantung pada lokasi & tanggal**. **Solusi:** sertakan **date & time-of-day dimension terpisah** untuk lokal & ter-ekuivalensi (mis. Departure Date Key + GMT Departure Date Key + Departure Time-of-Day Key + GMT Departure Time-of-Day Key). Time-of-day dimension mendukung pengelompokan shift/rush period.

**Localization recap.** Lokalisasi mencakup: zona waktu & kalender internasional (bab ini), multi-currency (Bab 6), multi-bahasa (Bab 8), plus **terjemahan teks UI** di BI tool (via text database yang dikonfigurasi per environment). Rumit: terjemahan Inggris→Eropa memanjangkan string (bisa memaksa redesain), Arab kanan-ke-kiri, banyak bahasa Asia berbeda total. (Analogi menarik: menara kontrol & pilot sedunia memakai **satu bahasa (Inggris) & satu satuan (feet)** untuk komunikasi kritis.)

---

## Ringkasan Bab

Bab 12 (maskapai/voyage) mengajarkan **multiple fact table granularity** (leg < segment < trip < itinerary; mulai di segment = grain terendah dengan revenue bermakna, grain fakta = per boarding pass), **role-playing masif** (date/time/airport), **menautkan segment→trip** (dua airport role-playing trip origin/destination, stopover >4 jam, tetap grain segment), dan **related fact tables** (leg ops, reservations, revenue/availability snapshot dengan "days prior to departure"). Ekstensi ke **cargo shipper** (grain container×bill-of-lading×leg, business entity 7 peran, port 4 peran, harmonized commodity codes) & **travel services**. Teknik desain: **menggabungkan dimensi berkorelasi** (class of service ~8 baris jadi satu dimensi) dan penanganan **origin/destination** (city-pair route dimension, tanpa bridge). Ditutup dengan **lokalisasi**: kalender per negara (outrigger), **date/time multi zona waktu** (lokal + ekuivalen GMT via dimensi terpisah), multi-currency, multi-bahasa, dan terjemahan UI. Bab 13 (Education) akan membahas factless fact table & accumulating snapshot dalam konteks pendidikan.
