# ReWear POS — Q&A Cheat Sheet

## 10 Pertanyaan Paling Mungkin Keluar

1. Apa bedanya ReWear POS dengan POS biasa?
2. Bagaimana model consignment dan seller share dihitung?
3. Apakah category dan condition sudah punya CRUD?
4. Bagaimana sistem menjaga histori seller?
5. Bagaimana checkout mencegah quantity melebihi stok?
6. Mengapa masih memakai JSON lokal?
7. Bagaimana mencegah data JSON rusak?
8. Apakah Telegram aman dan sudah diuji?
9. Apakah aplikasi bisa dipakai banyak kasir?
10. Apakah accounting/settlement seller sudah lengkap?

## 10 Jawaban Singkat Siap Hafal

1. ReWear POS bukan POS generik; fokusnya thrift dan consignment.
2. Produk consignment wajib seller aktif, dan commission berarti seller share.
3. Store margin consignment = selling price - seller share.
4. Category dan condition dikelola lewat Manage Master Data.
5. Data yang sudah dipakai tidak dihapus sembarangan, tetapi dibuat inactive.
6. Checkout punya quantity, available stock, payment validation, dan confirmation.
7. Transaksi menyimpan snapshot harga, seller_id, margin, dan commission.
8. JSON dipakai untuk prototype single-device; sudah ada atomic write, backup, recovery, logging.
9. Telegram opsional dan async, tetapi live delivery tergantung token/internet.
10. Aplikasi siap demo final, tetapi belum production-ready atau multi-user.

## 10 Killer Questions dan Jawaban Aman

1. Password masih plaintext?
   Jawaban: Benar untuk akun demo prototype. Jangan klaim security produksi; roadmap-nya hashing dan user management.

2. Dua kasir checkout bersamaan?
   Jawaban: Belum untuk multi-process/multi-device. Versi ini single-device; multi-kasir butuh database transaction.

3. JSON rusak saat transaksi?
   Jawaban: Write memakai temp file, fsync, replace, dan rollback backup. Jika corrupt, backup dibuat dan recovery manual ditawarkan.

4. Telegram token aman?
   Jawaban: Data terbaru kosong; env var bisa dipakai. Jika disimpan di settings, belum terenkripsi, jadi gunakan token demo.

5. Accounting lengkap?
   Jawaban: Belum. Ini accounting lite: revenue, margin, seller payout liability. Settlement paid/unpaid belum ada.

6. Refund/void ada?
   Jawaban: Belum. Saat ini transaction history read-only.

7. Print receipt ada?
   Jawaban: Belum print fisik; ada receipt modal dan transaction detail.

8. Category/condition kalau sudah dipakai?
   Jawaban: Tidak dihapus permanen, dibuat inactive untuk menjaga histori produk.

9. Aplikasi bebas lag?
   Jawaban: Tidak diklaim. Sudah migrasi PyQt6, caching, debounce, async Telegram; tetap perlu QA manual.

10. Production-ready?
   Jawaban: Siap demo lomba single-device, belum production-ready.

## Fakta Teknis Penting

- Framework final: PyQt6, bukan CustomTkinter.
- Navigation: `QStackedWidget`.
- View cache: `cached_views` per route.
- Storage: local JSON.
- Atomic write: temp file + `fsync` + `os.replace`.
- Checkout multi-write: products + transactions via `write_many_json`.
- Recovery: corrupt JSON dibackup lalu user pilih recover atau exit.
- Logging: `logs/app.log`.
- Telegram: async untuk sale alert/manual actions.
- Tests PASS: compileall, P0 smoke, P1 master data smoke, final judge feedback smoke.
- Data aktual saat audit: 15 produk, 3 available, total stock 4, 5 transaksi, 2 seller aktif.
- Telegram settings aktual: kosong/tidak configured.
- Password demo masih plaintext.

## Klaim yang Harus Dihindari

- Production-ready.
- Multi-user safe.
- Accounting lengkap.
- Settlement seller lengkap.
- Telegram selalu berhasil.
- Token/password sudah aman.
- Data tidak mungkin rusak.
- Bebas bug atau bebas lag.
- Print receipt/export/refund tersedia.
- Semua manual QA sudah selesai.

## Kalimat Penutup yang Kuat

"ReWear POS kami posisikan sebagai prototype final yang fokus dan realistis untuk thrift consignment. Perbaikan terbesar dari baseline ada di integritas data, checkout yang lebih aman, master data category-condition, dan UX PyQt6. Kami jujur bahwa versi ini masih single-device dan belum accounting penuh, tetapi fondasinya sudah jelas untuk naik ke database, settlement seller, audit log, dan deployment produksi."

