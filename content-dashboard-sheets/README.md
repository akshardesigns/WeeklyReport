# Content Production Dashboard — Backend Google Sheets

Dashboard yang sama seperti versi HTML kamu sebelumnya, tapi sekarang datanya
disimpan **live di Google Sheets** lewat Next.js API Route + service account.
Tidak ada lagi data yang hilang saat refresh, karena browser cuma jadi
"tampilan" — sumber data sesungguhnya ada di spreadsheet.

Fitur tambahan: **filter mingguan** (tombol "Semua Minggu" / "Filter: Mingguan"
+ navigasi ‹ minggu › di atas tabel "Daftar Brief"). Saat filter aktif, KPI,
grafik status, donut pilar, dan grafik ketepatan waktu semuanya otomatis ikut
menghitung ulang berdasarkan data minggu yang dipilih saja.

---

## 1. Struktur Project

```
content-dashboard-sheets/
├─ lib/
│  └─ googleSheets.js        # koneksi service account + CRUD ke Sheets
├─ pages/
│  ├─ _app.js
│  ├─ index.js                # UI dashboard (React)
│  └─ api/
│     └─ briefs/
│        ├─ index.js          # GET (list) & POST (tambah)
│        └─ [id].js           # PUT (update) & DELETE (hapus)
├─ styles/
│  └─ globals.css
├─ .env.local.example
├─ package.json
└─ README.md
```

Kolom di Google Sheets (tab default bernama **`input`**), urutan HARUS persis:

| A  | B         | C     | D        | E     | F      | G           |
|----|-----------|-------|----------|-------|--------|-------------|
| id | tglMasuk  | pilar | platform | brief | status | tglSelesai  |

Baris header ini akan **dibuat otomatis** oleh aplikasi kalau sheet-nya masih
kosong, jadi kamu tidak perlu mengetik manual — cukup buat sheet baru yang
kosong.

---

## 2. Bikin Service Account (sekali saja)

1. Buka [Google Cloud Console](https://console.cloud.google.com/), buat
   project baru (atau pakai yang sudah ada).
2. Di menu **APIs & Services → Library**, cari **Google Sheets API**, klik
   **Enable**.
3. Di menu **APIs & Services → Credentials**, klik **Create Credentials →
   Service Account**. Kasih nama bebas, misalnya `brief-dashboard`.
4. Setelah service account dibuat, buka tab **Keys** di service account
   tersebut → **Add Key → Create new key → JSON**. File JSON akan otomatis
   terdownload — **simpan baik-baik, jangan diupload ke publik/GitHub**.
5. Dari file JSON itu kamu akan butuh dua nilai:
   - `client_email` → ini untuk `GOOGLE_SERVICE_ACCOUNT_EMAIL`
   - `private_key` → ini untuk `GOOGLE_PRIVATE_KEY`

## 3. Siapkan Google Sheet-nya

1. Buat Google Sheet baru (atau pakai yang sudah ada), rename salah satu tab
   jadi **`input`** (atau nama lain, nanti diisi di `.env.local`).
2. Kosongkan tab tersebut (biar header dibuat otomatis oleh aplikasi), atau
   isi manual baris pertama sesuai tabel kolom di atas.
3. **Share** sheet tersebut ke email `client_email` dari file JSON tadi
   (misalnya `brief-dashboard@nama-project.iam.gserviceaccount.com`) dengan
   akses **Editor**. Tanpa langkah share ini, service account tidak akan bisa
   baca/tulis ke sheet-nya.
4. Ambil `GOOGLE_SHEET_ID` dari URL sheet:
   `https://docs.google.com/spreadsheets/d/`**`INI_ID_NYA`**`/edit`

## 4. Konfigurasi Environment

```bash
cp .env.local.example .env.local
```

Isi `.env.local`:

```env
GOOGLE_SERVICE_ACCOUNT_EMAIL=brief-dashboard@nama-project.iam.gserviceaccount.com
GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMIIEvQ...banyak-baris...\n-----END PRIVATE KEY-----\n"
GOOGLE_SHEET_ID=1AbCdEfGhIjKlMnOpQrStUvWxYz
GOOGLE_SHEET_NAME=input
```

> Tips: kalau copy-paste `private_key` dari file JSON, biasanya sudah dalam
> bentuk satu baris dengan `\n` literal di dalamnya — itu sudah benar, tinggal
> tempel apa adanya dan bungkus dengan tanda kutip dua seperti contoh di atas.

## 5. Jalankan

```bash
npm install
npm run dev
```

Buka `http://localhost:3000`. Tambah / edit / hapus brief lewat form seperti
biasa — setiap aksi langsung mengubah data di Google Sheets, dan kalau kamu
refresh halaman, data akan dimuat ulang dari sheet (bukan dari localStorage
browser).

## 6. Deploy (opsional, misalnya ke Vercel)

1. Push project ini ke GitHub (pastikan `.env.local` **tidak** ikut ter-commit
   — sudah ada di `.gitignore`).
2. Import repo ke [Vercel](https://vercel.com/).
3. Di **Project Settings → Environment Variables**, isi 4 variabel yang sama
   seperti di `.env.local` (`GOOGLE_SERVICE_ACCOUNT_EMAIL`,
   `GOOGLE_PRIVATE_KEY`, `GOOGLE_SHEET_ID`, `GOOGLE_SHEET_NAME`).
4. Deploy.

---

## Cara Kerja Singkat

- **GET `/api/briefs`** — baca semua baris dari sheet, tampil di dashboard.
- **POST `/api/briefs`** — tambah baris baru (append) di sheet.
- **PUT `/api/briefs/:id`** — cari baris dengan `id` tersebut, replace isinya.
- **DELETE `/api/briefs/:id`** — cari baris dengan `id` tersebut, hapus baris
  itu dari sheet (pakai `batchUpdate` → `deleteDimension`, bukan cuma
  dikosongkan).
- Filter mingguan murni di sisi frontend (`pages/index.js`): tidak menambah
  request baru ke Sheets, cukup memfilter data yang sudah dimuat berdasarkan
  rentang Senin–Minggu yang dipilih.

## Kalau mau ubah kolom / logic KPI

- Urutan & nama kolom ada di `HEADERS` (`lib/googleSheets.js`) — kalau diubah,
  pastikan urutan di Sheet juga diubah sama persis.
- Aturan "On Time / Late" (maks 2 hari untuk Ads & Carousel, 3 hari untuk
  Video & Lainnya) ada di fungsi `maxDays()` / `kpiFor()` di `pages/index.js`.
