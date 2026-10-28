# Dokumen Kebutuhan Produk (PRD): Sistem Penerimaan Barang Gudang

| | |
|---|---|
| **Versi** | 1.0 |
| **Status** | Disetujui untuk diimplementasi |
| **Pembaca** | Developer dan AI coding agent (Antigravity). Kebutuhan ditulis agar bisa diimplementasi tanpa menebak perilaku bisnis. |
| **Dokumen terkait** | `technical-spec.md`, `erd.mmd`, `openapi.yaml`, `data-flow.mmd`, `testing.md`, `ai-workflow.md` |

> Ini adalah terjemahan Indonesia dari `prd.md` (versi Inggris). Jika ada perbedaan, **versi Inggris (`prd.md`) menjadi acuan utama**. Istilah teknis, kode error, nama status, dan ID kebutuhan sengaja dibiarkan dalam bahasa Inggris agar sama persis dengan kode.

**Konvensi.** Kata MUST (WAJIB), MUST NOT (DILARANG), SHOULD (SEBAIKNYA), dan MAY (BOLEH) mengikuti RFC 2119. ID kebutuhan bersifat tetap dan dirujuk oleh spesifikasi teknis dan test: `FR-*` kebutuhan fungsional, `NFR-*` non-fungsional, `BR-*` aturan bisnis, `US-*` user story, `A-*` asumsi, `Q-*` pertanyaan terbuka.

---

## 1. Pernyataan Masalah dan Tujuan Bisnis

### 1.1 Masalah

Gudang mencatat barang masuk menggunakan spreadsheet. Akibatnya:

- Penerimaan berlebih tidak terdeteksi, karena tidak ada yang menghalangi angka yang diketik melebihi pesanan.
- Tidak ada catatan yang bisa dipercaya tentang siapa menerima apa dan kapan.
- Angka stok menyimpang dari kenyataan karena beberapa orang mengedit file yang sama.
- Tidak ada cara cepat untuk mengetahui pengiriman mana yang masih terbuka, sebagian, atau selesai tanpa membaca setiap baris.

### 1.2 Tujuan

| ID | Tujuan | Ukuran keberhasilan |
|---|---|---|
| O-1 | Membuat penerimaan melebihi jumlah yang diharapkan menjadi mustahil. | 0 penerimaan yang melebihi sisa jumlah, termasuk saat request bersamaan atau duplikat. |
| O-2 | Menjaga jumlah stok selalu benar setelah penerimaan. | Untuk setiap produk, `on_hand` sama dengan jumlah seluruh kuantitas transaksinya. |
| O-3 | Menyediakan jejak audit yang bisa dipercaya. | Setiap perubahan stok punya baris transaksi dengan user, waktu, dan dokumen sumber. |
| O-4 | Menampilkan progres pengiriman sekilas pandang. | Setiap dokumen menampilkan status dan jumlah expected/received/remaining per item. |
| O-5 | Cukup sederhana untuk dipakai operator saat pengiriman berlangsung. | Mencatat penerimaan cukup satu layar dan tidak lebih dari 3 interaksi per item. |

---

## 2. Pengguna dan Peran

| Peran | Deskripsi | Tanggung jawab |
|---|---|---|
| **OPERATOR** | Staf lapangan gudang yang membongkar dan menghitung kiriman. | Membuat draft dokumen penerimaan, mengedit draft, mencatat jumlah yang diterima, melihat produk, dokumen, inventory, dan transaksi. |
| **SUPERVISOR** | Kepala gudang yang memegang data master dan memberi persetujuan. | Semua yang bisa dilakukan operator, ditambah: membuat produk, merilis (open) dokumen, membatalkan dokumen. |

### 2.1 Matriks Hak Akses

| Aksi | OPERATOR | SUPERVISOR |
|---|:---:|:---:|
| Login / melihat profil sendiri | Ya | Ya |
| Melihat daftar dan detail produk | Ya | Ya |
| Membuat produk | Tidak | Ya |
| Melihat daftar dan detail dokumen penerimaan | Ya | Ya |
| Membuat dokumen penerimaan (`DRAFT`) | Ya | Ya |
| Mengedit dokumen `DRAFT` (header dan item) | Ya | Ya |
| Membuka (rilis) dokumen: `DRAFT` ke `OPEN` | Tidak | Ya |
| Membatalkan dokumen | Tidak | Ya |
| Mencatat penerimaan | Ya | Ya |
| Melihat inventory | Ya | Ya |
| Melihat transaksi inventory | Ya | Ya |

Backend WAJIB menegakkan matriks ini pada setiap request. Frontend hanya menyembunyikan aksi yang dilarang sebagai kemudahan, bukan sebagai pengaman.

---

## 3. Ruang Lingkup

### 3.1 Dalam Lingkup

1. Autentikasi dengan dua peran (JWT).
2. Manajemen produk: daftar, pencarian, pembuatan. Kode SKU.
3. Dokumen penerimaan: buat, daftar, filter berdasarkan status, lihat detail beserta item, edit selama `DRAFT`.
4. Siklus hidup dokumen: `DRAFT`, `OPEN`, `PARTIALLY_RECEIVED`, `COMPLETED`, `CANCELLED`.
5. Pencatatan penerimaan sebagian dan penuh terhadap item dokumen.
6. Pembaruan inventory otomatis dan riwayat transaksi yang hanya bisa ditambah (append-only).
7. Daftar inventory dan daftar riwayat transaksi dengan filter dan paginasi.
8. Penegakan semua aturan bisnis di backend, ditambah constraint database sebagai lapisan kedua.
9. Operasi penerimaan yang idempotent.
10. Endpoint health dan readiness, deployment Docker, migrasi, seed user.

