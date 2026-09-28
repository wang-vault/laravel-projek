# Anubis Store — versi Laravel

Toko online sederhana dengan **pembayaran transfer manual**, dibuat dengan **Laravel 12 + Blade + SQLite**.
Ini port dari sisi depan [Anubis Store](https://github.com/wang-vault/anubis) (aslinya Next.js + Supabase).

Pembeli tidak perlu daftar akun. Alurnya: pilih produk → isi form pesan → transfer sendiri →
tekan "Saya sudah transfer" → penjual cek mutasi rekening → status naik sampai selesai.

---

## Daftar isi

1. [Fitur](#1-fitur)
2. [Tutorial: menjalankan di komputer sendiri](#2-tutorial-menjalankan-di-komputer-sendiri)
3. [Tutorial: mencoba alur aplikasi](#3-tutorial-mencoba-alur-aplikasi)
4. [Struktur proyek](#4-struktur-proyek)
5. [Bagian yang perlu dijelaskan secara rinci](#5-bagian-yang-perlu-dijelaskan-secara-rinci)
6. [Cara deploy](#6-cara-deploy)
7. [Kekurangan dan batasan](#7-kekurangan-dan-batasan)
8. [Rute dan skema database](#8-rute-dan-skema-database)
9. [Kredit dan lisensi](#9-kredit-dan-lisensi)

---

## 1. Fitur

| Untuk | Fitur |
|---|---|
| Pembeli (tanpa login) | Lihat katalog + pencarian, detail produk, checkout, halaman pesanan lewat kode, tombol "Saya sudah transfer", lihat testimoni |
| Penjual (login) | CRUD produk, daftar semua pesanan + saringan, setujui / tolak klaim transfer, naikkan status, ubah dan hapus pesanan |
| Keamanan | Login berbasis session, middleware `auth`, CSRF, validasi server-side, pembatasan laju (rate limit) |
| Tampilan | Blade layout component `<x-layouts.app>`, CSS statis `public/css/anubis.css`, halaman error 403/404/419/429/500 buatan sendiri |

---

## 2. Tutorial: menjalankan di komputer sendiri

### Persyaratan

- **PHP 8.2 atau lebih baru** dengan ekstensi `pdo_sqlite`, `mbstring`, `openssl`, `ctype`, `json`, `tokenizer`, `xml`, `curl`, `fileinfo`, `bcmath`
- **Composer 2.x**
- Git

Cek cepat:

```bash
php -v
composer -V
php -m | grep -Ei "pdo_sqlite|mbstring|openssl|tokenizer|xml|curl|fileinfo|bcmath"
```

### Langkah instalasi

```bash
# 1. Ambil kode
git clone https://github.com/wang-vault/laravel-projek.git
cd laravel-projek

# 2. Pasang dependensi PHP
composer install

# 3. Siapkan file konfigurasi
cp .env.example .env
php artisan key:generate

# 4. Siapkan database SQLite (file kosong)
touch database/database.sqlite          # Windows PowerShell: New-Item database/database.sqlite

# 5. Buat tabel + isi data contoh
php artisan migrate --seed

# 6. Jalankan
php artisan serve
```

Buka <http://127.0.0.1:8000>.

Tidak perlu `npm install` atau `npm run build`, karena layout memakai file CSS statis
(`public/css/anubis.css`), bukan Vite.

### Akun penjual demo

```
Email    : admin@anubis.test
Password : password
```

> Akun ini hanya untuk demo. **Ganti passwordnya** sebelum aplikasi dipasang di internet (lihat bagian deploy).

### Menjalankan test

```bash
php artisan test
php artisan test --filter=OrderTest      # hanya satu kelompok
```

---

## 3. Tutorial: mencoba alur aplikasi

Urutan ini juga cocok dipakai sebagai skrip demo video.

**Sebagai pembeli (tanpa login)**

1. Buka `/` lalu klik salah satu produk, atau buka `/products` dan coba kotak pencarian.
2. Di halaman detail klik **Pesan Produk Ini**.
3. Isi nama, nomor WhatsApp (boleh `0812…`, `+62 812…`, dsb.), dan jumlah. Klik simpan.
4. Kamu diarahkan ke `/orders/ORD-YYYYMMDD-XXXXXX`. **Simpan kode itu**, karena itu satu-satunya kunci untuk membuka pesanan lagi.
5. Klik **Saya sudah transfer** (nomor referensi dan catatan boleh diisi). Status masih `PENDING`, karena menunggu penjual.

**Sebagai penjual**

1. Buka `/login`, masuk dengan akun demo.
2. Buka `/orders`. Pesanan yang diklaim akan terlihat.
3. Buka pesanannya, lalu **setujui** (status jadi `PAID`) atau **tolak** klaim (pembeli boleh klaim ulang).
4. Naikkan status: `PAID` → `PROCESSING` → `DONE`. Melompat atau mundur ditolak.
5. Buka `/testimoni`. Pesanan `DONE` muncul di sana dengan nama pembeli disamarkan ("Budi S.").
6. Coba juga CRUD produk di `/products` setelah login, dan coba hapus produk yang sudah pernah dipesan (akan ditolak dengan pesan ramah).

---

## 4. Struktur proyek

```
app/
  Http/Controllers/
    HomeController.php          beranda, about, downloader
    ProductController.php       CRUD produk + pencarian + pagination
    CheckoutController.php      form pesan + pembuatan pesanan
    OrderController.php         halaman pesanan, klaim, kelola, ubah status
    TestimonialController.php   testimoni dari pesanan DONE
    Auth/LoginController.php    login / logout + pembatasan percobaan
  Models/
    Product.php                 scope active(), formatted_price
    Order.php                   alur status, kode pesanan, normalisasi WA, samarkan nama
  Providers/AppServiceProvider.php   rate limiter + tampilan pagination
bootstrap/app.php               arah redirect untuk auth / guest
routes/web.php                  semua rute
database/
  migrations/                   products, orders (+ users/cache/jobs bawaan Laravel)
  seeders/                      UserSeeder, ProductSeeder, OrderSeeder
  factories/                    ProductFactory, OrderFactory
resources/views/
  components/layouts/app.blade.php   layout bersama
  products/ orders/ auth/ errors/    halaman-halaman
  checkout.blade.php  testimoni.blade.php  welcome.blade.php ...
public/css/anubis.css           seluruh tampilan
tests/Feature/                  AuthTest, ProductTest, OrderTest, TestimonialTest
preview-kit/                    perkakas sandbox (PHP-WASM), bukan bagian aplikasi
```

---

## 5. Bagian yang perlu dijelaskan secara rinci

Bagian ini disusun supaya bisa langsung dipakai sebagai kerangka **3 video penjelasan**.

### Video 1 — Gambaran umum, database, dan model (± 8–10 menit)

| Yang dijelaskan | File | Poin penting |
|---|---|---|
| Tujuan aplikasi dan demo alur | seluruh aplikasi | Tunjukkan demo pembeli lalu penjual (bagian 3) sebelum masuk kode |
| Struktur folder Laravel | bagian 4 | Pola MVC: route → controller → model → view |
| Migration `products` dan `orders` | `database/migrations/` | `restrictOnDelete` pada `product_id`, kolom **snapshot**, index `[order_status, created_at]` |
| Seeder dan factory | `database/seeders/`, `database/factories/` | Kenapa ada akun demo dan data contoh |
| Model `Product` | `app/Models/Product.php` | `$fillable`, `casts`, `scopeActive()`, accessor `formatted_price` |
| Model `Order` | `app/Models/Order.php` | Konstanta `FLOW` dan `TRANSITIONS`, `canTransitionTo()`, hook `saving` yang menghitung ulang `total_amount` |

### Video 2 — Alur kode inti: route, controller, view (± 10–12 menit)

| Yang dijelaskan | File | Poin penting |
|---|---|---|
| Rute dan middleware | `routes/web.php` | Rute publik vs `auth`; **urutan penting**: `/products/create` harus sebelum `/products/{product}` |
| Route model binding kustom | `routes/web.php` | `{order:order_code}` memakai kode, bukan `id` |
| Checkout | `CheckoutController.php` | Normalisasi WhatsApp (`0812…` → `62812…`), validasi, pembuatan snapshot |
| Kode pesanan | `Order::generateCode()` | Format `ORD-YYYYMMDD-XXXXXX`, zona WIB, alfabet tanpa `0/O/1/I/L`, pengecekan unik |
| Klaim dan verifikasi | `OrderController.php` | `claim()`, `updateStatus()`, `rejectClaim()`; kenapa klaim tidak langsung menaikkan status |
| Mesin status | `Order.php` | `PENDING → PAID → PROCESSING → DONE`, hanya maju satu langkah |
| CRUD produk | `ProductController.php` | Validasi, pencarian `LIKE`, `paginate()->withQueryString()`, penolakan hapus produk yang sudah dipesan |
| Testimoni dan privasi | `TestimonialController.php` | Query hanya mengambil 4 kolom aman; `maskBuyerName()` |
| Blade | `components/layouts/app.blade.php`, `orders/show.blade.php` | `<x-layouts.app>`, `@csrf`, `@method`, `@error`, `@forelse`, `@auth`, linimasa status |

### Video 3 — Keamanan, test, deploy, dan kekurangan (± 8–10 menit)

| Yang dijelaskan | File | Poin penting |
|---|---|---|
| Login | `LoginController.php`, `bootstrap/app.php` | `Auth::attempt`, `session()->regenerate()`, redirect `intended()`, batas 5 percobaan per email + IP |
| Rate limiting | `AppServiceProvider.php` | Empat limiter dan alasan kuncinya (IP untuk tamu, id penjual untuk penjual) |
| Perlindungan bawaan Laravel | seluruh form | CSRF, escaping Blade `{{ }}`, mass-assignment lewat `$fillable` |
| Test | `tests/Feature/` | Apa yang diuji; cara menjalankan `php artisan test` |
| Deploy | bagian 6 | Langkah produksi dan `.env` |
| Kekurangan | bagian 7 | Sampaikan dengan jujur, ini menunjukkan pemahaman |

---

## 6. Cara deploy

Pilih salah satu. **Opsi A** paling umum untuk tugas/portofolio, **Opsi B** paling fleksibel.

### Persiapan `.env` produksi (berlaku untuk semua opsi)

```dotenv
APP_NAME="Anubis Store"
APP_ENV=production
APP_DEBUG=false                # WAJIB false di produksi
APP_URL=https://domain-kamu.com
APP_KEY=                       # isi dengan: php artisan key:generate

DB_CONNECTION=sqlite           # atau mysql (lihat catatan di bawah)
LOG_LEVEL=error
```

`SESSION_DRIVER`, `CACHE_STORE`, dan `QUEUE_CONNECTION` di `.env.example` memakai `database`, jadi tabelnya
harus ada (dibuat oleh `php artisan migrate`).

### Opsi A — Shared hosting / cPanel

1. Di komputer sendiri jalankan `composer install --no-dev --optimize-autoloader`.
2. Upload seluruh proyek (termasuk `vendor/`) ke folder di luar `public_html`, misalnya `~/anubis`.
3. Isi `public_html` dengan isi folder `public/`, lalu ubah dua baris path di `public_html/index.php` supaya menunjuk ke `~/anubis/vendor/autoload.php` dan `~/anubis/bootstrap/app.php`.
   (Alternatif: arahkan document root domain ke `~/anubis/public` kalau panelnya mengizinkan.)
4. Buat `.env` di `~/anubis` (lihat di atas), lalu lewat Terminal cPanel atau SSH:

   ```bash
   php artisan key:generate --force
   touch database/database.sqlite
   php artisan migrate --force
   ```

5. Pastikan `storage/`, `bootstrap/cache/`, dan `database/` (beserta file `database.sqlite`) **bisa ditulis** oleh PHP (`chmod -R 775`).
6. Buat akun penjual (lihat "Membuat akun penjual produksi" di bawah).
7. Optimasi:

   ```bash
   php artisan config:cache && php artisan route:cache && php artisan view:cache
   ```

### Opsi B — VPS (Ubuntu + Nginx + PHP-FPM)

```bash
# Paket dasar
sudo apt update
sudo apt install nginx php8.3-fpm php8.3-cli php8.3-sqlite3 php8.3-mbstring \
     php8.3-xml php8.3-curl php8.3-bcmath unzip git composer

# Kode
cd /var/www
sudo git clone https://github.com/wang-vault/laravel-projek.git anubis
cd anubis
sudo composer install --no-dev --optimize-autoloader
sudo cp .env.example .env         # lalu edit sesuai bagian "Persiapan .env produksi"
sudo php artisan key:generate --force
sudo touch database/database.sqlite
sudo php artisan migrate --force

# Izin tulis
sudo chown -R www-data:www-data storage bootstrap/cache database
sudo chmod -R 775 storage bootstrap/cache database

# Cache
sudo php artisan config:cache && sudo php artisan route:cache && sudo php artisan view:cache
```

Konfigurasi Nginx (`/etc/nginx/sites-available/anubis`):

```nginx
server {
    listen 80;
    server_name domain-kamu.com;
    root /var/www/anubis/public;      # harus menunjuk ke folder public/

    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\.(?!well-known).* { deny all; }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/anubis /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d domain-kamu.com      # HTTPS
```

### Membuat akun penjual produksi

Jangan pakai `admin@anubis.test` / `password`. Buat akun sendiri lewat tinker:

```bash
php artisan tinker
>>> \App\Models\User::create(['name' => 'Nama Penjual', 'email' => 'kamu@domain.com', 'password' => \Illuminate\Support\Facades\Hash::make('PasswordKuat!')]);
```

Kalau tadi kamu menjalankan `migrate --seed`, hapus atau ganti akun demo dan data contohnya.
Untuk produksi sebaiknya cukup `php artisan migrate --force` (tanpa `--seed`).

### Kalau memakai Cloudflare atau reverse proxy

Rate limiter memakai alamat IP. Di belakang proxy, semua permintaan bisa terlihat berasal dari IP yang sama.
Atur *trusted proxies* di `bootstrap/app.php`:

```php
$middleware->trustProxies(at: '*');   // atau daftar IP proxy kamu
```

### Update setelah ada perubahan kode

```bash
git pull
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan config:cache && php artisan route:cache && php artisan view:cache
```

### Checklist sebelum publik

- [ ] `APP_DEBUG=false` dan `APP_ENV=production`
- [ ] `APP_KEY` sudah dibuat
- [ ] Password akun demo sudah diganti / dihapus
- [ ] Document root mengarah ke `public/`
- [ ] `.env` tidak ikut ter-upload ke repo publik
- [ ] HTTPS aktif
- [ ] `storage/` dan `database/` bisa ditulis, tapi database SQLite **tidak** bisa diakses lewat URL
- [ ] Ada cadangan (backup) file `database/database.sqlite`

---

## 7. Kekurangan dan batasan

### Fitur yang sengaja belum ada

- **Tidak ada gerbang pembayaran.** Pembeli transfer sendiri, penjual cek mutasi rekening secara manual. Tidak ada QRIS atau virtual account otomatis.
- **Tidak ada notifikasi.** Nomor WhatsApp disimpan tetapi aplikasi tidak mengirim pesan apa pun (WhatsApp, Telegram, email). Penjual harus rajin membuka `/orders`.
- **Tidak ada bukti transfer.** Pembeli hanya mengisi nomor referensi dan catatan teks, tidak bisa mengunggah foto.
- **Instruksi pembayaran statis.** Info rekening tidak bisa diubah dari aplikasi, hanya teks di halaman.
- **Tidak ada stok.** Tabel `products` tidak punya kolom stok, jadi produk bisa dipesan tanpa batas.
- **Pesanan tidak kedaluwarsa.** Pesanan yang tidak dibayar tetap `PENDING` selamanya.
- **Gambar produk hanya lewat URL** (`image_url`), tidak ada unggah file. Kalau situs asal gambarnya mati, gambar hilang.
- **Halaman `/downloader` hanya tiruan tampilan**, tidak berfungsi.
- **Tidak ada registrasi, lupa password, atau verifikasi email.** Hanya satu jenis akun: penjual.
- **Teks masih tertanam di kode.** Folder `lang/` tidak dipakai, dan `APP_LOCALE` bawaan masih `en`.

### Keterbatasan teknis dan keamanan

- **Kode pesanan = kunci akses.** Siapa pun yang tahu kodenya bisa membuka dan mengklaim pesanan itu. Kodenya acak (6 karakter dari 31 huruf/angka) dan ada rate limit, tetapi ini tetap bukan autentikasi sungguhan. Kode juga terlihat kalau pembeli membagikan tautannya.
- **Tidak ada otorisasi per pengguna** (`Policy`/`Gate`). Semua akun yang login dianggap penjual dengan hak penuh. Aman selama akun hanya dibuat manual, tapi tidak siap untuk banyak penjual atau peran.
- **Akun demo berpassword `password`.** Berbahaya kalau terbawa ke produksi. `.env.example` juga memakai `APP_DEBUG=true` secara bawaan.
- **Penjual bebas mengubah pesanan.** Lewat form ubah, harga satuan dan jumlah bisa diedit bahkan setelah lunas. Belum ada catatan riwayat perubahan (audit log).
- **SQLite.** Cukup untuk toko kecil, tetapi menulis bersamaan dalam jumlah banyak akan menjadi bottleneck, dan backup harus manual. Untuk trafik lebih besar, pindah ke MySQL/PostgreSQL.
- **Session, cache, dan queue memakai database.** Mudah disiapkan, tetapi lebih lambat dari Redis dan membebani database yang sama.
- **Rate limiter berbasis IP.** Pengguna di jaringan yang sama (kantor, kampus) berbagi batas, dan di belakang proxy IP-nya bisa salah kalau *trusted proxies* belum diatur.
- **Saringan status di `/orders` kemungkinan tidak bekerja.** Di `Order::scopeStatus()`, pengecekan memakai `in_array($status, self::TRANSITIONS, true)`, padahal isi `TRANSITIONS` berupa array sehingga tidak pernah cocok dengan string status. Kemungkinan besar seharusnya `array_keys(self::TRANSITIONS)`. *Ini temuan dari membaca kode, belum dijalankan; sebaiknya dites dulu sebelum diperbaiki.* Saringan `?payment=` memakai daftar yang benar sehingga tidak terkena masalah ini.
- **Klaim transfer dan pengecekan mutasi murni manual**, jadi rawan salah cek dan penipuan bukti transfer kalau penjual kurang teliti.
- **Tidak ada ekspor laporan** (CSV/PDF) dan tidak ada dashboard ringkasan penjualan.

### Catatan dokumentasi

- Versi README lama menyebut workflow CI (`.github/workflows/ci.yml`), tetapi filenya tidak ada di repositori ini. Tambahkan filenya atau abaikan klaim tersebut.
- Jumlah test yang tertulis di README lama (65) belum diverifikasi; jalankan `php artisan test` untuk angka yang sebenarnya.

---

## 8. Rute dan skema database

### Rute

| Method | URI | Akses | Controller |
|---|---|---|---|
| GET | `/`, `/about`, `/downloader`, `/testimoni` | publik | `HomeController`, `TestimonialController` |
| GET | `/products`, `/products/{product}` | publik (tamu hanya melihat produk aktif) | `ProductController` |
| GET / POST | `/checkout/{product}` | publik, POST kena `throttle:order-create` | `CheckoutController` |
| GET | `/orders/{kode}` | publik lewat kode | `OrderController@show` |
| POST | `/orders/{kode}/claim` | publik, `throttle:order-claim` | `OrderController@claim` |
| GET / POST | `/login` | tamu | `LoginController` |
| POST | `/logout` | penjual | `LoginController@destroy` |
| GET / POST / PUT / DELETE | `/products/create`, `/products`, `/products/{id}/edit`, `/products/{id}` | penjual, `throttle:product-write` | `ProductController` |
| GET | `/orders`, `/orders/{kode}/edit` | penjual | `OrderController` |
| POST / PUT / DELETE | `/orders/{kode}/status`, `/reject-claim`, `/orders/{kode}` | penjual, `throttle:order-status` | `OrderController` |
| GET | `/up` | health check bawaan Laravel | — |

### Batas laju

| Limiter | Batas | Kunci |
|---|---|---|
| `login` (di controller) | 5 / menit | email + IP |
| `product-write` | 20 / menit | id penjual |
| `order-create` | 10 / menit | IP |
| `order-claim` | 5 / menit | IP |
| `order-status` | 30 / menit | id penjual |

### Tabel `products`

| Kolom | Tipe | Aturan |
|---|---|---|
| `id` | bigint | primary key |
| `name` | string | wajib, 2–120 karakter |
| `description` | text | opsional, maks. 2000 |
| `price` | unsigned bigint | Rupiah penuh, Rp1.000 – Rp100.000.000 |
| `image_url` | string | opsional, harus URL |
| `is_active` | boolean | default `true` |
| `created_at`, `updated_at` | timestamp | otomatis |

### Tabel `orders` (ringkas)

| Kolom | Keterangan |
|---|---|
| `order_code` | unik, `ORD-YYYYMMDD-XXXXXX` |
| `product_id` | FK → `products`, `ON DELETE RESTRICT` |
| `product_name_snapshot`, `unit_price_snapshot` | nama dan harga dibekukan saat pesan |
| `quantity`, `total_amount` | 1–20; total = harga satuan × jumlah (dihitung ulang oleh model) |
| `payment_status` | `PENDING` / `PAID` |
| `order_status` | `PENDING` → `PAID` → `PROCESSING` → `DONE` |
| `manual_claim_*` | klaim pembeli (waktu, catatan, referensi) |
| `manual_review_*` | keputusan penjual (`APPROVED` / `REJECTED`, catatan, waktu) |
| `buyer_*_snapshot` | nama, WhatsApp (format `62…`), email opsional |
| `paid_at` | terisi saat status jadi `PAID` |

### Alur status

```
PENDING ──setujui──> PAID ──> PROCESSING ──> DONE
   │
   └─ pembeli klaim "sudah transfer" → status TETAP PENDING sampai penjual memutuskan
```

---

## 9. Kredit dan lisensi

- Desain dan skema produk: [wang-vault/anubis](https://github.com/wang-vault/anubis)
- Bentuk proyek: [qwerti1945/dasar_laravel](https://github.com/qwerti1945/dasar_laravel)
- Framework: [Laravel](https://laravel.com) (MIT)

Lisensi proyek ini: **MIT** (lihat file `LICENSE`).
