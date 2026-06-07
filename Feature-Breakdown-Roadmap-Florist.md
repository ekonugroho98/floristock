# Feature Breakdown & Roadmap — Sistem Web FloriStock

| | |
|---|---|
| **Proyek** | FloriStock — Otomasi Stok Berbasis Resep (BOM) — Versi Web |
| **Dokumen** | Feature Breakdown & Development Roadmap |
| **Versi** | 1.0 (Draft) |
| **Tanggal** | 6 Juni 2026 |
| **Acuan** | PRD-Sistem-Otomasi-Stock-Penjualan-Florist.md |
| **Disusun oleh** | Eko Nugroho (Bluebird Group) |

---

## 1. Cara Membaca Dokumen Ini

Breakdown disusun tiga lapis: **Epic** (kelompok besar fitur) → **User Story** (kebutuhan dari sudut pandang pengguna) → **Task Teknis** (pekerjaan dev konkret). Setiap fitur diberi prioritas **MoSCoW**:

- **MUST** — wajib ada di MVP; tanpa ini sistem tidak berfungsi.
- **SHOULD** — penting, tapi MVP masih jalan tanpanya; masuk segera setelah MVP.
- **COULD** — nilai tambah; dikerjakan bila waktu/anggaran ada.
- **WON'T (now)** — sengaja ditunda; di luar cakupan saat ini.

Setiap story juga diberi **estimasi relatif** (story point / S-M-L) sebagai panduan kasar, bukan janji waktu.

---

## 2. Peta Epic

| # | Epic | Tujuan | Prioritas |
|---|------|--------|-----------|
| E1 | Manajemen Bahan Baku | Master data bahan + stok | MUST |
| E2 | Manajemen Produk & Resep (BOM) | Definisikan produk & resep penyusun | MUST |
| E3 | Pencatatan Penjualan + Auto-Deduct Stok | Jantung sistem: jual → stok turun otomatis | MUST |
| E4 | Manajemen Stok (Restock & Penyesuaian) | Stok naik kembali & koreksi opname | MUST |
| E5 | Dashboard & Peringatan Stok | Pantau stok real-time + alert menipis | MUST/SHOULD |
| E6 | Laporan & Analisis | Penjualan, pemakaian bahan, HPP, terlaris | SHOULD |
| E7 | Autentikasi & Multi-User | Login, peran admin/staf | SHOULD |
| E8 | Notifikasi & Integrasi | WA/email alert, ekspor, marketplace | COULD/WON'T |

---

## 3. Breakdown per Epic

### E1 — Manajemen Bahan Baku  · MUST

**US-1.1 (M)** — Sebagai admin, saya bisa menambah/ubah/hapus bahan (nama, satuan, stok awal, harga beli, titik pesan ulang, supplier).
- Tabel `bahan` + migrasi DB.
- API CRUD bahan (create/read/update/delete + soft delete).
- Halaman daftar bahan (tabel, cari, filter, paginasi).
- Form tambah/edit bahan + validasi (satuan, angka non-negatif).

**US-1.2 (S)** — Sebagai admin, saya bisa melihat sisa stok terkini tiap bahan di daftar.
- Kolom stok berjalan dihitung dari mutasi (lihat E4).

**US-1.3 (S)** — Sebagai admin, saya bisa set titik pesan ulang (reorder point) per bahan.
- Field reorder point + indikator visual saat stok ≤ titik.

---

### E2 — Manajemen Produk & Resep (BOM)  · MUST

**US-2.1 (M)** — Sebagai admin, saya bisa membuat produk jadi/bundel (nama, harga jual, status aktif).
- Tabel `produk` + API CRUD.
- Halaman daftar produk + form.

**US-2.2 (L)** — Sebagai admin, saya bisa menyusun resep produk: pilih beberapa bahan + kuantitas per unit.
- Tabel `resep` (produk_id, komponen_id, komponen_tipe, qty).
- UI builder resep (tambah baris bahan + qty, hapus baris).
- Validasi: qty > 0, komponen tidak kosong.