### 3.2 Di Luar Lingkup

| Item | Alasan |
|---|---|
| Mengedit, membalik, atau menghapus penerimaan (koreksi) | Menjaga ledger tetap append-only. Dicatat sebagai keterbatasan yang diketahui. |
| Menutup dokumen kiriman kurang sebelum diterima penuh | Lihat Q-1. |
| Banyak gudang, lokasi, bin | Diasumsikan satu gudang (A-13). |
| Nomor batch/lot, tanggal kedaluwarsa, nomor seri | Tidak diminta skenario. |
| Data master supplier dan purchase order | Supplier hanyalah isian teks bebas pada dokumen. |
| Pengiriman keluar, transfer, penyesuaian stok, retur | Hanya penerimaan. |
| Mengedit atau menonaktifkan produk setelah dibuat | Tidak diperlukan. SKU memang harus tetap tidak berubah. |
| UI manajemen user, registrasi, reset password | User dibuat lewat skrip seed. |
| Pemindaian barcode, pencetakan, notifikasi email | Tidak diperlukan. |
| Laporan dan analitik, impor/ekspor CSV | Tidak diperlukan. |
| UI multi-bahasa | UI hanya berbahasa Inggris. |

---

## 4. Glosarium

| Istilah | Arti |
|---|---|
| **SKU** | Stock Keeping Unit. Kode unik yang mengidentifikasi produk. |
| **Dokumen penerimaan (receiving document)** | Dokumen inbound yang memuat produk dan jumlah yang diharapkan untuk satu pengiriman. |
| **Item dokumen (line)** | Satu produk dengan jumlah yang diharapkan di dalam dokumen. |
| **Receipt** | Satu kejadian pencatatan: sekumpulan jumlah yang diterima untuk item dari satu dokumen, dikirim dalam satu request. |
| **Remaining quantity (sisa)** | `expected_quantity - received_quantity` untuk sebuah item dokumen. |
| **On hand** | Stok tersedia saat ini untuk sebuah produk. |
| **Inventory transaction** | Satu baris ledger yang tidak dapat diubah, menggambarkan perubahan stok. |
| **Idempotency key** | Nilai unik yang dibuat client agar request penerimaan yang diulang tetap aman. |

---

## 5. Kebutuhan Fungsional

### 5.1 Autentikasi dan Otorisasi

| ID | Kebutuhan |
|---|---|
| FR-AUTH-1 | Sistem WAJIB mengautentikasi user dengan username dan password melalui `POST /api/v1/auth/login` dan mengembalikan JWT bertanda tangan beserta profil user (`id`, `username`, `fullName`, `role`). |
| FR-AUTH-2 | Semua endpoint kecuali login dan health/readiness WAJIB memerlukan header `Authorization: Bearer <token>` yang valid. |
| FR-AUTH-3 | Token WAJIB kedaluwarsa setelah durasi yang dapat dikonfigurasi (default 8 jam). Token kedaluwarsa atau rusak mengembalikan `401`. |
| FR-AUTH-4 | Password WAJIB disimpan hanya sebagai hash bcrypt. |
| FR-AUTH-5 | Role WAJIB dibaca dari token yang telah diverifikasi dan diperiksa ulang terhadap user di database (user harus ada dan aktif). |
| FR-AUTH-6 | User yang memanggil aksi yang tidak diizinkan untuk perannya WAJIB menerima `403`. |
| FR-AUTH-7 | Dua user WAJIB dibuat oleh skrip seed: satu `OPERATOR` dan satu `SUPERVISOR`. Username dan password berasal dari environment variable. Tidak ada kredensial yang di-commit. |
| FR-AUTH-8 | `GET /api/v1/auth/me` mengembalikan user saat ini. |

### 5.2 Produk

| ID | Kebutuhan |
|---|---|
| FR-PRD-1 | Sistem WAJIB menampilkan daftar produk dengan paginasi dan kata pencarian yang mencocokkan SKU atau nama (tidak peka huruf besar/kecil, sebagian kata). |
| FR-PRD-2 | Supervisor WAJIB dapat membuat produk dengan: `sku` (wajib), `name` (wajib), `unit` (opsional, default `PCS`), `description` (opsional). |
| FR-PRD-3 | SKU WAJIB unik (tidak peka huruf besar/kecil). Input di-trim dan disimpan dalam huruf kapital. |
| FR-PRD-4 | Format SKU: 2 sampai 32 karakter, `A-Z`, `0-9`, dan tanda hubung, diawali huruf atau angka. |
| FR-PRD-5 | SKU DILARANG diubah setelah dibuat. Tidak ada endpoint update produk. |
| FR-PRD-6 | Pembuatan produk WAJIB juga membuat baris saldo inventory dengan `on_hand = 0`, dalam transaksi yang sama. |
| FR-PRD-7 | Detail produk WAJIB dapat diambil berdasarkan id. |

### 5.3 Dokumen Penerimaan

