# PRD — Sistem Otomasi Stok Bahan & Penjualan untuk Bisnis Florist/Crafting

| | |
|---|---|
| **Nama Produk (kerja)** | FloriStock — Otomasi Stok Berbasis Resep (BOM) |
| **Versi Dokumen** | 1.0 (Draft) |
| **Tanggal** | 6 Juni 2026 |
| **Disusun oleh** | Eko Nugroho (Bluebird Group) |
| **Sumber kebutuhan** | Percakapan Threads calon client (@tnziadnsf) di komunitas "data entry" |
| **Status** | Draft untuk diskusi dengan calon client |

---

## 1. Latar Belakang & Konteks

Calon client adalah pemilik bisnis **florist / crafting** (rangkaian bunga, bouquet, dan kerajinan tangan). Saat ini pencatatan stok bahan dan penjualan dilakukan manual, sehingga setiap kali ada penjualan, stok bahan tidak otomatis terpotong.

Inti permasalahan yang disampaikan langsung oleh calon client:

> "Semisal penjualan satu bunga mawar, butuh 9 kawat bulu dan 1 tangkai. Itu bisa otomatis berkurang di stock saat kita input 'mawar' atau mungkin saat input 'bouquet S'."

Artinya, satu produk jadi (mis. *mawar* atau *bouquet S*) tersusun dari **beberapa bahan baku** dengan kuantitas tertentu. Yang diinginkan adalah: saat sebuah produk terjual, sistem **otomatis mengurangi stok seluruh bahan penyusunnya** sesuai resep, tanpa harus mengurangi item satu per satu secara manual.

Validasi pain point ini juga muncul dari pelaku florist lain di thread yang sama (@latte_wood):

> "Bisa aja pake aplikasi kasir berbayar, tapi PR banget karena satu bouquet harus dikurangin item satu per satu (kertas tisu, cellophane, stik dll). Kalau orderan masih sedikit bisa, tapi kalau floristnya udah gede banget itu keteteran banget."

Calon client sendiri sudah membayangkan solusinya berbasis spreadsheet/pivot ("aku cari siapa tau ada yg ahli pake pivot dll buat otomatisinnya"), sehingga **MVP berbasis spreadsheet sangat realistis** dan sesuai ekspektasi mereka, dengan jalur upgrade ke aplikasi bila bisnis membesar.

---

## 2. Pernyataan Masalah

Pemilik florist tidak memiliki cara yang **cepat dan akurat** untuk mengetahui sisa stok bahan baku setelah penjualan, karena:

1. Satu produk jadi terdiri dari banyak bahan dengan kuantitas berbeda (struktur *Bill of Materials* / resep).
2. Pengurangan stok manual per bahan memakan waktu dan rawan salah, terutama saat order banyak.
3. Tidak ada peringatan dini saat bahan menipis, sehingga berisiko kehabisan stok di tengah pengerjaan order.
4. Tidak ada data historis penjualan & pemakaian bahan untuk perencanaan pembelian dan analisis margin.

**Dampak:** keteteran saat order ramai, stok bahan mati/kelebihan beli, potensi kehilangan penjualan karena kehabisan bahan, dan keputusan pembelian yang berbasis tebakan.

---

## 3. Tujuan & Sasaran

**Tujuan utama:** Setiap penjualan produk otomatis memotong stok seluruh bahan penyusunnya sesuai resep, secara akurat dan tanpa input manual per bahan.

Sasaran terukur (target setelah 1 bulan pemakaian):

- Waktu pencatatan penjualan turun dari menghitung manual per bahan menjadi **< 30 detik per transaksi** (cukup pilih produk + qty).
- **0 kejadian** kehabisan bahan mendadak di tengah order karena ada peringatan stok menipis.
- Pemilik bisa melihat **sisa stok real-time** dan **laporan pemakaian bahan** kapan saja.
- Akurasi stok tercatat vs fisik **≥ 95%** pada stock opname pertama.

**Non-tujuan (di luar cakupan awal):** integrasi marketplace/e-commerce, akuntansi penuh, multi-cabang, dan pembayaran online. Lihat Bagian 11.

---

## 4. Target Pengguna (Persona)

**Persona 1 — Pemilik/Admin Florist ("Bu Florist")**
Pelaku UMKM florist/crafting, melek aplikasi dasar (WhatsApp, Excel ringan), bukan teknis. Mengelola order dari DM/WA, merangkai sendiri atau dengan 1–3 staf. Butuh alat **sederhana, cepat, tidak ribet**, idealnya bisa diisi dari HP atau laptop. Inilah pengambil keputusan dan pengguna utama.

**Persona 2 — Staf Perangkai/Kasir (opsional)**
Membantu input penjualan saat ramai. Butuh tampilan input yang minim langkah dan sulit salah.

**Karakteristik penting:** volume order fluktuatif (musiman: Valentine, wisuda, Lebaran), banyak SKU bahan kecil (kawat, tangkai, tisu, cellophane, stik, pita), dan produk yang sering berupa paket/bundel (Bouquet S/M/L).