**US-2.3 (L)** — Sebagai admin, saya bisa menyusun **resep bertingkat** (bundel berisi produk lain, mis. Bouquet S = 5 Mawar + bahan pembungkus).
- Komponen resep bisa bertipe `bahan` ATAU `produk`.
- **Proteksi loop**: cegah resep melingkar (A berisi B, B berisi A) — validasi saat simpan.
- Fungsi "ratakan resep" (flatten) untuk hitung total bahan mentah.

**US-2.4 (S)** — Sebagai admin, saya bisa melihat estimasi modal bahan (HPP) per produk dari resepnya.
- Hitung Σ(qty × harga beli bahan), rekursif untuk nested.

---

### E3 — Pencatatan Penjualan + Auto-Deduct Stok  · MUST (jantung sistem)

**US-3.1 (M)** — Sebagai kasir/admin, saya bisa mencatat penjualan: pilih produk + jumlah (boleh beberapa produk dalam 1 transaksi).
- Tabel `penjualan` + `penjualan_item`.
- Halaman input penjualan (≤ 3 langkah, dropdown produk + qty).

**US-3.2 (L)** — Saat penjualan disimpan, stok semua bahan penyusun otomatis berkurang sesuai resep, **termasuk nested BOM**.
- **Service `applyBOMDeduction()`**: telusuri resep secara rekursif, hitung total konsumsi tiap bahan mentah, kurangi stok.
- Jalankan dalam **transaksi DB (atomic)** — semua berhasil atau semua dibatalkan.
- Catat ke `mutasi_stok` (jenis: penjualan) untuk audit.
- *Contoh uji wajib:* jual 1 "Mawar" → kawat bulu −9, tangkai −1. Jual 1 "Bouquet S" (5 Mawar) → kawat bulu −45, tangkai −5, + bahan pembungkus.

**US-3.3 (S)** — Sebagai kasir, saya diberi peringatan bila stok bahan tidak cukup untuk memenuhi penjualan.
- Cek ketersediaan sebelum commit; tampilkan bahan yang kurang.
- Opsi: blokir, atau izinkan stok minus dengan peringatan (sesuai preferensi client).

**US-3.4 (S)** — Sebagai admin, saya bisa membatalkan/koreksi transaksi penjualan dan stok dikembalikan.
- Reversal mutasi (kembalikan stok) + log alasan.

---

### E4 — Manajemen Stok: Restock & Penyesuaian  · MUST

**US-4.1 (M)** — Sebagai admin, saya bisa mencatat pembelian/restock bahan (qty, biaya, tanggal, supplier) dan stok naik otomatis.
- Tabel `pembelian` + mutasi stok (jenis: pembelian).

**US-4.2 (S)** — Sebagai admin, saya bisa melakukan penyesuaian stok manual (rusak/hilang/hasil opname) dengan catatan alasan.
- Tabel/entri `penyesuaian` + mutasi (jenis: adjustment).

**US-4.3 (M)** — Stok berjalan dihitung konsisten dari ledger mutasi.
- Pola **stock ledger**: stok = Σ masuk − Σ keluar. Hindari menyimpan satu angka yang ditimpa-timpa; pakai mutasi agar bisa ditelusuri.

---

### E5 — Dashboard & Peringatan Stok  · MUST/SHOULD

**US-5.1 (M)** — Sebagai admin, saya melihat dashboard sisa stok semua bahan real-time.
- Halaman dashboard + ringkasan.

**US-5.2 (M)** — Sebagai admin, saya melihat daftar bahan yang menipis (≤ titik pesan ulang) sebagai "shopping list".
- Query bahan di bawah reorder point + highlight.

**US-5.3 (S)** — Ringkasan cepat: penjualan hari ini, produk terlaris minggu ini.
- Widget angka ringkas di dashboard.

---

### E6 — Laporan & Analisis  · SHOULD

**US-6.1 (S)** — Laporan penjualan per periode (harian/mingguan/bulanan).
**US-6.2 (S)** — Laporan pemakaian bahan per periode (untuk perencanaan beli).
**US-6.3 (C)** — Laporan margin/HPP per produk & per periode.
**US-6.4 (C)** — Ekspor laporan ke Excel/CSV.

---

### E7 — Autentikasi & Multi-User  · SHOULD