| ID | Kebutuhan |
|---|---|
| FR-DOC-1 | Operator dan supervisor WAJIB dapat membuat dokumen penerimaan dengan: `supplierName` (opsional, maks 120), `expectedDate` (opsional, tanggal), `notes` (opsional, maks 500), dan 1 sampai 100 item, masing-masing `{productId, expectedQuantity}`. |
| FR-DOC-2 | Sistem WAJIB membuat nomor dokumen unik yang mudah dibaca: `RCV-` diikuti sequence 6 digit berawalan nol (contoh `RCV-000042`). Celah nomor diperbolehkan. Client tidak dapat menentukannya. |
| FR-DOC-3 | Dokumen baru WAJIB berstatus `DRAFT`. |
| FR-DOC-4 | Daftar dokumen WAJIB mendukung paginasi, filter berdasarkan status, dan pencarian berdasarkan nomor dokumen atau nama supplier. Urutan default: terbaru dahulu. |
| FR-DOC-5 | Detail dokumen WAJIB mengembalikan field header, status, pembuat, timestamp, dan semua item dengan `expectedQuantity`, `receivedQuantity`, `remainingQuantity`. WAJIB juga mengembalikan riwayat receipt (lihat FR-REC-8). |
| FR-DOC-6 | Selama `DRAFT`, field header dan seluruh daftar item BOLEH diganti melalui `PUT`. Pada status lain, pengeditan ditolak. |
| FR-DOC-7 | Supervisor WAJIB dapat membuka dokumen `DRAFT` (`DRAFT` ke `OPEN`). Dokumen WAJIB memiliki minimal satu item. |
| FR-DOC-8 | Supervisor WAJIB dapat membatalkan dokumen berstatus `DRAFT`, atau `OPEN` tanpa receipt. Alasan pembatalan (1 sampai 255 karakter) wajib diisi. |
| FR-DOC-9 | Tidak ada endpoint untuk menyelesaikan dokumen secara manual. `COMPLETED` hanya ditetapkan oleh sistem (BR-4). |
| FR-DOC-10 | Dokumen DILARANG dihapus. |

### 5.4 Penerimaan (Mencatat Receipt)

| ID | Kebutuhan |
|---|---|
| FR-REC-1 | Operator dan supervisor WAJIB dapat mencatat receipt pada dokumen berstatus `OPEN` atau `PARTIALLY_RECEIVED` melalui `POST /api/v1/receiving-documents/{id}/receipts`. |
| FR-REC-2 | Request receipt berisi `lines` (1 sampai 100): masing-masing `{itemId, quantity}`, plus `note` opsional (maks 255). |
| FR-REC-3 | `quantity` WAJIB berupa bilangan bulat dari 1 sampai sisa kuantitas item (BR-1). |
| FR-REC-4 | Setiap `itemId` WAJIB milik dokumen tersebut dan hanya boleh muncul satu kali per request. |
| FR-REC-5 | Request WAJIB menyertakan header `Idempotency-Key` (UUID, 8 sampai 64 karakter). Request tanpa header ini ditolak dengan `400`. Lihat BR-7. |
| FR-REC-6 | Operasi WAJIB atomik: jika satu line saja tidak valid, tidak ada line yang diterapkan dan stok tidak berubah. |
| FR-REC-7 | Saat berhasil, dalam satu transaksi database yang sama, sistem WAJIB: (a) membuat receipt beserta line-nya, (b) menambah `received_quantity` setiap item, (c) menambah `on_hand` setiap produk, (d) menambahkan satu inventory transaction per line, (e) menghitung ulang dan memperbarui status dokumen. |
| FR-REC-8 | Response WAJIB menyertakan receipt, item yang diperbarui, status dokumen terbaru, dan `on_hand` terbaru untuk setiap produk yang terdampak. Detail dokumen WAJIB menampilkan receipt sebelumnya (nomor receipt, user, waktu, line). |

### 5.5 Inventory dan Transaksi

| ID | Kebutuhan |
|---|---|
| FR-INV-1 | Sistem WAJIB menampilkan inventory: satu baris per produk dengan `sku`, `name`, `unit`, `onHand`, `updatedAt`. Produk dengan stok nol WAJIB disertakan. Mendukung pencarian dan paginasi. |
| FR-INV-2 | Sistem WAJIB menampilkan transaksi inventory, terbaru dahulu, dengan paginasi. |
| FR-INV-3 | Setiap transaksi WAJIB menampilkan: id, timestamp, produk (SKU dan nama), tipe, perubahan kuantitas, saldo setelahnya, nomor dokumen sumber, id receipt, user pelaksana. |
| FR-INV-4 | Transaksi WAJIB dapat difilter berdasarkan produk, dokumen, dan rentang tanggal. |
| FR-INV-5 | Transaksi bersifat append-only. Tidak ada API atau jalur kode yang mengubah atau menghapusnya. |
| FR-INV-6 | Satu-satunya tipe transaksi pada rilis ini adalah `RECEIPT`. Tipe dimodelkan sebagai enum agar tipe lain dapat ditambahkan nanti. |

### 5.6 Sistem

| ID | Kebutuhan |
|---|---|
| FR-SYS-1 | `GET /health` mengembalikan `200` saat proses hidup (tanpa pemeriksaan database). |
| FR-SYS-2 | `GET /ready` mengembalikan `200` hanya saat database dapat dijangkau dan migrasi sudah diterapkan; jika tidak, `503`. |
| FR-SYS-3 | Migrasi WAJIB dijalankan melalui satu perintah yang terdokumentasi dan aman dijalankan berulang. |
| FR-SYS-4 | Perintah seed WAJIB membuat dua user dari environment variable, dan BOLEH membuat contoh produk bila diaktifkan secara eksplisit. |

### 5.7 Frontend

