# lmsconceptbook
Writen in Indonesian

---

## 1. Arsitektur Umum

* **Stack utama**: Laravel 11 LTS, PHP 8.3, MariaDB 10.6, Composer, Git.
* **Frontend**: Blade + Bootstrap Admin (template bawaan atau AdminLTE agar cepat).
* **Deployment**: Nginx di VPS Ubuntu 22.04, queue pakai database driver, scheduler via `cron`.

## 2. Modul Fungsional

### 2.1 Autentikasi dan Role

* Registrasi, login, lupa sandi.
* Role: admin, mentor, siswa.
  Middleware `role:{role}` mengamankan route.

### 2.2 Manajemen Kelas

* CRUD kelas: judul, deskripsi, harga, kategori, status publish.
* Relasi `kelas` ⇄ `mentor` (users).

### 2.3 Materi Video

* CRUD materi di setiap kelas.
* Field inti: judul, deskripsi singkat, `youtube_id`, urutan tayang.
* Validasi otomatis agar hanya link YouTube yang diterima.

### 2.4 Pembelian Kelas

* Siswa klik tombol Beli.
* Backend membuat record `purchases` status `pending` lalu meminta Snap token ke Midtrans.
* Frontend redirect ke Snap pop-up.

### 2.5 Monitoring dan Manajemen Pembayaran

* Endpoint webhook `/midtrans/callback`.
* Signature verification sebelum proses.
* Update `purchases.status` menjadi `settlement`, `capture`, `expire`, atau `failure`.
* Retry manual via tombol Cek Status di halaman admin jika webhook gagal.
* Hanya status sukses yang memberi akses ke kelas (tabel `class_user_access`).

### 2.6 Progress Belajar

* Tabel `progress`: user, video, is\_completed, completed\_at.
* Update setiap kali siswa menekan tombol Selesai atau mencapai 90 persen durasi video (opsional via JavaScript YouTube API).

### 2.7 Dashboard

* **Admin**: statistik pengguna, omzet, transaksi harian.
* **Mentor**: daftar kelas, jumlah siswa, progress rata-rata.
* **Siswa**: kelas dimiliki, progres per kelas, materi berikutnya.

### 2.8 Email Notifikasi

* Event listener `PaymentSucceeded` dan `UserRegistered`.
* Mailer queue agar respons tetap cepat.

### 2.9 Sertifikat (opsional)

* Generator PDF dengan DomPDF.
* Disediakan endpoint download jika seluruh materi selesai.

## 3. Desain Database (skema singkat)

```plaintext
users
  id PK
  name
  email UNIQUE
  password
  role ENUM(admin,mentor,student)
  email_verified_at
  created_at
  updated_at

classes
  id PK
  mentor_id FK→users.id
  title
  description
  price DECIMAL(12,2)
  is_free TINYINT
  status ENUM(draft,publish)
  created_at
  updated_at

videos
  id PK
  class_id FK→classes.id
  title
  youtube_id
  description
  order_index INT
  created_at
  updated_at

purchases
  id PK
  user_id FK→users.id
  class_id FK→classes.id
  midtrans_order_id
  amount DECIMAL(12,2)
  status ENUM(pending,settlement,capture,expire,failure)
  payment_type
  paid_at
  created_at
  updated_at

progress
  id PK
  user_id FK→users.id
  video_id FK→videos.id
  is_completed TINYINT
  completed_at
  created_at
  updated_at

midtrans_logs
  id PK
  payload JSON
  received_at
  processed TINYINT
```

## 4. Rute dan Controller Kunci (ringkas)

| HTTP | Endpoint                  | Controller\@method           | Middleware              |
| ---- | ------------------------- | ---------------------------- | ----------------------- |
| GET  | /kelas                    | ClassController\@index       | auth                    |
| GET  | /kelas/create             | ClassController\@create      | auth,role\:mentor       |
| POST | /kelas                    | ClassController\@store       | auth,role\:mentor       |
| GET  | /kelas/{id}               | ClassController\@show        | auth                    |
| POST | /kelas/{id}/beli          | PurchaseController\@store    | auth,role\:student      |
| POST | /midtrans/callback        | MidtransController\@callback | none (verified via sig) |
| POST | /progress/{video}/selesai | ProgressController\@complete | auth,role\:student      |

## 5. Alur Pembayaran

1. Siswa tekan Beli Kelas → POST `/kelas/{id}/beli`.
2. Server:

   * Buat `purchases` status `pending`.
   * Hit Snap API → respon `token` dan `redirect_url`.
3. Frontend buka Snap pop-up.
4. Midtrans proses pembayaran → panggil webhook.
5. `MidtransController@callback`:

   * Verifikasi signature.
   * Update `purchases.status`.
   * Jika sukses → insert ke `class_user_access`.
6. Siswa otomatis dapat akses materi.

## 6. Keamanan dan Logging

* Sanctum/Passport untuk API token jika perlu mobile.
* Rate limiting di webhook.
* Semua request sensitif dicatat ke `midtrans_logs` dan Laravel log.

## 7. Testing dan Deployment

* PHPUnit untuk unit test model dan service Midtrans.
* Pest atau Laravel Dusk untuk test end-to-end.
* CI/CD sederhana pakai GitHub Actions: lint, test, deploy ke VPS via SSH.

## 8. Estimasi Biaya

Salary Rp. 100k/jam (6 tahun pengalaman), berdasarkan standar salary Fullstack Web Developer yang berpengalaman lebih dari **5 tahun**.

| No | Komponen                               | Estimasi Jam | Biaya (100k/jam)|
| -- | -------------------------------------- | ------------ | --------------- |
| 1  | Setup Laravel Project dan Autentikasi  | 4            | Rp400.000       |
| 2  | Role dan Hak Akses                     | 5            | Rp500.000       |
| 3  | Manajemen Kelas                        | 7            | Rp700.000       |
| 4  | Modul Materi dan Embed YouTube         | 7            | Rp700.000       |
| 5  | Sistem Pembelian Kelas                 | 6            | Rp600.000       |
| 6  | Integrasi Midtrans Snap API            | 9            | Rp900.000       |
| 7  | Monitoring dan Manajemen Pembayaran    | 14           | Rp1.400.000     |
| 8  | Progress Belajar Siswa                 | 8            | Rp800.000       |
| 9  | Dashboard Dinamis                      | 8            | Rp800.000       |
| 10 | Email Notifikasi                       | 4            | Rp400.000       |
| 11 | Testing, Debugging, Bug Fixes          | 6            | Rp600.000       |
| 12 | Dokumentasi, Panduan Admin, Deployment | 4            | Rp400.000       |
|    | **Total Waktu**                        | **92 jam**   | **Rp9.200.000** |

---