---

## 5. Konsep Inti: Resep / Bill of Materials (BOM)

Ini adalah jantung sistem. Setiap **Produk** memiliki **Resep**: daftar bahan + kuantitas yang dikonsumsi saat 1 unit produk terjual.

**Contoh dari calon client — Produk "Mawar" (1 tangkai mawar jadi):**

| Bahan | Qty per produk | Satuan |
|---|---|---|
| Kawat bulu | 9 | pcs |
| Tangkai | 1 | pcs |

**Contoh produk bundel — "Bouquet S":**

| Bahan / Komponen | Qty | Satuan |
|---|---|---|
| Mawar (produk jadi) | 5 | tangkai |
| Kertas tisu | 2 | lembar |
| Cellophane | 1 | lembar |
| Stik | 3 | pcs |
| Pita | 1 | meter |

Catatan penting: bundel bisa tersusun dari **produk lain** (mis. Bouquet S memakai 5 "Mawar"). Sistem harus mendukung **resep bertingkat (nested BOM)** — saat Bouquet S terjual, sistem menurunkan 5× resep Mawar (= 45 kawat bulu + 5 tangkai) **ditambah** bahan pembungkus bouquet. Ini fitur yang membedakan dari aplikasi kasir biasa yang hanya 1 level.

**Aturan pengurangan stok (logika inti):**

```
Saat transaksi penjualan dicatat (Produk P, jumlah N):
  untuk setiap bahan B dalam resep(P):
    stok[B] = stok[B] − (qty_B_per_P × N)
  jika resep(P) mengandung produk lain P2:
    terapkan resep(P2) secara rekursif × (qty_P2 × N)
  catat pemakaian ke log_pemakaian
  jika stok[B] ≤ titik_pesan_ulang(B): tandai PERLU RESTOCK
```

---

## 6. Kebutuhan Fungsional

**F1 — Master Bahan Baku.** CRUD daftar bahan: nama, satuan, stok awal, harga beli per satuan, titik pesan ulang (reorder point), supplier (opsional).

**F2 — Master Produk & Resep (BOM).** Buat produk jadi/bundel, tetapkan harga jual, dan susun resepnya (pilih bahan + qty). Mendukung resep bertingkat (produk berisi produk).

**F3 — Pencatatan Penjualan.** Pilih produk + jumlah → sistem otomatis menghitung dan memotong stok seluruh bahan penyusun (termasuk nested). Mendukung 1 transaksi berisi beberapa produk.

**F4 — Stok Real-time.** Tampilan sisa stok setiap bahan ter-update otomatis setelah setiap penjualan dan setiap pembelian/restock.

**F5 — Restock / Pembelian Bahan.** Catat penambahan stok (pembelian bahan) agar stok naik kembali; mencatat tanggal & biaya untuk analisis.

**F6 — Peringatan Stok Menipis.** Highlight/daftar bahan yang stoknya ≤ titik pesan ulang, sebagai "shopping list" otomatis.

**F7 — Laporan & Analisis.** Laporan penjualan per periode, pemakaian bahan per periode, produk terlaris, dan estimasi modal bahan (HPP) per produk untuk membantu lihat margin.

**F8 — Riwayat/Log.** Catat setiap transaksi penjualan & restock untuk audit dan koreksi (termasuk koreksi/penyesuaian stok manual saat opname).

**F9 — Penyesuaian Stok (Adjustment).** Koreksi manual stok untuk barang rusak, hilang, atau hasil stock opname, dengan catatan alasan.

---

## 7. Kebutuhan Non-Fungsional

- **Kemudahan pakai:** input penjualan ≤ 3 langkah; istilah dalam Bahasa Indonesia; cocok untuk pengguna non-teknis.
- **Aksesibilitas:** bisa diakses dari laptop; idealnya juga dari HP (Google Sheets / web).
- **Keandalan data:** perhitungan stok harus konsisten; ada log agar kesalahan bisa ditelusuri & dikoreksi.
- **Biaya rendah:** MVP memanfaatkan tool yang sudah dimiliki client (spreadsheet) untuk menekan biaya.
- **Skalabilitas:** desain data memungkinkan migrasi ke aplikasi/database bila volume order membesar.
- **Backup:** data tersimpan di cloud (mis. Google Drive) agar tidak hilang.

---

## 8. Usulan Solusi Bertahap

**Tahap 1 — MVP berbasis Spreadsheet (Google Sheets / Excel)**
Sesuai keinginan client ("pake pivot dll"). Struktur sheet:

- `Bahan` (master bahan + stok berjalan), `Produk`, `Resep` (relasi produk↔bahan↔qty), `Penjualan`, `Pembelian`, `Dashboard`.
- Pengurangan stok dihitung dengan formula (SUMIFS/QUERY) + PivotTable untuk laporan; stok berjalan = stok awal + total pembelian − total pemakaian.
- Input penjualan via form sederhana (Google Form atau baris input dengan dropdown produk).
- Kelebihan: cepat dibuat, murah, mudah dipahami client. Cocok untuk volume kecil–menengah.
- Keterbatasan: nested BOM dan banyak pengguna bersamaan agak terbatas; perlu desain formula yang rapi.