**US-7.1 (S)** — Login aman (email + password).
**US-7.2 (S)** — Peran **Admin** (kelola semua) vs **Staf/Kasir** (hanya input penjualan).
**US-7.3 (C)** — Log aktivitas pengguna (siapa input apa, kapan).

---

### E8 — Notifikasi & Integrasi  · COULD / WON'T (now)

**US-8.1 (C)** — Notifikasi WA/email saat stok menipis.
**US-8.2 (W)** — Manajemen pelanggan/order (CRM ringan) + status pengerjaan.
**US-8.3 (W)** — Integrasi marketplace/e-commerce & pembayaran online.
**US-8.4 (W)** — Multi-cabang/multi-gudang.

---

## 4. Definisi MVP

MVP = **E1 + E2 + E3 + E4 + E5 (US-5.1 & US-5.2)**. Dengan ini client sudah bisa: kelola bahan & resep, catat penjualan dengan auto-deduct nested BOM, restock, dan memantau stok + peringatan menipis. Inilah yang langsung menjawab keluhan "keteteran kurangi item satu-satu".

E6 & E7 menyusul tepat setelah MVP stabil. E8 ditunda.

---

## 5. Roadmap / Urutan Pengerjaan (Milestone)

> Estimasi waktu bersifat indikatif untuk 1 developer; sesuaikan dengan kapasitas tim.

**M0 — Fondasi (≈ 3–5 hari)**
Setup repo, framework, DB, autentikasi dasar, struktur folder, deploy pipeline minimal. Skema DB awal (bahan, produk, resep, mutasi).

**M1 — Master Data (≈ 1 minggu) → E1 + E2 (tanpa nested dulu)**
CRUD bahan & produk, builder resep 1 level. Bisa input data, belum ada transaksi.

**M2 — Inti Transaksi & BOM (≈ 1.5–2 minggu) → E3 + E2 nested (US-2.3) + E4**
Pencatatan penjualan, **engine auto-deduct rekursif**, restock, penyesuaian, stock ledger. **Ini milestone paling berisiko/teknis — alokasikan buffer & test ketat.**

**M3 — Dashboard & Peringatan (≈ 4–5 hari) → E5**
Dashboard stok real-time + daftar bahan menipis + ringkasan. **→ Akhir M3 = MVP siap diuji client.**

**M4 — Laporan & Multi-User (≈ 1 minggu) → E6 + E7**
Laporan periode, ekspor, peran admin/staf.

**M5 — Penyempurnaan (≈ menyesuaikan) → E8 terpilih + polish UX**
Notifikasi, perbaikan dari feedback client, optimasi.

---

## 6. Risiko & Catatan Teknis

1. **Nested BOM (resep bertingkat)** — bagian paling rumit. Wajib: penelusuran rekursif, proteksi loop, dan unit test dengan contoh nyata (Mawar & Bouquet S). Salah di sini = stok kacau.
2. **Konsistensi stok** — gunakan pola **stock ledger** + transaksi DB atomic, bukan menimpa satu angka stok. Memudahkan audit & koreksi.
3. **Perubahan resep/harga** — putuskan apakah penjualan lama ikut berubah (idealnya tidak: simpan snapshot qty saat transaksi terjadi).
4. **Stok minus** — tentukan bersama client: diblokir atau diizinkan dengan peringatan.
5. **Migrasi dari spreadsheet** — siapkan importer bila client sudah punya data di Sheets.

---

## 7. Definition of Done (per fitur)

Sebuah fitur dianggap selesai bila: fungsi berjalan sesuai user story, ada validasi input, tertangani error dasar, ada minimal 1 test untuk logika kritis (khususnya auto-deduct), UI dalam Bahasa Indonesia & mudah dipakai non-teknis, serta sudah dicoba di alur end-to-end.

---

## 8. Langkah Lanjutan

Setelah breakdown ini disetujui, dokumen pendukung berikut sebaiknya dibuat: **(a)** Rekomendasi Tech Stack & diagram arsitektur, **(b)** Skema Database detail + logika query nested BOM, **(c)** Wireframe layar utama (input penjualan, builder resep, dashboard). Ketiganya membuat developer bisa langsung mulai M0.