| ID | Kebutuhan |
|---|---|
| FR-UI-1 | Halaman login. Setelah login, token disimpan selama sesi dan dilampirkan pada panggilan API. Token kedaluwarsa mengarahkan ke login. |
| FR-UI-2 | Halaman produk: tabel yang dapat dicari; form "New product" hanya terlihat oleh supervisor. |
| FR-UI-3 | Halaman dokumen penerimaan: tabel dengan badge status, filter status, pencarian, dan aksi "New document". |
| FR-UI-4 | Halaman buat/edit dokumen: supplier, tanggal diharapkan, catatan, tambah/hapus baris item dengan pemilih produk, validasi sisi client yang meniru aturan server. |
| FR-UI-5 | Halaman detail dokumen: header, badge status, tabel item dengan expected/received/remaining dan indikator progres, riwayat receipt, dan tombol aksi sesuai peran (Open, Cancel, Record receipt). |
| FR-UI-6 | Form receive: satu input angka per item dengan sisa kuantitas sebagai nilai maksimum. Saat submit, tombol dinonaktifkan selama request berjalan, dan idempotency key buatan client dipakai ulang saat mengulang submit yang sama. |
| FR-UI-7 | Halaman inventory dan transaksi dengan filter dan paginasi. |
| FR-UI-8 | Error dari server WAJIB ditampilkan dalam bentuk yang mudah dibaca menggunakan `message` error API dan error per field. UI DILARANG menganggap validasinya sendiri sudah cukup. |

---

## 6. Aturan Bisnis dan Validasi

### 6.1 Aturan Bisnis

| ID | Aturan |
|---|---|
| **BR-1** | **Tidak boleh menerima berlebih.** Untuk setiap item, `received_quantity` TIDAK BOLEH melebihi `expected_quantity`. Line receipt dengan `quantity > remaining` ditolak dan tidak ada yang diterapkan. |
| **BR-2** | **Hanya dokumen aktif yang dapat menerima.** Receipt diterima hanya jika status `OPEN` atau `PARTIALLY_RECEIVED`. |
| **BR-3** | **Kuantitas adalah bilangan bulat positif.** Kuantitas expected dan received berupa bilangan bulat minimal 1 (maksimum 1.000.000 per line). Nol, negatif, dan pecahan ditolak. |
| **BR-4** | **Penyelesaian otomatis dan ketat.** Setelah receipt diterapkan, jika setiap item memiliki `received_quantity = expected_quantity`, status menjadi `COMPLETED`. Jika tidak, status menjadi `PARTIALLY_RECEIVED`. Dokumen tidak boleh `COMPLETED` selama ada item yang masih memiliki sisa di atas nol. |
| **BR-5** | **Produk unik per dokumen.** Satu produk hanya boleh muncul satu kali dalam satu dokumen. |
| **BR-6** | **Tidak dapat diubah setelah dirilis.** Item dan jumlah expected hanya dapat diedit saat `DRAFT`. Setelah dirilis, nilainya tetap. |
| **BR-7** | **Penerimaan idempotent.** Request receipt diidentifikasi oleh `Idempotency-Key`-nya. Key sama dengan payload sama mengembalikan response asli (HTTP `200`, header `Idempotent-Replay: true`) dan tidak mengubah apa pun. Key sama dengan payload berbeda atau dokumen berbeda mengembalikan `409 IDEMPOTENCY_KEY_REUSED`. Key unik secara global. |
| **BR-8** | **Batas pembatalan.** Hanya dokumen `DRAFT` dan `OPEN` dengan nol receipt yang dapat dibatalkan. Dokumen `PARTIALLY_RECEIVED`, `COMPLETED`, dan `CANCELLED` tidak dapat dibatalkan. |
| **BR-9** | **Status akhir.** `COMPLETED` dan `CANCELLED` tidak mengizinkan perubahan lebih lanjut. |
| **BR-10** | **Integritas stok.** Untuk setiap produk, `on_hand` sama dengan jumlah kuantitas transaksinya dan tidak pernah negatif. `balance_after` setiap transaksi sama dengan saldo sebelumnya ditambah kuantitasnya. |
| **BR-11** | **Keamanan konkurensi.** Dua receipt bersamaan untuk item yang sama tidak boleh melebihi jumlah expected. Satu berhasil, dan yang lain menerima error aturan bisnis bila total gabungan akan melebihi. |
| **BR-12** | **Keunikan SKU.** SKU unik tanpa peka huruf besar/kecil dan tidak dapat diubah. |
| **BR-13** | **Open butuh item.** Dokumen tanpa item tidak dapat dibuka. |
| **BR-14** | **Atomisitas.** Receipt bersifat semua-atau-tidak-sama-sekali di seluruh line-nya. |

### 6.2 Transisi Status yang Valid

| Dari | Ke | Pemicu | Pelaku | Kondisi |
|---|---|---|---|---|
| `DRAFT` | `OPEN` | Aksi Open | Supervisor | Minimal 1 item (BR-13) |
| `DRAFT` | `CANCELLED` | Aksi Cancel | Supervisor | Alasan diisi |
| `OPEN` | `PARTIALLY_RECEIVED` | Receipt diterapkan | Sistem | Masih ada item dengan sisa |
| `OPEN` | `COMPLETED` | Receipt diterapkan | Sistem | Semua item diterima penuh |
| `OPEN` | `CANCELLED` | Aksi Cancel | Supervisor | Nol receipt, alasan diisi |
| `PARTIALLY_RECEIVED` | `PARTIALLY_RECEIVED` | Receipt diterapkan | Sistem | Masih ada item dengan sisa |
| `PARTIALLY_RECEIVED` | `COMPLETED` | Receipt diterapkan | Sistem | Semua item diterima penuh |

