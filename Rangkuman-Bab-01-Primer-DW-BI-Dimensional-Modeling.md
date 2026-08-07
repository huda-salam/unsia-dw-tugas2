# Bab 1 — Primer Data Warehousing, Business Intelligence, dan Dimensional Modeling

> *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
> Rangkuman bahasa Indonesia · kedalaman sedang

Bab ini adalah fondasi seluruh buku: menjelaskan **untuk apa** sistem DW/BI dibangun, **konsep inti** pemodelan dimensional (fact & dimension), **arsitektur** Kimball beserta alternatifnya, dan **mitos-mitos** yang masih beredar.

---

## 1. Tujuan Sistem DW/BI

Kebutuhan bisnis terhadap data sudah sama selama puluhan tahun ("kami punya banyak data tapi sulit mengaksesnya", "kami habiskan rapat berdebat siapa yang punya angka benar"). Dari keluhan itu, Kimball menurunkan **tujuan wajib** sebuah sistem DW/BI:

- **Mudah diakses** — konten harus dipahami pengguna bisnis (bukan cuma developer); struktur dan label meniru cara berpikir & kosakata pengguna. Ringkasnya: *simple and fast*.
- **Konsisten** — data dirakit dari banyak sumber, dibersihkan, dijamin kualitasnya. Label/definisi seragam: nama sama berarti hal sama; hal beda dilabeli beda.
- **Adaptif terhadap perubahan** — kebutuhan, kondisi bisnis, data, dan teknologi pasti berubah. Perubahan tidak boleh merusak data/aplikasi yang sudah ada.
- **Tepat waktu** — makin sering dipakai untuk keputusan operasional, makin cepat data mentah harus jadi informasi yang bisa ditindaklanjuti.
- **Aman** — "permata mahkota" organisasi tersimpan di sini; akses ke informasi rahasia harus dikendalikan.
- **Fondasi tepercaya untuk pengambilan keputusan** — output terpenting adalah *keputusan* yang diambil berdasar data. Istilah lamanya masih paling tepat: *decision support system*.
- **Diterima komunitas bisnis** — pemakaian DW/BI sering bersifat opsional; kalau bisnis tidak memakainya, proyek gagal, sehebat apa pun arsitekturnya.

> **Poin kunci:** dua tujuan terakhir (fondasi tepercaya & penerimaan bisnis) paling penting sekaligus paling sering diabaikan. Praktisi DW/BI harus jadi "hibrida DBA/MBA" — satu kaki di IT, satu kaki di dunia bisnis.

**Metafora penerbitan:** manajer DW/BI ibarat pemimpin redaksi majalah — harus memahami pembaca (pengguna), menyajikan konten yang menarik & berkualitas, dan terus menjaga relevansi. Fokusnya pada pembaca, bukan pada teknologi.

---

## 2. Pengantar Dimensional Modeling

Pemodelan dimensional adalah teknik yang diterima luas untuk menyajikan data analitik karena memenuhi **dua kebutuhan sekaligus**:

1. Data yang **mudah dipahami** pengguna bisnis.
2. **Performa query yang cepat.**

Analoginya: eksekutif berkata "kami menjual produk di berbagai pasar dan mengukur kinerja dari waktu ke waktu". Perancang menangkap tiga sumbu: **produk, pasar, waktu** → data dibayangkan sebagai **kubus** yang bisa di-*slice and dice*. Kesederhanaan adalah kunci: model yang mulai sederhana berpeluang tetap sederhana. (Einstein: *"Make everything as simple as possible, but not simpler."*)

**Dimensional vs Normalized (3NF):**

| Aspek | Model Normalized (3NF) | Model Dimensional |
|---|---|---|
| Tujuan | Menghapus redundansi, ideal untuk transaksi operasional (insert/update satu tempat) | Kemudahan pemahaman & kecepatan query analitik |
| Bentuk | Ratusan tabel — "seperti peta jalan tol LA" | Struktur bintang yang ringkas |
| Untuk query BI | Terlalu rumit; membingungkan pengguna & membebani optimizer | Intuitif dan cepat |

> Model dimensional memuat **informasi yang sama** dengan model normalized, hanya dikemas agar mudah dipahami, cepat di-query, dan tahan perubahan.

**Star Schema vs OLAP Cube:**
- **Star schema** — implementasi dimensional di database relasional (RDBMS).
- **OLAP cube** — implementasi di database multidimensional; performa lebih unggul via prakalkulasi/agregasi, fungsi analitik lebih kaya dari SQL, tapi ada "harga" saat load data besar.
- **Rekomendasi Kimball:** simpan data **atomik/detail** di star schema; OLAP cube (opsional) diisi dari star schema tersebut. Sebagian besar teknik di buku ini dijelaskan dalam konteks star schema relasional.

