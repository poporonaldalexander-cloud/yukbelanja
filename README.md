# Yuk Belanja — Toko Online (Next.js + Midtrans)

Toko pakaian e-commerce dengan **database produk**, **panel admin** (kelola produk + upload foto), dan **pembayaran asli via Midtrans Snap** (QRIS, e-wallet, Virtual Account, kartu kredit). Siap deploy di Vercel.

> Ini **starter project**. Kode sudah berjalan, tapi Anda perlu mengisi kunci/akun Anda sendiri (database, Midtrans, Vercel Blob) dan mengujinya sebelum go-live. Belum diuji di mesin Anda — jalankan langkah di bawah dan beri tahu saya jika ada error.

---

## Yang sudah termasuk

- **Storefront** (`/`) — katalog produk dari database, navigasi kategori/subkategori, pencarian, keranjang.
- **Checkout + Midtrans Snap** — order disimpan, total dihitung ulang di server (anti-manipulasi harga), popup pembayaran Midtrans.
- **Webhook** (`/api/midtrans/notification`) — verifikasi tanda tangan, update status order otomatis, **dan kurangi stok otomatis** saat lunas (idempotent: tidak dobel-potong).
- **Admin produk** (`/admin`) — login sederhana, tambah/ubah/hapus produk, upload foto ke Vercel Blob.
- **Admin pesanan** (`/admin/orders`) — lihat semua pesanan & ubah status (Menunggu → Lunas → Dikirim → Selesai).
- **Lacak pesanan pelanggan** (`/lacak`) — pelanggan cek status pakai nomor pesanan + email.
- **Email konfirmasi** (opsional) — terkirim otomatis saat pembayaran lunas bila Resend dikonfigurasi.

## Stack
Next.js 14 (App Router) · Prisma + PostgreSQL · Midtrans (`midtrans-client`) · Vercel Blob.

---

## A. Menjalankan di lokal

```bash
# 1. Install dependency
npm install

# 2. Siapkan environment
cp .env.example .env
# lalu isi nilai-nilai di .env (lihat bagian C & D)

# 3. Buat tabel di database & isi contoh produk
npm run db:push
npm run db:seed

# 4. Jalankan
npm run dev
# buka http://localhost:3000  dan  http://localhost:3000/admin
```

---

## B. Siapkan database (pilih satu)

Paling mudah & gratis untuk mulai: **Supabase** atau **Neon** (keduanya PostgreSQL).

1. Buat project, salin **connection string**.
2. Tempel ke `DATABASE_URL` (dan `DIRECT_URL` untuk Supabase/Neon).
3. Jalankan `npm run db:push` lalu `npm run db:seed`.

Kalau deploy di Vercel, Anda juga bisa pakai **Vercel Postgres** (Storage tab) — env-nya terisi otomatis.

---

## C. Siapkan Midtrans (pembayaran)

1. Daftar di **dashboard.midtrans.com**. Mulai dengan mode **Sandbox** (uang palsu untuk uji coba).
2. Buka **Settings → Access Keys**, salin **Server Key** dan **Client Key**.
3. Isi di `.env`:
   - `MIDTRANS_SERVER_KEY`, `MIDTRANS_CLIENT_KEY`
   - `NEXT_PUBLIC_MIDTRANS_CLIENT_KEY` (sama dengan client key)
   - Biarkan `MIDTRANS_IS_PRODUCTION="false"` selama uji coba.
4. Set **Payment Notification URL** di Midtrans (Settings → Configuration):
   `https://DOMAIN-ANDA.vercel.app/api/midtrans/notification`
5. Uji bayar pakai data kartu/akun sandbox dari dokumentasi Midtrans.
6. Saat siap go-live: lengkapi verifikasi bisnis di Midtrans, ganti key ke Production, set `MIDTRANS_IS_PRODUCTION="true"` dan `NEXT_PUBLIC_MIDTRANS_PRODUCTION="true"`.

> Untuk go-live, Midtrans biasanya meminta data usaha (KTP, dan disarankan NPWP/izin usaha). Siapkan ini lebih awal.

---

## D. Penyimpanan foto (Vercel Blob)

1. Di Vercel: **Storage → Create → Blob**.
2. Token otomatis tersedia di project. Untuk lokal, salin `BLOB_READ_WRITE_TOKEN` ke `.env`.
3. Upload foto dilakukan dari halaman `/admin`.

(Alternatif: Cloudinary — tinggal ganti isi `src/app/api/upload/route.js`.)

---

## E. Deploy ke Vercel

1. Push folder ini ke GitHub.
2. Vercel → **New Project** → import repo. Framework: **Next.js** (otomatis).
3. Tambahkan semua environment variable dari `.env` di **Settings → Environment Variables**.
4. Deploy. Setelah dapat domain, daftarkan URL webhook ke Midtrans (langkah C-4).
5. Jalankan migrasi DB sekali (lewat lokal yang menunjuk ke DB produksi, atau Vercel CLI): `npm run db:push && npm run db:seed`.

---

## F. Akun admin
- URL: `/admin` (akan minta login bila belum masuk).
- Sandi diambil dari `ADMIN_PASSWORD`. Ganti dengan sandi kuat.
- Auth ini sederhana (cookie ber-HMAC). Untuk banyak admin/keamanan lebih, ganti ke NextAuth/Clerk.

---

## Catatan keamanan & langkah lanjut yang disarankan
- ~~Pengurangan stok otomatis~~ ✅ sudah (di webhook, saat status PAID).
- ~~Dashboard pesanan~~ ✅ sudah (`/admin/orders`). ~~Tracking pelanggan~~ ✅ sudah (`/lacak`). ~~Email konfirmasi~~ ✅ sudah (opsional via Resend).
- Tambah **akun pelanggan** penuh (login, daftar alamat, riwayat) — saat ini tracking cukup pakai nomor pesanan + email.
- Stok bisa jadi negatif bila pembelian melebihi stok bersamaan; tambahkan validasi stok saat checkout bila perlu.
- Patuhi aspek legal toko online di Indonesia (izin usaha, kebijakan privasi, S&K, perlindungan konsumen).

## Konfigurasi email (opsional)
1. Daftar di **resend.com**, verifikasi domain pengirim, buat API key.
2. Isi `RESEND_API_KEY` dan `EMAIL_FROM` di `.env`.
3. Email konfirmasi otomatis terkirim saat pembayaran lunas. Jika dibiarkan kosong, fitur ini dilewati tanpa error.