Transisi lain tidak valid dan mengembalikan `409 INVALID_STATE_TRANSITION` (atau `409 DOCUMENT_NOT_RECEIVABLE` untuk receipt).

### 6.3 Ringkasan Validasi Field

| Field | Aturan |
|---|---|
| `username` | Wajib, 3 sampai 50 karakter. |
| `password` | Wajib, 8 sampai 128 karakter pada input login. |
| `sku` | Wajib, di-trim, dijadikan huruf kapital, regex `^[A-Z0-9][A-Z0-9-]{1,31}$`. |
| `product.name` | Wajib, di-trim, 1 sampai 120 karakter. |
| `product.unit` | Opsional, 1 sampai 16 karakter, default `PCS`. |
| `product.description` | Opsional, maks 500 karakter. |
| `supplierName` | Opsional, maks 120 karakter. |
| `expectedDate` | Opsional, tanggal ISO `YYYY-MM-DD`. Tanggal lampau diperbolehkan. |
| `notes` | Opsional, maks 500 karakter. |
| `items` | 1 sampai 100 entri, `productId` unik. |
| `expectedQuantity` | Bilangan bulat, 1 sampai 1.000.000. |
| `quantity` (line receipt) | Bilangan bulat, 1 sampai sisa kuantitas. |
| `Idempotency-Key` | Wajib pada receive. 8 sampai 64 karakter, `A-Za-z0-9-_`. |
| Alasan cancel | Wajib, 1 sampai 255 karakter. |
| Paginasi | `page` >= 1 (default 1), `pageSize` 1 sampai 100 (default 20). |

### 6.4 Kebutuhan Penanganan Error

Semua error memakai satu bentuk JSON:

```json
{
  "error": {
    "code": "OVER_RECEIPT",
    "message": "Quantity 12 exceeds remaining quantity 10 for item 7.",
    "details": [{ "field": "lines[0].quantity", "issue": "exceeds_remaining", "remaining": 10 }],
    "requestId": "b1f1c7a2-..."
  }
}
```

| HTTP | Arti | Contoh kode |
|---|---|---|
| 400 | Input salah format atau tidak sesuai skema | `VALIDATION_ERROR`, `IDEMPOTENCY_KEY_REQUIRED` |
| 401 | Token hilang, kedaluwarsa, atau tidak valid / kredensial salah | `UNAUTHENTICATED`, `INVALID_CREDENTIALS` |
| 403 | Role tidak diizinkan | `FORBIDDEN` |
| 404 | Resource tidak ditemukan | `NOT_FOUND` |
| 409 | Konflik status atau keunikan | `SKU_ALREADY_EXISTS`, `INVALID_STATE_TRANSITION`, `DOCUMENT_NOT_RECEIVABLE`, `DOCUMENT_NOT_EDITABLE`, `IDEMPOTENCY_KEY_REUSED` |
| 422 | Aturan bisnis dilanggar | `OVER_RECEIPT`, `DUPLICATE_PRODUCT_IN_DOCUMENT`, `DOCUMENT_HAS_NO_ITEMS`, `DUPLICATE_ITEM_IN_RECEIPT` |
| 500 | Error tak terduga (tanpa membocorkan detail internal) | `INTERNAL_ERROR` |
| 503 | Belum siap | `NOT_READY` |

Spesifikasi teknis menjadi acuan untuk daftar kode lengkap.

---

## 7. User Story dan Kriteria Penerimaan

### US-01: Login

*Sebagai user gudang, saya ingin login agar dapat memakai sistem dengan hak akses sesuai peran saya.*

- **Given** kredensial valid, **when** saya login, **then** saya menerima token dan peran saya, lalu masuk ke halaman dokumen.
- **Given** kredensial salah, **when** saya login, **then** saya melihat "Invalid username or password" dan API mengembalikan `401 INVALID_CREDENTIALS` tanpa mengungkap bagian mana yang salah.
- **Given** token kedaluwarsa, **when** saya memanggil API apa pun, **then** saya mendapat `401` dan UI mengarahkan saya ke login.

### US-02: Mengelola produk

*Sebagai supervisor, saya ingin mendaftarkan produk dengan SKU agar dapat diterima.*

- **Given** saya supervisor, **when** saya mengirim SKU `abc-100` dan nama "Widget", **then** produk tersimpan dengan SKU `ABC-100`, dan baris inventory dengan `on_hand = 0` terbentuk.
- **Given** SKU `ABC-100` sudah ada, **when** saya membuat `abc-100`, **then** saya mendapat `409 SKU_ALREADY_EXISTS`.
- **Given** SKU `bad sku!`, **when** saya mengirim, **then** saya mendapat `400 VALIDATION_ERROR` yang menyebut field `sku`.
- **Given** saya operator, **when** saya memanggil create product, **then** saya mendapat `403`, dan UI tidak menampilkan form.
- **Given** 30 produk, **when** saya mencari "widget", **then** hanya baris dengan SKU atau nama yang cocok yang dikembalikan, dengan paginasi.

### US-03: Membuat dokumen penerimaan

*Sebagai operator, saya ingin mendaftarkan kiriman yang diharapkan beserta produk dan jumlahnya.*