**Tahap 2 — Aplikasi ringan (bila bisnis membesar)**
Web app / app no-code (mis. AppSheet, atau aplikasi custom) dengan database. Mengatasi keterbatasan spreadsheet: input dari HP yang lebih nyaman, multi-user, nested BOM otomatis, notifikasi stok, dan laporan lebih kaya. Inilah jawaban atas kekhawatiran "kalau floristnya udah gede banget keteteran".

Rekomendasi: **mulai dari Tahap 1** untuk validasi cepat & biaya rendah, lalu naik ke Tahap 2 saat volume order tinggi.

---

## 9. Model Data (Ringkas)

**Bahan**: id, nama, satuan, stok_awal, stok_saat_ini, harga_beli, titik_pesan_ulang, supplier.

**Produk**: id, nama, tipe (produk/bundel), harga_jual, aktif.

**Resep (BOM)**: id, produk_id, komponen_id, komponen_tipe (bahan/produk), qty_per_unit. *(satu produk punya banyak baris resep)*

**Penjualan**: id, tanggal, produk_id, qty, total_harga, catatan.

**Pembelian/Restock**: id, tanggal, bahan_id, qty, biaya, supplier.

**Penyesuaian**: id, tanggal, bahan_id, qty_perubahan, alasan.

Relasi kunci: `Produk 1—* Resep *—1 Bahan/Produk`. Stok bahan = stok_awal + Σpembelian + Σpenyesuaian − Σpemakaian (dihitung dari penjualan × resep, rekursif).

---

## 10. Alur Pengguna Utama

**Alur A — Setup awal (sekali di depan):**
1. Input semua bahan + stok awal + titik pesan ulang.
2. Buat produk (Mawar, Bouquet S/M/L, dll) + harga jual.
3. Susun resep tiap produk (Mawar = 9 kawat bulu + 1 tangkai; Bouquet S = 5 Mawar + bahan pembungkus).

**Alur B — Transaksi harian (sering):**
1. Ada order → pilih produk + jumlah → simpan.
2. Sistem otomatis potong stok semua bahan penyusun (termasuk nested).
3. Dashboard stok ter-update; bahan menipis muncul di daftar belanja.

**Alur C — Restock:**
1. Beli bahan → catat di Pembelian (qty + biaya) → stok naik otomatis.

**Alur D — Stock opname (berkala):**
1. Hitung fisik → bandingkan dengan sistem → input penyesuaian bila ada selisih + alasan.

---

## 11. Cakupan MVP vs Pengembangan Lanjutan

**Masuk MVP (Tahap 1):** F1–F9 dalam bentuk spreadsheet, fokus pada otomasi pengurangan stok berbasis resep, dashboard stok, peringatan menipis, dan laporan dasar (pivot).

**Pengembangan lanjutan (di luar MVP awal):**
- Aplikasi/HP multi-user dengan database (Tahap 2).
- Notifikasi otomatis (WA/email) saat stok menipis.
- Manajemen order/pelanggan (CRM ringan) & status pengerjaan.
- Integrasi marketplace/e-commerce & pembayaran.
- Multi-cabang/multi-gudang.
- Pelacakan margin & laba detail, serta forecasting kebutuhan bahan musiman.

---

## 12. Metrik Keberhasilan

- **Adopsi:** client memakai sistem untuk ≥ 90% transaksi setelah 2 minggu.
- **Efisiensi:** waktu catat penjualan < 30 detik/transaksi.
- **Akurasi:** selisih stok sistem vs fisik ≤ 5% saat opname pertama.
- **Pencegahan kehabisan:** 0 kejadian kehabisan bahan mendadak karena peringatan berfungsi.
- **Kepuasan:** client menyatakan tidak lagi "keteteran" saat order ramai.

---

## 13. Asumsi & Pertanyaan untuk Calon Client

**Asumsi sementara:** client nyaman dengan Google Sheets; volume order saat ini kecil–menengah; semua resep produk bisa didefinisikan dengan kuantitas tetap.

**Perlu dikonfirmasi ke calon client sebelum eksekusi:**
1. Berapa rata-rata jumlah order per hari/minggu (untuk pilih Tahap 1 vs langsung 2)?
2. Berapa banyak jenis bahan dan jenis produk/bundel yang dikelola?
3. Apakah input dilakukan dari HP, laptop, atau keduanya? Berapa orang yang input?
4. Apakah perlu mencatat data pelanggan/order, atau cukup stok + penjualan?
5. Apakah harga bahan & resep sering berubah?
6. Preferensi tool: Google Sheets, Excel, atau terbuka untuk aplikasi?
7. Anggaran & tenggat (mis. menjelang musim ramai seperti Valentine/wisuda)?

---

*Dokumen ini disusun sebagai dasar diskusi dengan calon client. Setelah pertanyaan di Bagian 13 terjawab, PRD dapat difinalkan beserta scope kerja dan estimasi.*
