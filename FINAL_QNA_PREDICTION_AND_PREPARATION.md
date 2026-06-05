# ReWear POS — Final Q&A Prediction and Preparation

## 1. Executive Summary

Project kedua adalah peningkatan besar dari baseline: GUI sudah berpindah dari CustomTkinter ke PyQt6, navigasi memakai `QStackedWidget`, view di-cache, category/condition punya CRUD master data, checkout punya quantity dan confirmation, seller historis tidak dihapus permanen, Telegram tidak lagi menyimpan token aktif di data repo, dan JSON write sudah lebih aman dengan temp file + `os.replace()`.

Status audit:
- Fakta kode: diverifikasi dari `app.py`, `views/`, `services/`, `utils/`, `data/`, `tests/`, `ReWearPOS.spec`, `FINAL_AUDIT_CHECKLIST.md`, `RELEASE_CANDIDATE_REPORT.md`, dan build output.
- Baseline file: `BASELINE_PROJECT_ANALYSIS.md` tidak ditemukan di repo terbaru. Perbandingan memakai audit baseline dari tahap sebelumnya dalam konteks kerja.
- Test yang dijalankan: `python -m compileall .`, `python tests\p0_smoke.py`, `python tests\p1_master_data_smoke.py`, `python tests\final_judge_feedback_smoke.py`.
- Hasil test: PASS semua.
- Catatan test: muncul warning Qt font directory, bukan test failure, tetapi perlu dicek visual di mesin demo.
- Manual QA: sebagian checklist sudah centang, tetapi banyak flow penting masih belum dicentang di `MANUAL_QA_RELEASE_CHECKLIST.md`, terutama master data, seller, checkout penuh, recovery dialog, Telegram live, dan distribusi lintas folder/perangkat.

Kesiapan final: kuat untuk demo kompetisi satu perangkat, tetapi belum boleh diklaim production-ready, multi-user, accounting lengkap, atau bebas bug.

## 2. Updated Project Audit

### Architecture

| Area | Temuan |
| --- | --- |
| GUI framework | PyQt6. Bukti: `app.py:12`, `requirements.txt`. |
| Entry point | `main.py` membuat `ReWearPOSApp()`; `app.py` juga punya `main()`. |
| Navigation | `QStackedWidget` untuk root/content stack, route map dinamis. Bukti: `app.py:46`, `app.py:123`, `app.py:159`, `app.py:282`. |
| View cache | `cached_views` per label route; refresh dipanggil saat view lama dibuka lagi. Bukti: `app.py:121`, `app.py:286`, `app.py:290`. |
| Role | Admin melihat Dashboard, Products, Sellers, Checkout, Transactions, Reports, Settings; cashier hanya Checkout. Bukti: `app.py:214`. |
| Storage | JSON lokal tetap dipakai, tetapi sudah memakai lock, sanitize, atomic temp write, `fsync`, dan `os.replace`. Bukti: `services/storage_service.py:81`, `services/storage_service.py:90`, `services/storage_service.py:91`. |
| Multi-file transaction write | Produk dan transaksi ditulis bersama lewat `write_many_json`. Bukti: `services/transaction_service.py:119`, `services/storage_service.py:99`. |
| Recovery JSON | Mode manual di app: backup corrupt file, user pilih recover atau exit. Bukti: `app.py:87`, `app.py:313`, `services/storage_service.py:159`, `services/storage_service.py:180`. |
| Logging | `logs/app.log` melalui `utils/app_logger.py`; recovery, write failure, checkout failure, integrity issue dilog. |
| Background task | `QThreadPool`/`QRunnable` untuk Telegram actions. Bukti: `views/common.py:614`, `views/common.py:635`. |
| Image cache | `QImageReader` + `IMAGE_CACHE` max 128. Bukti: `views/product_images.py`. |
| Build | `ReWearPOS.spec` memakai `SPECPATH`, memasukkan `data`, `assets`, `qtawesome`, hidden imports. Build output ada di `dist\ReWearPOS`. |

### Feature Inventory

| Area | Status | Audit |
| --- | --- | --- |
| Authentication | [!] | Login admin/cashier tersedia dan teruji, tetapi password masih plaintext di `data/users.json`. |
| Role navigation | [x] | Admin/cashier nav berbeda; test instantiate route PASS. |
| Dashboard | [~] | KPI, recent transaction, dead stock, Telegram buttons tersedia; visual/live Telegram perlu QA. |
| Product CRUD | [x] | Add/edit/delete dengan validation dan sold product guard. |
| Category CRUD | [x] | Dialog master data, active/inactive, duplicate guard, tested. |
| Condition CRUD | [x] | Sama seperti category, tested. |
| Upload image | [~] | File picker dan read-only path tersedia; manual picker/live file permission perlu QA. |
| Seller CRUD | [x] | Add/edit/delete safe; seller historis jadi inactive, tested. |
| Seller phone | [x] | UI regex digit dan backend digit-only. |
| Commission inheritance | [x] | Commission product consignment mengikuti default seller dan read-only, tested. |
| Date picker | [~] | `NoWheelDateEdit` dengan calendar popup; belum diverifikasi manual popup visual. |
| Checkout quantity | [x] | Plus/minus, stock limit, subtotal, backend overstock validation, tested. |
| Payment flow | [x] | Cash received, quick payment, validation, change, button state, confirmation. |
| Checkout confirmation | [x] | Confirm dialog sebelum save; cancel tidak mengubah data, tested. |
| Transactions | [x] | History, detail, search/date prefix, snapshot item. |
| Reports | [~] | Aggregate all-time; belum ada filter periode/export. |
| Telegram | [~] | Async send/detect/test tersedia; token kosong dan live delivery belum diuji. |
| Data integrity | [~] | Atomic write, backup, inactive safety tersedia; JSON tetap single-machine, bukan DB multi-user. |
| Build `.exe` | [~] | Build sukses dan dist ada; manual QA distribusi/perangkat lain masih perlu. |