- **Given** item valid, **when** saya membuat dokumen, **then** dokumen mendapat nomor seperti `RCV-000001`, status `DRAFT`, dan `receivedQuantity = 0` pada semua item.
- **Given** produk yang sama dua kali, **when** saya mengirim, **then** saya mendapat `422 DUPLICATE_PRODUCT_IN_DOCUMENT`, dan tidak ada yang tersimpan.
- **Given** `expectedQuantity` 0, -3, atau 2.5, **when** saya mengirim, **then** saya mendapat `400 VALIDATION_ERROR`.
- **Given** `productId` yang tidak ada, **when** saya mengirim, **then** saya mendapat `404 NOT_FOUND` (atau `422`, sesuai spesifikasi) dan tidak ada yang tersimpan.
- **Given** daftar item kosong, **when** saya mengirim, **then** saya mendapat `400 VALIDATION_ERROR`.

### US-04: Mengedit draft

- **Given** dokumen `DRAFT`, **when** saya mengganti item dan header-nya, **then** perubahan tersimpan.
- **Given** dokumen berstatus lain, **when** saya mencoba mengeditnya, **then** saya mendapat `409 DOCUMENT_NOT_EDITABLE`.

### US-05: Merilis dokumen

*Sebagai supervisor, saya ingin membuka draft agar operator dapat mulai menerima barang.*

- **Given** `DRAFT` yang memiliki item, **when** saya membukanya, **then** statusnya menjadi `OPEN`.
- **Given** saya operator, **when** saya membukanya, **then** saya mendapat `403`.
- **Given** dokumen yang bukan `DRAFT`, **when** saya membukanya, **then** saya mendapat `409 INVALID_STATE_TRANSITION`.

### US-06: Membatalkan dokumen

- **Given** dokumen `DRAFT` atau `OPEN` tanpa receipt, **when** supervisor membatalkan dengan alasan, **then** statusnya menjadi `CANCELLED` dan alasannya tersimpan.
- **Given** dokumen `PARTIALLY_RECEIVED` atau `COMPLETED`, **when** saya membatalkan, **then** saya mendapat `409 INVALID_STATE_TRANSITION`.
- **Given** tanpa alasan, **when** saya membatalkan, **then** saya mendapat `400 VALIDATION_ERROR`.

### US-07: Mencatat penerimaan sebagian

*Sebagai operator, saya ingin mencatat jumlah yang benar-benar datang.*

- **Given** dokumen `OPEN` dengan item A expected 100, **when** saya menerima 40 dari A, **then** item A menunjukkan received 40 dan remaining 60, status menjadi `PARTIALLY_RECEIVED`, `on_hand` produk A bertambah 40, dan satu transaksi `RECEIPT` sebesar +40 terbentuk dengan `balance_after` yang benar.
- **Given** kondisi tersebut, **when** saya menerima 60 lagi dari A (dan A adalah satu-satunya item), **then** status menjadi `COMPLETED`, dan `on_hand` telah bertambah total 100.

### US-08: Mencegah penerimaan berlebih

- **Given** item A dengan remaining 60, **when** saya menerima 61, **then** saya mendapat `422 OVER_RECEIPT`, dan received quantity, `on_hand`, transaksi, serta status tidak berubah.
- **Given** receipt dengan dua line di mana line 1 valid dan line 2 melebihi sisa, **when** saya mengirim, **then** tidak ada line yang diterapkan (atomisitas).
- **Given** kuantitas 0, -5, atau 1.5, **when** saya mengirim, **then** saya mendapat `400 VALIDATION_ERROR`.

### US-09: Aturan penyelesaian

- **Given** dokumen dengan item A (expected 10) dan B (expected 5), **when** hanya A yang diterima penuh, **then** statusnya `PARTIALLY_RECEIVED`, bukan `COMPLETED`.
- **Given** dokumen yang sama, **when** B kemudian diterima penuh, **then** status otomatis menjadi `COMPLETED`.
- **Given** dokumen `COMPLETED`, `CANCELLED`, atau `DRAFT`, **when** saya mencatat receipt, **then** saya mendapat `409 DOCUMENT_NOT_RECEIVABLE` dan tidak ada stok yang berubah.
- **Given** panggilan API apa pun, **when** saya mencoba mengatur status secara langsung, **then** tidak ada endpoint seperti itu.

### US-10: Pengulangan aman dan submit ganda

- **Given** receipt berhasil dengan key K, **when** request yang sama dengan key K dikirim lagi (klik dua kali, retry jaringan), **then** saya mendapat hasil asli dengan `Idempotent-Replay: true`, dan stok tidak berubah karena pengulangan itu.
- **Given** key K pernah dipakai dengan line berbeda, **when** saya mengirimnya lagi, **then** saya mendapat `409 IDEMPOTENCY_KEY_REUSED`.
- **Given** tanpa `Idempotency-Key`, **when** saya mengirim, **then** saya mendapat `400 IDEMPOTENCY_KEY_REQUIRED`.
- **Given** dua request bersamaan dengan key berbeda yang jika digabung melebihi sisa, **when** keduanya diproses, **then** tepat satu berhasil, yang lain mendapat `422 OVER_RECEIPT`, dan received quantity akhir tidak melebihi expected.
- **Given** dua request bersamaan dengan key yang sama, **when** keduanya diproses, **then** hanya satu receipt yang terbentuk.

### US-11: Melihat detail dokumen

- **Given** sebuah dokumen, **when** saya membuka halaman detailnya, **then** saya melihat header, status, item dengan expected/received/remaining, dan riwayat receipt beserta user dan waktu.

### US-12: Melihat dan memfilter daftar dokumen