---

## 3. Fact Table — Tabel Pengukuran

Fact table menyimpan **hasil pengukuran** dari event proses bisnis (mis. penjualan ritel).

- **Satu baris = satu event pengukuran.** Prinsip fundamental: satu event fisik ↔ satu baris fact.
- **Grain (granularitas)** = tingkat detail satu baris (mis. "satu produk per transaksi penjualan"). **Semua baris dalam satu fact table harus punya grain yang sama** agar pengukuran tidak salah dihitung ganda.
- **Fakta paling berguna: numerik & aditif** (mis. jumlah rupiah penjualan), karena BI hampir selalu menjumlahkan ribuan/jutaan baris.
  - **Semi-aditif** — tidak bisa dijumlahkan lintas waktu (mis. saldo rekening).
  - **Non-aditif** — tidak pernah bisa dijumlahkan (mis. harga satuan) → pakai count/average.
- **Sparse (jarang):** tidak ada aktivitas → tidak ada baris. Jangan isi nol. Meski jarang, fact table menempati ≥90% ruang model — **dalam** (banyak baris) tapi **sempit** (sedikit kolom).
- **Tiga jenis grain fact table:** *transaction*, *periodic snapshot*, *accumulating snapshot* (dibahas di Bab 3 & 4).
- **Foreign key & integritas:** setiap FK di fact cocok dengan primary key dimensi (referential integrity). Primary key fact biasanya **composite key** (gabungan sebagian FK). *Setiap tabel dengan composite key adalah fact table; fact table mengekspresikan relasi many-to-many.*

---

## 4. Dimension Table — Konteks Deskriptif

Dimensi adalah pasangan tak terpisahkan dari fact table; memuat konteks tekstual **"siapa, apa, di mana, kapan, bagaimana, dan mengapa"** dari sebuah event.

- Sering punya **banyak kolom/atribut** (lazim 50–100 atribut).
- **Lebar** (banyak kolom teks) tapi **baris lebih sedikit** dibanding fact table.
- Setiap dimensi punya **satu primary key** sebagai dasar referential integrity.
- Atribut dimensi adalah sumber utama untuk **filter, group by, dan label** pada laporan — makin kaya dimensinya, makin kuat sistem DW/BI-nya.

> **Star schema** = satu fact table di tengah, dikelilingi dimensi-dimensi (mis. Date, Product, Store, Customer, Promotion). Query BI mengakses fact melalui dimensi yang di-join.

---

## 5. Arsitektur DW/BI Kimball

Empat komponen inti:

1. **Operational Source Systems** — sistem operasional pencatat transaksi. Dianggap **di luar** data warehouse (kendali minim). Prioritasnya performa & ketersediaan transaksi, bukan query analitik luas; menyimpan sedikit histori.
2. **ETL System (Extract, Transform, Load)** — seluruh proses antara sumber dan area presentasi.
   - **Extract:** membaca & menyalin data yang dibutuhkan.
   - **Transform:** pembersihan (koreksi ejaan, resolusi konflik domain, isi data hilang), penggabungan sumber, deduplikasi.
   - **Load:** menyusun & memuat ke model dimensional target (penetapan surrogate key, lookup kode, denormalisasi dimensi).
   - Tujuan desain: **throughput, integritas, konsistensi.** Boleh memakai struktur normalized untuk membantu ETL, **tapi struktur itu harus tertutup dari query pengguna** karena mengalahkan tujuan kemudahan & performa.
3. **Presentation Area** — tempat data diorganisir & tersedia untuk query langsung. Aturan Kimball:
   - Harus **dimensional** (star schema atau OLAP cube).
   - Harus memuat **data atomik/detail** (boleh ditambah agregat untuk performa) — jangan hanya ringkasan.
   - Diorganisir per **proses bisnis** (bukan per departemen).
   - Memakai **conformed dimensions** sesuai **enterprise data warehouse bus architecture** (dibahas Bab 4). Tanpa dimensi yang di-*conform*, model jadi aplikasi terisolasi (*stovepipe*).
4. **BI Applications** — seluruh kapabilitas untuk pengguna: query ad hoc, laporan standar, aplikasi analitik, data mining/modeling. Sebagian besar pengguna mengaksesnya lewat aplikasi/template berparameter, bukan menulis query langsung.

**Metafora restoran:**
- **ETL = dapur (back room):** mengubah bahan mentah jadi hidangan siap saji; dirancang untuk throughput, kualitas, konsistensi & integritas; **tertutup dari tamu.** Aturan sekali dibuat oleh profesional ETL, bukan diulang tiap pengguna.
- **Presentation Area & BI = ruang makan (front room):** yang dilihat & disentuh pengunjung. Dinilai dari kualitas "makanan" (data), "dekor", "layanan", dan "biaya". Manajer harus proaktif memantau kepuasan pengguna.