### Business Logic

Store-owned product:
- Dibuat admin dari Product Management.
- Tidak punya seller; commission dipaksa 0.
- Store margin = selling price - cost price.
- Stock berkurang sesuai quantity checkout; status jadi sold jika stock 0.

Consignment product:
- Wajib memilih seller aktif.
- Seller ditampilkan sebagai `S001 - Dina Wardrobe`, backend menyimpan `seller_id`.
- Commission percent mengikuti default seller dan berarti seller share/payout liability.
- Store margin = selling price - seller share.
- Seller historis yang sudah punya produk/transaksi tidak hard-delete; ditandai inactive.

Category/condition:
- Master data tersimpan di `data/categories.json` dan `data/conditions.json`.
- Used category/condition tidak bisa di-rename untuk menjaga histori; delete berubah menjadi inactive.
- Inactive tidak tersedia untuk produk baru tetapi tetap bisa tampil pada produk historis.

Checkout:
- Cart menyimpan `{product, quantity}` di UI dan mengirim `{product_id, quantity}` ke service.
- UI membatasi quantity sampai available stock, backend memvalidasi ulang requested quantity.
- `checkout_in_progress` mencegah reentrant/double-click.
- Confirmation dialog menjadi titik terakhir sebelum transaksi dan stok disimpan.
- Transaksi menyimpan snapshot item per unit, bukan field quantity agregat.

Reports:
- Menampilkan total revenue, store margin, seller commission, category counts, seller commission, aging stock.
- Belum ada settlement accounting lengkap, payment method, refund, tax, atau export.

### Data Integrity

| Check | Status | Catatan |
| --- | --- | --- |
| Seller-product relation | [x] | Product create/update menolak seller inactive untuk produk baru. |
| Seller history | [x] | Seller historis jadi inactive, tidak hard-delete. |
| Product-transaction relation | [x] | Sold product tidak bisa dihapus; transaction item snapshot tersimpan. |
| Orphan prevention via UI/service | [x] | Delete seller/category/condition yang dipakai menjadi inactive. |
| Manual JSON tampering repair | [~] | Issue dilog, belum auto-repair semua referensi rusak. |
| Atomic single-file write | [x] | Temp file + fsync + replace. |
| Atomic-ish multi-file write | [~] | Ada rollback `.bak`; tetap bukan ACID database. |
| Backup corrupt JSON | [x] | Backup timestamp dibuat sebelum recovery. |
| Recovery confirmation | [x] | App mode manual meminta pilihan user. |
| Migration versioning | [ ] | Tidak ada schema version formal; sanitizer/fallback dipakai. |
| Multi-user safety | [ ] | Tidak ada file lock antar proses/database transaction. |

## 3. Baseline vs Updated Comparison

| Area | Project Pertama | Project Kedua | Perubahan | Dampak | Status |
| --- | --- | --- | --- | --- | --- |
| Framework GUI | CustomTkinter | PyQt6 | Migrasi penuh ke Qt widgets | UI lebih responsif dan profesional | [x] |
| Navigation | Sidebar + pack/forget cache | `QStackedWidget` + cached views | Switching lebih ringan | Mengurangi rebuild view | [x] |
| Performa | Banyak destroy/create widget; benchmark berat | View cache, Qt layout, pagination, debounce | Lebih siap demo | Masih perlu visual resize QA | [~] |
| Products | CRUD produk dasar | CRUD + master category/condition + date picker | Intake lebih lengkap | Menutup banyak feedback juri | [x] |
| Sellers | CRUD, delete seller available-only block | Safe delete: inactive bila historis | Histori lebih aman | Menutup orphan seller risk | [x] |
| Category CRUD | Belum ada | Dialog CRUD | Master data tersedia | Feedback juri ditutup | [x] |
| Condition CRUD | Belum ada | Dialog CRUD | Master data tersedia | Feedback juri ditutup | [x] |
| Upload image | File picker sudah ada, path disabled | File picker PyQt6, field read-only | Tetap ada, UI lebih native | Perlu QA manual picker | [~] |
| Seller phone validation | Backend digit-only | UI regex + backend digit-only | Lebih kuat | Input huruf ditolak lebih awal | [x] |
| Seller name display | `ID - Name` | `ID - Name`, inactive label historis | Lebih aman untuk histori | Juri mudah paham | [x] |
| Date picker | Manual text | `QDateEdit` calendar popup | Lebih baik | Popup perlu manual QA | [~] |
| Commission inheritance | Ada, tetapi field state kurang jelas | Read-only, mengikuti seller | Lebih jelas | Mengurangi input berulang | [x] |
| Available stock | Product card menampilkan stock | Product card + cart row available stock | Lebih jelas | Menutup feedback checkout | [x] |
| Quantity cart | Tidak ada | Plus/minus qty dan stock limit | Perbaikan besar | Stock >1 usable | [x] |
| Payment flow | Ada cash/change, cukup jelas | Button state, quick pay, confirmation | Lebih aman | Mengurangi transaksi salah | [x] |
| Checkout confirmation | Tidak ada | Dialog Confirm Payment | Menutup double-confirm issue | Cancel tidak menyimpan | [x] |
| Delete confirmation | Ada | Ada + pesan histori/inactive | Lebih jelas | Aman saat demo | [x] |
| Seller inactive handling | Tidak ada | Ada `is_active` | Histori seller aman | Menutup risiko Q&A | [x] |
| JSON backup | Tidak ada | Backup corrupt dan `.bak` rollback multi-write | Lebih reliable | Bukan database ACID | [~] |
| JSON recovery | Reset seed otomatis | Manual recovery choice | Lebih aman | Perlu manual GUI recovery QA | [~] |
| Logging | Hampir tidak ada | `logs/app.log` + notification logs | Lebih audit-able | Belum full audit log CRUD | [~] |
| Telegram async | Checkout blocking | Sale alert async; manual actions background | UI tidak freeze karena network | Live Telegram belum diuji | [~] |
| Build `.exe` | Spec absolute path | Spec relatif `SPECPATH`, dist ada | Lebih portable | Perlu QA di device lain | [~] |
| Remaining risks | Banyak P0 data/security | Risiko tersisa lebih terbatas | Siap demo final | Tetap prototype local JSON | [~] |