- **Given** dokumen dengan berbagai status, **when** saya memfilter `PARTIALLY_RECEIVED`, **then** hanya dokumen tersebut yang tampil. Pencarian berdasarkan nomor atau supplier berfungsi. Hasil berpaginasi, terbaru dahulu.

### US-13: Melihat inventory

- **Given** produk dengan receipt, **when** saya membuka inventory, **then** setiap produk menampilkan `on_hand` saat ini, termasuk produk dengan stok 0.
- **Given** receipt pada US-07, **when** saya membuka inventory, **then** `on_hand` sama dengan jumlah kuantitas yang diterima untuk produk tersebut di semua dokumen.

### US-14: Melihat riwayat transaksi

- **Given** receipt telah dicatat, **when** saya membuka transaksi, **then** saya melihat setiap perubahan stok dengan produk, +kuantitas, saldo setelahnya, nomor dokumen, user, dan waktu, terbaru dahulu.
- **Given** filter produk, filter dokumen, atau rentang tanggal, **when** saya menerapkannya, **then** hanya baris yang cocok yang tampil.
- **Given** peran apa pun, **when** saya mencari cara mengedit atau menghapus transaksi, **then** tidak ada caranya.

### US-15: Penegakan peran

- **Given** token operator, **when** memanggil endpoint create product, open, atau cancel secara langsung (melewati UI), **then** masing-masing mengembalikan `403`.
- **Given** tanpa token, **when** memanggil endpoint terproteksi mana pun, **then** mengembalikan `401`.

### US-16: Operasional

- **Given** mesin bersih yang memiliki Docker, **when** saya mengikuti README, **then** stack berjalan, migrasi diterapkan, seed user ada, `/health` dan `/ready` mengembalikan `200`, dan saya dapat login.

---

## 8. Kebutuhan Non-Fungsional

| ID | Kategori | Kebutuhan |
|---|---|---|
| NFR-1 | Integritas data | Semua penulisan multi-baris (receipt, buat/edit dokumen, buat produk) WAJIB berjalan dalam satu transaksi database. |
| NFR-2 | Integritas data | Database WAJIB menegakkan: `expected_quantity > 0`, `received_quantity >= 0`, `received_quantity <= expected_quantity`, `on_hand >= 0`, SKU unik (tidak peka huruf besar/kecil), `(document_id, product_id)` unik, idempotency key unik, dan foreign key valid. |
| NFR-3 | Konkurensi | Penerimaan WAJIB mengunci baris dokumen dan item terkait (`SELECT ... FOR UPDATE`) dengan urutan yang konsisten untuk mencegah lost update dan deadlock. |
| NFR-4 | Keamanan | Password di-hash dengan bcrypt (cost 10 atau lebih). Secret JWT dari environment, minimal 32 karakter. Input divalidasi dengan skema. Query diparameterisasi (ORM). Header keamanan lewat `helmet`. CORS dibatasi pada origin yang dikonfigurasi. Login dibatasi lajunya (rate limit). |
| NFR-5 | Keamanan | Tidak ada secret di repository. `.env.example` hanya berisi daftar variabel dengan nilai placeholder. Response error tidak pernah membocorkan stack trace atau SQL. |
| NFR-6 | Performa | Dengan 10.000 dokumen dan 100.000 transaksi, endpoint daftar merespons di bawah 300 ms pada p95 di laptop developer. Receipt 20 line selesai di bawah 500 ms. Daftar WAJIB berpaginasi dan didukung index. |
| NFR-7 | Keandalan | API hanya mulai setelah migrasi berhasil. Shutdown yang halus (graceful) menutup koneksi database. |
| NFR-8 | Observabilitas | Log request terstruktur dengan request ID yang dikembalikan di `X-Request-Id` dan di body error. Receipt dicatat di log dengan id dokumen dan id user. |
| NFR-9 | Portabilitas | Seluruh stack berjalan dengan `docker compose up`. Semua konfigurasi lewat environment variable. |
| NFR-10 | Keterawatan | TypeScript mode strict. Lapisan dipisah: routes, services (aturan bisnis), repositories/Prisma. Aturan bisnis hanya berada di services dan tidak diduplikasi secara tidak konsisten di handler. |
| NFR-11 | Kemudahan diuji | Aturan bisnis dicakup test otomatis yang berjalan terhadap instance PostgreSQL sungguhan (bukan mock) untuk alur penerimaan. |
| NFR-12 | Waktu | Semua timestamp disimpan dalam UTC (`timestamptz`) dan dikembalikan dalam ISO 8601. |
| NFR-13 | Kegunaan | Tata letak responsif, dapat dipakai di tablet dan desktop. State loading, kosong, dan error di setiap halaman. Form dapat dioperasikan dengan keyboard dan berlabel. |
| NFR-14 | API | REST, JSON, diberi versi di bawah `/api/v1`, didokumentasikan di `openapi.yaml`, yang WAJIB sesuai dengan implementasi. |
| NFR-15 | Kompatibilitas | Browser evergreen (Chrome, Edge, Firefox, Safari versi terbaru). |

---

## 9. Asumsi, Batasan, dan Pertanyaan Terbuka

### 9.1 Asumsi

