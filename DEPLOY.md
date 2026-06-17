# Panduan Deploy — Yuk Belanja (Vercel + Supabase + Midtrans)

Ikuti berurutan. **Supabase dulu, baru Vercel.** Butuh Node.js terpasang di komputer Anda.

> Catatan versi: project ini memakai **Prisma 5.18**, jadi `url`/`directUrl` di `schema.prisma` tetap berlaku. **Jangan upgrade ke Prisma 7** karena cara koneksinya berubah.

---

## 0. Soal password admin (penting)

Tidak ada password bawaan. Anda menentukannya sendiri:

- `ADMIN_PASSWORD` = sandi untuk login ke `/admin`. Isi dengan sandi kuat pilihan Anda.
- `ADMIN_SECRET` = string acak panjang untuk menandatangani cookie sesi (bukan untuk diketik). Buat sekali, mis:
  ```bash
  openssl rand -hex 32
  ```
  Tempel hasilnya sebagai nilai `ADMIN_SECRET`.

Kalau `ADMIN_PASSWORD` kosong, login admin akan selalu ditolak (disengaja).

---

## 1. Buat database di Supabase

1. Daftar di https://supabase.com → **New project**.
2. Isi nama, pilih region terdekat (mis. Singapore), dan **buat password database — simpan baik-baik.**
3. Tunggu sampai project siap (1–2 menit).
4. Klik tombol **Connect** (bar atas) → tab **ORMs** → pilih **Prisma**. Salin dua string berikut.

Untuk Vercel (serverless), pakai kombinasi ini:

| Variabel | Pakai string | Port | Catatan |
|---|---|---|---|
| `DATABASE_URL` | **Transaction Pooler** | 6543 | tambahkan `?pgbouncer=true&connection_limit=1` di akhir |
| `DIRECT_URL` | **Direct connection** | 5432 | dipakai untuk migrasi (`db push`) |

Contoh bentuk:
```
DATABASE_URL="postgres://postgres.<ref>:<password>@aws-0-<region>.pooler.supabase.com:6543/postgres?pgbouncer=true&connection_limit=1"
DIRECT_URL="postgresql://postgres:<password>@db.<ref>.supabase.co:5432/postgres"
```
Ganti `<password>` dengan password yang Anda buat di langkah 2.

> **Kalau migrasi "stuck"/timeout:** jaringan Anda kemungkinan IPv4-only. Ganti `DIRECT_URL` ke string **Session Pooler** (port 5432, host `...pooler.supabase.com`).

---

## 2. Buat tabel & isi contoh produk (dari komputer Anda)

```bash
npm install
cp .env.example .env        # lalu edit .env: isi DATABASE_URL, DIRECT_URL, ADMIN_PASSWORD, ADMIN_SECRET, dan kunci Midtrans
npm run db:push             # membuat tabel di Supabase
npm run db:seed             # mengisi contoh produk (opsional)
npm run dev                 # uji lokal di http://localhost:3000
```
Cek di Supabase → **Table Editor**: tabel `Product` dan `Order` harus muncul.

---

## 3. Siapkan Midtrans (mulai mode Sandbox)

1. Daftar di https://dashboard.midtrans.com (mode **Sandbox** untuk uji coba).
2. **Settings → Access Keys**, salin **Server Key** & **Client Key**.
3. Isi ke `.env`:
   ```
   MIDTRANS_SERVER_KEY=SB-Mid-server-xxxx
   MIDTRANS_CLIENT_KEY=SB-Mid-client-xxxx
   MIDTRANS_IS_PRODUCTION=false
   NEXT_PUBLIC_MIDTRANS_CLIENT_KEY=SB-Mid-client-xxxx
   NEXT_PUBLIC_MIDTRANS_PRODUCTION=false
   ```
4. Go-live nanti: lengkapi verifikasi bisnis di Midtrans, ganti ke key Production, set kedua `*_PRODUCTION` jadi `true`.

---

## 4. Deploy ke Vercel

1. Push project ke **GitHub** (file `.env` tidak ikut — sudah diabaikan `.gitignore`).
2. https://vercel.com → **Add New → Project** → import repo. Framework: **Next.js** (otomatis).
3. **Environment Variables** — masukkan semuanya, sama seperti `.env`:
   - `DATABASE_URL`, `DIRECT_URL`
   - `ADMIN_PASSWORD`, `ADMIN_SECRET`
   - `MIDTRANS_SERVER_KEY`, `MIDTRANS_CLIENT_KEY`, `MIDTRANS_IS_PRODUCTION`
   - `NEXT_PUBLIC_MIDTRANS_CLIENT_KEY`, `NEXT_PUBLIC_MIDTRANS_PRODUCTION`
   - (opsional) `RESEND_API_KEY`, `EMAIL_FROM`
4. **Deploy.** Build menjalankan `prisma generate && next build` otomatis.

---

## 5. Dua sambungan terakhir setelah online

1. **Foto produk (Vercel Blob):** Vercel → tab **Storage → Create → Blob** → hubungkan ke project. Token `BLOB_READ_WRITE_TOKEN` ditambahkan otomatis. Lakukan **Redeploy** sekali. Setelah ini upload foto di `/admin` berfungsi.
2. **Webhook Midtrans:** dashboard Midtrans → **Settings → Configuration → Payment Notification URL**:
   ```
   https://NAMA-PROJECT-ANDA.vercel.app/api/midtrans/notification
   ```
   Ini yang membuat status pesanan & stok ter-update otomatis saat pembayaran masuk.

---

## 6. Uji end-to-end

1. Buka `https://NAMA-PROJECT-ANDA.vercel.app` → tambah produk ke keranjang → checkout.
2. Bayar memakai data **Sandbox** Midtrans (lihat dokumentasi Midtrans untuk nomor kartu/akun uji).
3. Cek `/admin/orders` — status pesanan harus berubah jadi **Lunas**, stok berkurang.
4. Cek `/lacak` — masukkan nomor pesanan + email untuk melihat statusnya.

---

## Halaman penting
- `/` — toko (storefront)
- `/admin` — kelola produk (login pakai `ADMIN_PASSWORD`)
- `/admin/orders` — kelola pesanan
- `/lacak` — pelanggan melacak pesanan

## Tips perawatan
- Setiap kali mengubah `schema.prisma`, jalankan lagi `npm run db:push` dari lokal (Vercel tidak migrasi saat build).
- Ganti `ADMIN_PASSWORD` secara berkala. Jangan pernah commit file `.env` ke GitHub.