## 4. Judge Feedback Closure

| Catatan Juri | Implementasi Terbaru | Bukti dari Kode | Cara Verifikasi | Status |
| --- | --- | --- | --- | --- |
| Category dan condition seharusnya memiliki CRUD tersendiri | Dialog `Manage Master Data` dengan tab Categories/Conditions | `views/master_data_dialog.py`, `services/master_data_service.py:45`, `:55`, `:69` | Jalankan Products > Manage Master Data; tests `p1_master_data_smoke.py` | [x] |
| Image seharusnya upload, bukan path manual | Field image read-only, tombol Upload pakai file picker | `views/products_view.py:448`, `:549` | Klik Upload dan pilih gambar | [~] |
| Phone seller hanya boleh angka | UI regex dan backend digit-only | `views/sellers_view.py`, `utils/validators.py` | Input huruf di Phone; save via backend test | [x] |
| Admin harus dapat menambahkan seller | Seller form + `SellerService.create()` | `views/sellers_view.py`, `services/seller_service.py:35` | Login admin > Sellers > Save Seller | [x] |
| Seller pada product management harus menggunakan nama, bukan hanya ID | Combo menampilkan `seller_id - name` | `views/products_view.py:472`, `:497` | Buka produk consignment | [x] |
| Tanggal harus menggunakan calendar | `NoWheelDateEdit` dengan calendar popup | `views/products_view.py:414` | Klik Entry Date calendar | [~] |
| Commission tidak perlu diisi berulang karena mengikuti seller | Commission read-only dan otomatis dari seller | `views/products_view.py:530`, `:539` | Pilih ownership consignment dan seller | [x] |
| Available items harus terlihat jelas ketika checkout | Header Available Items, count, card Stock, cart Available Stock | `views/checkout_view.py`, `:808` | Buka Checkout dan lihat product/card cart | [x] |
| Payment flow tidak boleh membingungkan | Cash Received, quick payment, validation label, change, confirmation | `views/checkout_view.py:725`, `:750`, `:962` | Coba underpay, exact, confirm/cancel | [x] |
| Harus ada confirmation sebelum delete | Product, seller, master data delete pakai `QMessageBox.question()` | `views/products_view.py`, `views/sellers_view.py:246`, `views/master_data_dialog.py` | Klik delete | [x] |
| Logout harus mudah ditemukan | Tombol Logout di sidebar bawah | `app.py:256` | Login lalu lihat sidebar | [~] |
| Tidak boleh ada button terpotong | Layout Qt, fixed widths, manual QA sebagian centang | `MANUAL_QA_RELEASE_CHECKLIST.md` | Visual QA semua halaman/resolusi | [~] |
| Aplikasi tidak boleh berat saat pindah halaman dan checkout | PyQt6, view cache, async Telegram, pagination | `app.py:286`, `views/common.py:635`, tests PASS | Manual resize/switching di mesin demo | [~] |

## 5. Remaining Risks

1. Password masih plaintext di `data/users.json`; jangan klaim security kuat.
2. JSON lokal tetap bukan database multi-user; tidak aman untuk dua kasir menulis bersamaan dari proses berbeda.
3. Accounting/settlement belum lengkap; seller commission masih payout liability/reporting, bukan workflow payout final.
4. Telegram live belum diuji dan token disimpan plaintext jika user mengisi lewat Settings.
5. Manual QA belum lengkap untuk master data dialog, recovery dialog, full checkout visual, Telegram, dan perangkat Windows lain.
6. Transaction snapshot menyimpan item per unit; tidak ada field quantity agregat di transaction item.
7. Tidak ada refund/void/edit transaction/export report/print receipt.
8. Warning Qt font directory muncul saat test; perlu cek visual font di mesin demo.
9. Build one-folder butuh folder writable; jangan install di `Program Files` tanpa strategi writable data.
10. Manual edit JSON yang merusak referensi hanya dilog, belum auto-repair penuh.

## 6. Top 15 Most Likely Questions