| ID | Asumsi | Alasan / dampak |
|---|---|---|
| A-1 | Himpunan status adalah `DRAFT`, `OPEN`, `PARTIALLY_RECEIVED`, `COMPLETED`, `CANCELLED`. | Soal menyebut pelacakan status tanpa menamainya. Tahap draft mendukung peninjauan sebelum pekerjaan dimulai. |
| A-2 | Operator dan supervisor sama-sama dapat membuat dokumen. | Staf lapangan sering paling dulu tahu apa yang akan datang. |
| A-3 | Hanya supervisor yang membuka (merilis) dokumen. | Memberi langkah persetujuan ringan dan pemisahan tugas. |
| A-4 | Kedua peran dapat mencatat receipt, hanya pada dokumen `OPEN` atau `PARTIALLY_RECEIVED`. | Operator melakukan penghitungan fisik. |
| A-5 | Penyelesaian otomatis, tanpa tombol "complete" manual. | Langsung memenuhi aturan "tidak boleh complete sebelum semua item diterima penuh". |
| A-6 | Hanya supervisor yang membatalkan, dan hanya bila belum ada yang diterima. | Pembatalan setelah penerimaan memerlukan pembalikan stok, yang di luar lingkup. |
| A-7 | Satu produk muncul sekali per dokumen. | Mencegah ambiguitas penerimaan sebagian pada line yang duplikat. |
| A-8 | Receipt tidak dapat diedit atau dibalik. | Menjaga ledger append-only dan mudah diaudit. Keterbatasan yang diketahui. |
| A-9 | Hanya supervisor yang membuat produk. SKU tidak dapat diubah dan unik. | SKU adalah data master yang dirujuk riwayat. |
| A-10 | Idempotency key wajib pada endpoint receive, disertai row locking. | Memenuhi kebutuhan bahwa operasi duplikat tidak merusak stok. |
| A-11 | CHECK constraint database menduplikasi aturan utama. | Pertahanan berlapis bila kode aplikasi punya bug. |
| A-12 | `inventory` menyimpan saldo saat ini. `inventory_transactions` adalah ledger append-only. Keduanya diperbarui dalam satu transaksi. | Pembacaan cepat plus jejak audit lengkap. BR-10 dapat diuji. |
| A-13 | Satu gudang, satu kolam stok per produk. | Lokasi tidak disebut dalam soal. |
| A-14 | Kuantitas berupa satuan utuh (bilangan bulat). | Menghindari penanganan desimal. Satuan pecahan di luar lingkup. |
| A-15 | Satu receipt boleh mencakup beberapa item dari satu dokumen dalam satu request. | Sesuai cara kiriman diperiksa dalam satu kali pengecekan. |
| A-16 | Menerima melebihi expected, sekecil apa pun, tidak pernah diizinkan. | Soal eksplisit. Tidak ada pengaturan toleransi. |
| A-17 | User dibuat lewat seed, bukan melalui UI. | Autentikasi bersifat "dasar" menurut soal. |
| A-18 | Supplier adalah teks bebas, bukan record master. | Menjaga lingkup tetap kecil. |

### 9.2 Batasan

- Database relasional: PostgreSQL.
- Backend: Node.js, Express, TypeScript, Prisma, Zod. Frontend: React, Vite, TypeScript.
- REST API terdokumentasi (OpenAPI 3).
- Docker dan Docker Compose untuk deployment. Konfigurasi hanya lewat environment variable.
- Dikembangkan dengan alur kerja AI agent berbasis spesifikasi (Antigravity). Dokumen di `/docs` adalah input utama agent.
- Target usaha 8 sampai 12 jam. Ketepatan alur inti lebih diutamakan daripada kerapian tampilan.
- Tidak ada fitur tambahan di luar lingkup di atas.

### 9.3 Pertanyaan Terbuka (belum terjawab, untuk ditanyakan ke pemangku kepentingan)

| ID | Pertanyaan | Penanganan saat ini |
|---|---|---|
| Q-1 | Apa yang terjadi bila supplier mengirim kurang dari yang diharapkan dan tidak akan mengirim sisanya? | Dokumen tetap `PARTIALLY_RECEIVED`. Aksi "close short" dengan persetujuan supervisor adalah jawaban masa depan yang paling mungkin. |
| Q-2 | Bagaimana kesalahan penerimaan dikoreksi (salah hitung, salah produk)? | Di luar lingkup. Diperlukan fitur penyesuaian stok atau pembalikan receipt di masa depan. |
| Q-3 | Apakah receipt harus dibatasi pada tanggal expected dokumen? | Tidak ada pembatasan. |
| Q-4 | Apakah produk memerlukan satuan selain item utuh, atau beberapa satuan ukur? | Satu label `unit` teks bebas, bersifat informatif saja. |
| Q-5 | Apakah lokasi/bin fisik diperlukan saat receipt? | Tidak dimodelkan. |
| Q-6 | Apakah single sign-on atau ekspor audit diperlukan? | Tidak diperlukan. |
| Q-7 | Apakah supervisor perlu melihat atau mencabut akses operator? | Tidak ada manajemen user dalam lingkup. |
| Q-8 | Berapa lama idempotency key harus disimpan? | Disimpan bersama receipt selama receipt itu ada. |

---

## 10. Definition of Done

Sebuah kebutuhan dianggap selesai bila:

1. Sudah diimplementasi di backend dan, jika berlaku, di UI.
2. Aturan bisnisnya ditegakkan di backend dan diverifikasi oleh test otomatis yang merujuk ID aturannya.
3. `openapi.yaml` sesuai dengan implementasi.
4. Berfungsi di bawah `docker compose up` dari lingkungan bersih.
5. Tercantum dengan benar di `testing.md`, dan setiap jalan pintas dicatat di bagian known limitations.