> Prinsip: geser pekerjaan dari front room (yang dikerjakan berulang oleh banyak pengguna) ke back room ETL (dikerjakan sekali oleh tim ETL).

---

## 6. Arsitektur Alternatif

- **Independent Data Mart** — tiap departemen membangun data mart sendiri secara terisolasi, tanpa berbagi/integrasi lintas perusahaan. Menimbulkan angka tak konsisten & *stovepipe*. **Tidak dianjurkan.**
- **Corporate Information Factory (CIF) / Inmon (hub-and-spoke)** — data masuk ke **EDW ternormalisasi (3NF)** yang bersifat **wajib**, lalu data mart dimensional (sering teragregasi & departemental) diisi dari EDW. Catatan Kimball: *normalisasi ≠ integrasi*; bentuk CIF paling murni dinilai tidak praktis karena data atomik "terkunci" di struktur sulit di-query.
- **Hybrid (CIF + Kimball)** — EDW 3NF (tertutup dari pengguna) hanya jadi sumber untuk **area presentasi Kimball** (dimensional, atomik, per proses bisnis, bus architecture). Cocok bila sudah terlanjur investasi EDW 3NF; jika mulai dari nol, cenderung lebih mahal & lama karena data bergerak & disimpan berulang.

---

## 7. Mitos Dimensional Modeling (dan Faktanya)

1. **"Hanya untuk data ringkasan"** — Salah. Justru harus menyediakan data paling detail agar pengguna bisa roll-up sesuai pertanyaan; ringkasan hanya pelengkap performa.
2. **"Bersifat departemental, bukan enterprise"** — Salah. Model diorganisir per **proses bisnis** (order, invoice, service call), bukan per departemen.
3. **"Tidak skalabel"** — Salah. Fact table lazim bermiliar baris (pernah dilaporkan 2 triliun baris); vendor database mengoptimalkannya.
4. **"Hanya untuk pemakaian yang bisa diprediksi"** — Salah. Rancang berdasar **event pengukuran** (stabil), bukan daftar laporan (selalu berubah). Grain paling detail = fleksibilitas maksimum. *"God is in the details."*
5. **"Tidak bisa diintegrasikan"** — Salah, asalkan patuh pada **bus architecture** dengan **conformed dimensions** (master data terpusat di ETL, dipakai ulang lintas model).

---

## 8. Berpikir Dimensional Lebih Luas + Pertimbangan Agile

Pemodelan dimensional tidak hanya soal desain tabel. Sepanjang proyek, **berpikirlah dalam kerangka proses bisnis**:
- **Requirement:** sintesis temuan di seputar proses bisnis, bukan sekadar daftar laporan.
- **Scope:** satu proses bisnis per proyek/iterasi.
- **Prioritas:** ranking proses berdasar **nilai bisnis** & **kelayakan**; kerjakan yang impact & feasibility-nya tertinggi.
- **Arsitektur data:** deliverable utamanya **enterprise data warehouse bus matrix** (Bab 4).
- **Data governance:** fokus dulu pada dimensi utama (date, customer, product, employee, dst.).

**Agile:** banyak prinsip agile selaras dengan Kimball (fokus nilai bisnis, kolaborasi dengan bisnis, iteratif, adaptif). Kritik umum terhadap agile — minim perencanaan/arsitektur & tantangan governance — dijawab oleh **bus matrix** sebagai *master plan*. **Conformed dimensions justru memungkinkan** pengembangan agile: begitu portofolio dimensi tersedia, pengembangan makin cepat (tinggal menambah fact table). Yang harus dihindari: memakai agile untuk membuat solusi terisolasi (*stovepipe*).

---

## Ringkasan Bab

Bab 1 menetapkan **tujuan** sistem DW/BI (mudah, konsisten, adaptif, tepat waktu, aman, tepercaya, diterima bisnis), memperkenalkan **konsep inti** dimensional modeling (**fact table** untuk pengukuran + **dimension table** untuk konteks, dirakit jadi **star schema**), memaparkan **arsitektur Kimball** (source → ETL → presentation area → BI, direkatkan **bus architecture** dengan **conformed dimensions**) beserta metafora restoran dan **alternatif** (independent data mart, CIF/Inmon, hybrid), lalu meluruskan **lima mitos** dan mendorong pembaca **berpikir dimensional** sejak awal proyek. Bab 2 akan memberi tur cepat pola & teknik pemodelan dimensional, sebelum studi kasus pertama di Bab 3 (Retail Sales).