| Pertanyaan | Probabilitas | Jenis juri | Alasan muncul | Jawaban ideal | Bukti di aplikasi | Risiko follow-up | Hal yang jangan diklaim |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Apa perbedaan ReWear POS dengan POS biasa? | Sangat tinggi | Product | Domain thrift/consignment jadi nilai utama | ReWear POS tidak hanya mencatat penjualan, tetapi membedakan barang milik toko dan titipan seller, menghitung store margin dan seller share, memantau aging/dead stock, serta menjaga histori seller. | Dashboard, Products, Sellers, Reports | "Apakah payout sudah lengkap?" | Jangan klaim accounting lengkap. |
| Bagaimana model consignment bekerja? | Sangat tinggi | Business | Juri akan cek logika seller/commission | Produk consignment wajib seller aktif. Commission percent mengikuti seller dan dihitung sebagai seller share. Store margin adalah sisa dari harga jual. | `ProductService`, `TransactionService` | "Commission ini fee toko atau hak seller?" | Jangan pakai istilah ambigu. |
| Apakah category dan condition sudah CRUD? | Sangat tinggi | Product/UX | Catatan juri eksplisit | Sudah, admin buka Manage Master Data di Products. Category/condition bisa add/edit/delete; jika sudah dipakai, di-inactive-kan untuk menjaga history. | `MasterDataDialog`, tests P1 | "Kenapa tidak menu sendiri?" | Jangan bilang CRUD kompleks penuh seperti ERP. |
| Bagaimana mencegah seller historis hilang? | Sangat tinggi | Data | Baseline punya risiko seller deleted | Seller yang pernah dipakai produk/transaksi tidak dihapus permanen, tetapi inactive. Jadi histori transaksi tetap punya seller record. | `SellerService.delete()` | "Bisa reactivate?" | Jangan klaim lifecycle seller lengkap jika belum ada reactivate UI. |
| Bagaimana checkout mencegah quantity melebihi stok? | Sangat tinggi | Technical | POS harus jaga stok | UI membatasi tombol plus sampai stock, backend menghitung requested quantity ulang sebelum save. | `CheckoutView`, `TransactionService` | "Bagaimana dua kasir bersamaan?" | Jangan klaim multi-user aman. |
| Apa yang terjadi saat user klik Complete Transaction? | Tinggi | UX/Data | Payment flow dinilai | App validasi pembayaran, tampilkan confirmation, lalu service menulis produk dan transaksi bersama. Cancel tidak mengubah data. | `confirm_checkout`, tests P0 | "Apakah bisa double-click?" | Jangan bilang mustahil secara absolut. |
| Mengapa memakai JSON lokal? | Tinggi | Technical | JSON sering dipertanyakan | Untuk prototype lomba single-device, JSON ringan, portable, mudah dibuild ke EXE. Kami menambahkan atomic write, backup, recovery, dan logging. Untuk produksi multi-user, migrasi ke SQLite/PostgreSQL. | `StorageService`, README | "Kenapa bukan SQLite sekarang?" | Jangan klaim setara database server. |
| Bagaimana mencegah data rusak? | Tinggi | Data | Baseline lemah di sini | Write pakai temp file, flush, fsync, os.replace. Jika JSON corrupt, file dibackup dan user memilih recover atau exit. | `StorageService` | "Apakah ACID?" | Jangan klaim ACID penuh. |
| Apakah Telegram aman dan selalu berhasil? | Tinggi | Security | Baseline token bocor | Token baseline sudah dikosongkan di data terbaru. Telegram opsional, env var bisa override, network call async. Tetapi live delivery tergantung token/internet dan perlu QA. | `settings.json`, `TelegramService` | "Token disimpan di mana?" | Jangan klaim terenkripsi. |
| Apakah aplikasi sudah production-ready? | Tinggi | Technical/Product | Juri red-team | Untuk final lomba dan single-device demo, siap. Untuk produksi, perlu database, user management, audit log, printer/export, backup policy, dan multi-device sync. | Remaining risks | "Apa roadmap?" | Jangan jawab "sudah production-ready". |
| Apa evidence bahwa versi terbaru lebih cepat? | Tinggi | Technical | Ada migrasi UI | Migrasi ke PyQt6, QStackedWidget cache, pagination, debounce, async Telegram. Automated tests PASS; manual resize/FPS masih harus QA. | `app.py`, tests | "Ada angka benchmark?" | Jangan klaim bebas lag/FPS tertentu. |
| Bagaimana receipt dan transaction history disimpan? | Tinggi | Data | POS core | Transaction menyimpan snapshot item: nama, kategori, harga, seller_id, seller commission, store margin, total, paid, change. | `transactions.json`, `TransactionService` | "Ada print receipt?" | Jangan klaim print/export. |
| Apakah cashier bisa akses menu admin? | Sedang | Security/Role | Role sederhana | UI untuk cashier hanya menampilkan Checkout. Ini role navigation untuk demo; service-level RBAC belum dibuat. | `app.py:214` | "Kalau route dipanggil langsung?" | Jangan klaim security enterprise. |
| Bagaimana build EXE dijamin portable? | Sedang | Technical | Final butuh deliverable | Spec sudah relatif `SPECPATH`, assets/data/qtawesome dimasukkan, dist build ada dan release report mencatat startup/write probe. Tetap perlu QA di mesin demo. | `ReWearPOS.spec`, release report | "Di PC lain?" | Jangan klaim semua PC pasti aman. |
| Apakah ada settlement seller? | Sedang | Business/Accounting | Commission butuh payout | Saat ini sistem menghitung seller commission sebagai payout liability dan summary seller. Workflow settlement paid/unpaid belum tersedia. | Sellers, Reports | "Accounting lengkap?" | Jangan klaim accounting lengkap. |

## 7. Follow-Up Questions

| Pertanyaan | Probabilitas | Jenis juri | Alasan muncul | Jawaban ideal | Bukti di aplikasi | Risiko follow-up | Hal yang jangan diklaim |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Kenapa category/condition inactive, bukan delete saja? | Tinggi | Data | Histori produk harus aman | Karena produk lama masih membutuhkan nilai lama untuk laporan dan histori. Jika tidak dipakai, bisa hard delete; jika sudah dipakai, inactive. | `MasterDataService.delete()` | Reactivate? | Jangan klaim lifecycle lengkap. |
| Apa arti seller commission? | Tinggi | Business | Istilah rawan ambigu | Di aplikasi ini artinya seller share/payout liability, bukan fee yang dibayar seller ke toko. | Reports/Sellers copy | Rumus? | Jangan menyebut "komisi toko". |
| Kalau harga produk berubah setelah transaksi? | Sedang | Data | Snapshot diuji | Transaksi lama tetap aman karena menyimpan snapshot harga, margin, dan seller commission. | `transactions.json` | Histori perubahan master? | Jangan klaim audit trail master. |
| Bagaimana jika file JSON diedit manual? | Tinggi | Red-team | JSON risk | Sanitizer menangani beberapa format invalid dan integrity issue dilog; jika corrupt, backup/recovery. Referensi rusak tidak auto-repair penuh. | `StorageService`, `SellerService` | Bisa mencegah? | Jangan klaim anti-tamper. |
| Apakah app bisa dipakai dua kasir? | Tinggi | Technical | POS naturally multi-cashier | Role cashier ada, tetapi storage JSON saat ini dirancang untuk single-device/single-process demo. Multi-kasir perlu DB/locking. | Storage design | Roadmap? | Jangan klaim multi-user safe. |
| Mengapa belum pakai SQLite? | Sedang | Technical | Alternative obvious | Fokus final adalah menyelesaikan UX dan data integrity prototype. SQLite adalah upgrade natural berikutnya untuk transaction/locking/query. | Release risks | Kenapa tidak sekarang? | Jangan menyerang pilihan DB. |
| Apakah Telegram memblokir checkout? | Sedang | UX/Technical | Baseline blocking | Sale alert dikirim async setelah transaksi berhasil, jadi checkout tidak menunggu network. | `send_message_async` | Thread safety? | Jangan klaim delivery selalu sukses. |
| Bagaimana kalau recovery dipilih? | Sedang | Data | Recovery workflow baru | File rusak sudah dibackup. Jika user pilih recover, app menulis fallback default agar bisa dibuka, lalu user review data. | `confirm_storage_recovery` | Restore backup otomatis? | Jangan klaim backup otomatis ter-restore. |
| Bagaimana menguji fitur? | Tinggi | Technical | Bukti kualitas | Ada compileall, P0 smoke, P1 master data smoke, final judge smoke, route instantiate offscreen, release report. | `tests/` | Manual QA? | Jangan klaim semua visual otomatis. |
| Mengapa transaction item tidak punya quantity field? | Sedang | Data | Quantity baru | Service menyimpan item per unit untuk kompatibilitas snapshot lama. UI menghitung quantity di cart, transaksi mencatat tiap unit item. | `TransactionService` | Report quantity? | Jangan klaim model data final ideal. |

## 8. Killer Questions

| Pertanyaan | Probabilitas | Jenis juri | Alasan muncul | Jawaban ideal | Bukti di aplikasi | Risiko follow-up | Hal yang jangan diklaim |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Password masih plaintext, bagaimana bisa disebut aman? | Tinggi | Security | Data users jelas plaintext | Untuk demo lomba, akun demo plaintext masih diterima sebagai prototype. Kami tidak mengklaim security produksi; tahap berikutnya hash password, user management, dan permission guard service. | `data/users.json`, `AuthService` | Kenapa belum hash? | Jangan bilang aman. |
| Kalau listrik mati saat checkout, apa data aman? | Tinggi | Data | POS reliability | Write produk+transaksi memakai temp file, fsync, replace, dan rollback backup. Ini mengurangi risiko partial write, tetapi belum setara database ACID. | `write_many_json` | Edge case crash di tengah multi-file? | Jangan klaim 100% mustahil rusak. |
| Dua kasir menjual barang sama bersamaan, apa yang terjadi? | Tinggi | Technical | JSON local race | Sistem sekarang single-machine/single-process. Backend validasi stok di proses itu; multi-process/multi-device butuh database transaction. | Storage architecture | Bisa dipakai toko real? | Jangan klaim multi-cashier. |
| Apakah accounting sudah lengkap? | Tinggi | Business | Ada commission/report | Belum. Saat ini accounting lite: revenue, margin, seller payout liability. Settlement paid/unpaid, ledger, refund, tax belum ada. | Reports/Sellers | Payout seller? | Jangan sebut accounting lengkap. |
| Kalau Telegram token diisi, apakah terenkripsi? | Sedang | Security | Settings plaintext | Belum terenkripsi. Token bisa disimpan di JSON atau env var override. Untuk demo gunakan token khusus, jangan token pribadi. | `settings.json`, `TelegramService.config()` | Token leak? | Jangan klaim secrets management. |
| Kenapa recovery fallback memakai seed, bukan restore backup otomatis? | Sedang | Data | Recovery design | Karena kalau file corrupt, seed membuat app bisa dibuka aman. Backup tetap dibuat agar data lama bisa diperiksa manual. Restore otomatis corrupt file tidak aman. | `prepare_json_recovery` | Bisa import backup? | Jangan klaim recovery sempurna. |
| Apakah bebas lag saat resize? | Sedang | UX/Performance | Manual QA belum penuh | Tidak kami klaim bebas lag. Kami sudah migrasi ke PyQt6, cache view, pagination, debounce; final tetap harus diuji manual di mesin demo. | `FINAL_AUDIT_CHECKLIST` | Ada benchmark? | Jangan klaim FPS. |
| Kenapa master data tidak punya ID? | Sedang | Data | Category/condition string | Untuk kompatibilitas data produk lama, category/condition tetap string. Inactive menjaga histori. Ke depan bisa ditingkatkan ke ID master data. | `products.json`, `MasterDataService` | Rename used item? | Jangan klaim relational schema. |
| Bagaimana jika seller inactive harus dipakai lagi? | Sedang | Business | Inactive workflow | Saat ini inactive seller dijaga untuk histori dan tidak tersedia untuk intake baru. Jika seller kembali aktif, fitur reactivate adalah enhancement berikutnya. | `SellerService.active()` | Bisa edit JSON? | Jangan klaim ada reactivate UI. |
| Apakah build pasti jalan di komputer juri? | Sedang | Build | Packaging risk | Build one-folder sudah dibuat dan report mencatat startup/write probe. Tetap perlu menjalankan checklist di mesin demo/juri, terutama font, writable data, dan antivirus. | `dist`, release report | Kalau folder read-only? | Jangan klaim universal. |

## 9. Rapid-Fire Questions

| Pertanyaan | Probabilitas | Jenis juri | Alasan muncul | Jawaban ideal | Bukti di aplikasi | Risiko follow-up | Hal yang jangan diklaim |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Framework? | Tinggi | Technical | Basic | Python + PyQt6. | `requirements.txt` | Kenapa Qt? | Jangan sebut CustomTkinter versi final. |
| Storage? | Tinggi | Technical | Basic | Local JSON dengan atomic write dan recovery. | `StorageService` | Multi-user? | Jangan klaim DB. |
| Role? | Tinggi | Security | Basic | Admin full menu, cashier Checkout only. | `app.py` | Service RBAC? | Jangan klaim deep RBAC. |
| Token Telegram default? | Sedang | Security | Baseline risk | Kosong di data terbaru; env var bisa override. | `data/settings.json` | Encryption? | Jangan klaim terenkripsi. |
| Print receipt? | Sedang | Product | POS expectation | Belum, receipt modal dan history tersedia. | Checkout/Transactions | Roadmap? | Jangan klaim print ada. |
| Refund/void? | Sedang | Product | POS expectation | Belum tersedia. | Transactions view | Accounting? | Jangan improvisasi. |
| Category duplicate? | Sedang | Data | Master data | Ditolak case-insensitive. | P1 tests | Unicode edge? | Jangan overclaim. |
| Product sold delete? | Tinggi | Data | Integrity | Ditolak oleh service. | `ProductService.delete()` | Hard delete DB? | Jangan bilang semua manipulasi dicegah. |
| Backup location? | Sedang | Data | Recovery | Di folder `data` dengan suffix timestamp `.backup`; log di `logs/app.log`. | `StorageService` | Restore? | Jangan klaim auto-restore. |
| Current demo stock? | Sedang | Demo | Demo readiness | Data saat audit: 3 available, total stock 4. Siapkan data demo agar tidak habis. | JSON summary | Reset? | Jangan checkout sembarang di data utama. |

## 10. Demo-Specific Questions

| Pertanyaan | Probabilitas | Jenis juri | Alasan muncul | Jawaban ideal | Bukti di aplikasi | Risiko follow-up | Hal yang jangan diklaim |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Tunjukkan alur consignment dari seller ke checkout. | Tinggi | Demo/Product | End-to-end | Buat/pilih seller aktif, tambah produk consignment, commission otomatis, checkout, lihat seller commission di report. | Products, Checkout, Reports | Stok habis | Jangan gunakan data utama tanpa rencana. |
| Tunjukkan category CRUD. | Tinggi | Demo/UX | Feedback juri | Products > Manage Master Data > Categories > Add/Edit/Delete. | MasterDataDialog | Visual dialog | Jangan klaim sudah screenshot-tested jika belum. |
| Tunjukkan payment kurang dari total. | Tinggi | Demo/UX | Payment flow | Masukkan nominal kurang; tombol/validation mencegah checkout. | Checkout | Bisa bypass? | Jangan klaim mutlak jika edit JSON. |
| Tunjukkan cancel confirmation. | Tinggi | Demo/Data | Transaksi ganda | Klik Complete, Cancel; stok/transaksi tidak berubah. | P0 test | Buktikan? | Jangan lakukan pada stok terakhir tanpa rencana. |
| Tunjukkan recovery JSON. | Sedang | Demo/Data | Reliability | Hanya di salinan data, app backup file rusak dan user pilih recover/exit. | Manual QA checklist | Merusak data utama | Jangan demo di data utama. |

## 11. Accounting or Settlement Questions

| Pertanyaan | Probabilitas | Jenis juri | Alasan muncul | Jawaban ideal | Bukti di aplikasi | Risiko follow-up | Hal yang jangan diklaim |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Apakah seller sudah bisa dibayar dari sistem? | Tinggi | Accounting | Commission report | Belum ada payout workflow. Sistem menghitung seller payout liability dan total commission earned. | Sellers/Reports | Paid/unpaid? | Jangan klaim settlement lengkap. |
| Bagaimana rumus margin toko? | Tinggi | Accounting | Core business | Store-owned: selling - cost. Consignment: selling - seller share. | `TransactionService` | Discount/tax? | Jangan klaim tax/discount ada. |
| Apakah seller commission berubah jika default seller diedit? | Sedang | Accounting | Histori | Transaksi lama aman karena snapshot. Produk baru mengikuti default saat dipilih. | Transaction snapshot | Product lama? | Jangan klaim audit perubahan master. |
| Apakah ada laporan utang seller per periode? | Sedang | Accounting | Payout | Ada seller commission aggregate all-time; filter periode belum tersedia. | Reports/Sellers | Export? | Jangan klaim laporan lengkap. |
| Apakah refund mengurangi commission? | Sedang | Accounting | POS real | Refund/void belum tersedia; roadmap accounting berikutnya. | Transactions | Real deployment? | Jangan mengarang fitur. |

## 12. Simulasi Q&A 15 Menit

### Menit 00:00–03:00 — Pertanyaan Utama
**Juri 1:** Apa masalah utama yang ReWear POS selesaikan?

**Jawaban Tim:** ReWear POS menyelesaikan masalah toko thrift yang menjual barang milik toko dan barang titipan seller dalam satu alur. Sistem membedakan ownership produk, menghitung margin toko dan seller share saat checkout, menjaga histori transaksi, dan memberi insight aging/dead stock untuk owner.

**Follow-up Juri:** Apa bedanya dengan POS biasa?

**Jawaban Tim:** POS biasa biasanya fokus pada kasir dan stok. ReWear POS menambahkan konteks consignment: seller, default commission, seller payout liability, dan proteksi histori seller/category agar laporan tetap konsisten.

### Menit 03:00–06:00 — UX dan Business Logic
**Juri 1:** Catatan sebelumnya menyebut category dan condition harus CRUD. Sudah?

**Jawaban Tim:** Sudah. Di Products ada tombol Manage Master Data. Admin bisa add, edit, delete category dan condition. Kalau data sudah dipakai produk, sistem tidak menghapus permanen, tetapi menjadikannya inactive agar produk historis tetap terbaca.

**Follow-up Juri:** Kenapa tidak dihapus saja?

**Jawaban Tim:** Karena category/condition lama masih dipakai laporan dan histori produk. Hard delete hanya aman jika belum ada referensi.

**Juri 1:** Commission masih harus diisi manual?

**Jawaban Tim:** Tidak. Untuk consignment, commission mengikuti default seller dan field dibuat read-only. Untuk store-owned, commission otomatis 0.

### Menit 06:00–10:00 — Technical dan Data Integrity
**Juri 2:** Mengapa masih JSON?

**Jawaban Tim:** Untuk scope lomba dan demo single-device, JSON membuat aplikasi ringan, portable, dan mudah dibuild ke EXE. Kami menambahkan atomic write, backup corrupt file, manual recovery, sanitizer, dan logging. Untuk produksi multi-user, kami akan migrasi ke SQLite atau PostgreSQL.

**Follow-up Juri:** Kalau file rusak?

**Jawaban Tim:** App membackup file rusak dengan timestamp dan menampilkan pilihan recover default data atau exit. Jadi tidak lagi silent reset seperti baseline.

**Juri 2:** Bagaimana mencegah transaksi ganda?

**Jawaban Tim:** UI punya confirmation sebelum save dan guard `checkout_in_progress`. Backend juga validasi stok sebelum menulis. Test reentrant/double-click guard sudah PASS.

### Menit 10:00–13:00 — Killer Questions
**Juri 2:** Password masih plaintext. Ini aman?

**Jawaban Tim:** Untuk prototype lomba, ini akun demo lokal. Kami tidak mengklaim keamanan produksi. Yang sudah kami perbaiki adalah token Telegram aktif tidak lagi ada di data, dan data write/recovery lebih aman. Roadmap security berikutnya adalah password hashing, user management, dan permission guard service.

**Juri 2:** Dua kasir menjual barang yang sama bersamaan?

**Jawaban Tim:** Versi ini belum dirancang untuk multi-kasir multi-process. Dalam satu aplikasi, stok divalidasi sebelum save. Untuk multi-kasir, perlu database transaction/locking. Itu batasan yang sengaja kami nyatakan.

### Menit 13:00–15:00 — Rapid Fire
**Juri 1:** Print receipt?

**Jawaban Tim:** Belum print fisik; saat ini receipt modal dan history tersedia.

**Juri 2:** Telegram live sudah diuji?

**Jawaban Tim:** Belum dalam audit ini. Fiturnya async dan opsional; sebelum demo harus diuji dengan token demo.

**Juri 1:** Settlement seller?

**Jawaban Tim:** Belum payout workflow. Saat ini seller commission adalah payout liability/reporting.

**Juri 2:** Production-ready?

**Jawaban Tim:** Siap untuk demo final single-device, belum production-ready.

## 13. Jawaban Siap Hafal

| Topik | 15 detik | 30 detik | Jika follow-up |
| --- | --- | --- | --- |
| Masalah yang diselesaikan | ReWear POS membantu toko thrift mencatat penjualan, stok, seller, dan margin dalam satu aplikasi. | Fokusnya adalah toko thrift/consignment yang punya barang milik toko dan barang titipan seller. Sistem membantu checkout, stock, seller share, laporan, dan aging stock. | Masalah utama bukan hanya kasir, tetapi transparansi margin dan komisi seller. |
| Fokus thrift dan consignment | Karena thrift punya barang unik dan titipan seller yang tidak cocok ditangani POS generik. | Produk thrift sering satuan, punya kondisi, tanggal masuk, dan bisa milik toko atau seller. Itu butuh logic seller share dan aging stock. | Ini niche yang membuat solusi lebih relevan daripada POS umum. |
| Bedanya dengan POS biasa | Ada seller, consignment, seller share, aging/dead stock, dan margin toko. | POS biasa biasanya hanya jual-beli. ReWear POS menjaga relasi seller, ownership produk, commission, dan histori agar owner tahu liability ke seller. | Kami tidak mengklaim menggantikan ERP, tapi menyelesaikan pain point thrift boutique. |
| Python dan PyQt6 | Python cepat untuk prototype; PyQt6 memberi UI desktop yang lebih responsif. | Versi awal CustomTkinter cukup, tetapi final pindah ke PyQt6 untuk layout, stack navigation, widget table, date picker, dan performa demo. | PyQt6 juga lebih cocok untuk build desktop Windows dengan PyInstaller. |
| JSON lokal | Dipilih untuk demo single-device yang ringan dan portable. | JSON membuat setup sederhana tanpa server. Kami menambahkan atomic write, backup, recovery, dan logging. Untuk multi-user, roadmap-nya database. | Jangan klaim JSON untuk banyak kasir. |
| Mencegah data rusak | Write memakai temp file, fsync, replace, backup, dan recovery manual. | Jika file corrupt, app membackup file rusak lalu user memilih recover default data atau exit untuk inspeksi backup. | Ini mitigasi prototype, bukan ACID penuh. |
| Mencegah transaksi ganda | Ada confirmation dan guard `checkout_in_progress`. | Complete Transaction tidak langsung save; user review payment. Reentrant/double-click guard diuji agar tidak membuat transaksi ganda. | Multi-process race tetap butuh database. |
| Menjaga stok | UI membatasi quantity dan backend memvalidasi ulang stok. | Checkout menghitung requested quantity, menolak overstock, lalu menulis produk+transaksi bersama. | Jika dua app terpisah jalan bersamaan, itu batasan JSON. |
| Seller dan commission | Seller punya default commission; produk consignment mengikuti seller. | Commission di sini berarti seller share/payout liability. Store margin dihitung dari harga jual dikurangi seller share. | Jangan menyebut komisi toko jika maksudnya bagian seller. |
| Histori seller | Seller historis tidak dihapus permanen, tetapi inactive. | Jika seller sudah punya produk/transaksi, delete akan membuatnya inactive agar histori tetap bisa dilacak. | Reactivate UI belum tersedia. |
| Mengapa seller tidak selalu dihapus | Untuk mencegah transaksi lama kehilangan konteks seller. | Hard delete hanya aman untuk seller tanpa referensi. Seller yang pernah dipakai disimpan sebagai inactive. | Ini tradeoff integrity di atas pembersihan data. |
| Category/condition management | Ada CRUD di Manage Master Data. | Category dan condition bisa add/edit/delete; used item menjadi inactive agar produk historis tetap aman. | Master data masih string-compatible untuk data lama. |
| Available stock | Stock terlihat di product card dan cart. | Checkout menampilkan Available Items dan Available Stock per cart row, plus tombol qty disabled saat stok tercapai. | Bisa demo dengan produk stock >1. |
| Payment flow | User isi cash received, lihat change, lalu confirm. | Payment kurang ditolak, quick pay tersedia, dan confirmation dialog muncul sebelum stok/transaksi berubah. | Cancel tidak mengubah data. |
| Telegram | Opsional, async, tidak memblokir checkout. | Token/chat ID bisa dari settings atau env var. Sale alert dikirim async; manual dashboard/settings juga background task. | Live delivery tergantung internet/token. |
| Build EXE | Build one-folder tersedia di `dist\ReWearPOS`. | Spec sudah relatif, memasukkan data, assets, qtawesome, dan hidden imports. Release report mencatat build sukses. | Tetap uji di mesin demo dan folder writable. |
| Keterbatasan | Belum production-ready dan belum multi-user. | Batasan utama: JSON lokal, password demo plaintext, belum print/refund/export/settlement, Telegram live perlu QA. | Jawaban jujur lebih defensif daripada overclaim. |
| Pengembangan lanjutan | Database, user security, settlement, printer/export. | Roadmap paling logis: SQLite/PostgreSQL, role guard, password hashing, refund/void, payout seller, report periode, receipt printer. | Prioritaskan sesuai pertanyaan juri. |
| Accounting/settlement | Saat ini accounting lite, bukan settlement lengkap. | Sistem menghitung revenue, store margin, dan seller payout liability, tetapi belum ada status paid/unpaid atau ledger. | Jangan klaim accounting lengkap. |
| Keamanan dasar | Ada role UI dan token kosong, tetapi password belum hashed. | Security baseline diperbaiki sebagian: token aktif tidak ada, recovery/logging lebih baik. Namun password demo plaintext dan JSON editable masih batasan. | Jawab sebagai prototype competition. |

## 14. Claims That Must Be Avoided

- "Production-ready."
- "Aman untuk multi-user atau banyak kasir."
- "Database kami aman seperti DBMS."
- "Accounting lengkap."
- "Settlement seller sudah selesai."
- "Telegram selalu berhasil."
- "Token Telegram terenkripsi."
- "Password sudah aman."
- "Data tidak mungkin rusak."
- "Aplikasi bebas bug atau bebas lag."
- "Sudah ada print receipt/export report/refund/void" jika belum ditunjukkan kode.
- "Manual QA semua selesai" karena checklist masih banyak item kosong.

## 15. Final Presentation Defense Strategy

1. Buka dengan domain problem: thrift + consignment, bukan POS generik.
2. Tunjukkan tiga perbaikan besar dari baseline: PyQt6 performance/navigation, master data CRUD, checkout quantity/confirmation.
3. Saat demo, gunakan data salinan atau siapkan produk available karena data aktual hanya 3 produk available dengan total stock 4.
4. Saat juri masuk security/data integrity, jawab jujur: JSON lokal untuk prototype single-device, sudah diperkuat dengan atomic write/recovery/logging, produksi butuh database.
5. Saat juri menekan accounting, pakai istilah "seller payout liability", bukan "accounting lengkap".
6. Jangan demo recovery atau Telegram di data utama. Gunakan salinan atau token demo.
7. Tutup dengan roadmap yang realistis: SQLite/PostgreSQL, settlement paid/unpaid, printer/export, password hashing, audit log, dan multi-device sync.

