# Tahap 01 — Persiapan & Setup Project

**Urutan:** 01 dari 09  
**Estimasi:** 1–2 jam  
**Tujuan:** Project Laravel siap jalan di lokal, layout Bootstrap CDN siap dipakai, database terkoneksi, repo rapi.

← [Kembali ke indeks](../../README.md) · Berikutnya: [02 — Autentikasi & Role](02-autentikasi-role.md) →

---

## Yang dihasilkan di tahap ini

Setelah tahap ini selesai, kamu punya:

- Project Laravel 11 di folder kerja
- Layout Blade dasar memakai **Bootstrap 5 via CDN**
- Database MySQL `magangtrack` terkoneksi
- Migrasi default Laravel (tabel `users`, `sessions`, dll.) sudah jalan
- File `.env` lokal (tidak di-commit)
- Struktur folder `docs/` tetap ada
- `php artisan serve` berjalan tanpa error

**Belum** dibuat di tahap ini: login custom, role, CRUD bisnis. Itu mulai tahap 02.

---

## Prasyarat di komputer

| Tool | Minimal | Cek perintah |
|------|---------|--------------|
| PHP | 8.2+ | `php -v` |
| Composer | 2.x | `composer -V` |
| MySQL | 8.x | `mysql --version` |
| Git | apa saja | `git --version` |

**Tidak perlu:** Node.js, npm, Vite build, Tailwind, Laravel UI, Laravel Breeze.

Ekstensi PHP yang biasanya dibutuhkan Laravel: `pdo_mysql`, `mbstring`, `openssl`, `tokenizer`, `xml`, `ctype`, `json`, `fileinfo`, `bcmath`.

---

## Keputusan teknis tahap ini (wajib diikuti)

| Topik | Keputusan |
|-------|-----------|
| UI | Bootstrap 5.3 CDN |
| Build frontend | Tidak ada (`npm` dilarang untuk project ini) |
| Auth package | Belum dipasang; auth dibuat manual di tahap 02 |
| Nama database | `magangtrack` |

---

## Langkah kerja (urut)

### 1. Siapkan folder project

Ada dua skenario:

**A. Folder sudah berisi dokumentasi (seperti repo ini)**

Buat project Laravel di subfolder sementara, lalu pindahkan isinya — **atau** install Laravel di folder kosong lalu salin folder `docs/` + `README.md` ke dalamnya.

Cara yang paling aman jika folder sudah berisi file:

```bash
# dari parent folder
composer create-project laravel/laravel magangtrack-app
```

Lalu salin `README.md` dan `docs/` dari repo dokumentasi ke dalam `magangtrack-app`, lalu kerjakan di situ.

**B. Folder benar-benar kosong**

```bash
composer create-project laravel/laravel .
```

Pastikan setelah selesai struktur kira-kira seperti ini:

```text
magangtrack/
├── app/
├── bootstrap/
├── config/
├── database/
├── docs/                    ← dokumentasi tahapan
│   └── tahapan/
├── public/
├── resources/
│   └── views/
├── routes/
├── .env
├── .env.example
├── artisan
├── composer.json
└── README.md
```

---

### 2. Buat database MySQL

Via CLI:

```bash
mysql -u root -p -e "CREATE DATABASE magangtrack CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

Atau lewat phpMyAdmin / TablePlus / MySQL Workbench: buat database bernama `magangtrack`, charset `utf8mb4`.

---

### 3. Setup environment

```bash
cp .env.example .env
php artisan key:generate
```

Edit `.env` (bagian penting):

```env
APP_NAME=MagangTrack
APP_ENV=local
APP_DEBUG=true
APP_URL=http://127.0.0.1:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=magangtrack
DB_USERNAME=root
DB_PASSWORD=
```

Sesuaikan `DB_USERNAME` / `DB_PASSWORD` dengan komputer kamu.

**Jangan** commit file `.env`. Default `.gitignore` Laravel sudah mengabaikannya — pastikan tetap begitu.

---

### 4. Jalankan migrasi awal

```bash
php artisan migrate
```

Ini membuat tabel bawaan Laravel (`users`, `password_reset_tokens`, `sessions`, `jobs`, `cache`, dll.). Belum ada tabel bisnis MagangTrack.

Jika error koneksi:

| Error umum | Perbaikan |
|------------|-----------|
| `Access denied for user` | Cek username/password MySQL di `.env` |
| `Unknown database` | Buat dulu database `magangtrack` |
| `Connection refused` | Pastikan MySQL service jalan; cek `DB_HOST` / `DB_PORT` |
| Socket error di Mac | Coba `DB_HOST=127.0.0.1` (bukan `localhost`) |

---

### 5. Buat layout Bootstrap CDN

Buat file `resources/views/layouts/app.blade.php`:

```blade
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    <title>@yield('title', 'MagangTrack')</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark mb-4">
        <div class="container">
            <a class="navbar-brand" href="{{ url('/') }}">MagangTrack</a>
            <div class="d-flex gap-2">
                @auth
                    <span class="navbar-text text-white-50">{{ auth()->user()->name }}</span>
                    {{-- form logout dibuat di tahap 02 --}}
                @else
                    <a href="{{ url('/login') }}" class="btn btn-outline-light btn-sm">Login</a>
                @endauth
            </div>
        </div>
    </nav>

    <main class="container pb-5">
        @if (session('success'))
            <div class="alert alert-success">{{ session('success') }}</div>
        @endif
        @if (session('error'))
            <div class="alert alert-danger">{{ session('error') }}</div>
        @endif

        @yield('content')
    </main>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
    @stack('scripts')
</body>
</html>
```

Buat halaman uji `resources/views/welcome-magang.blade.php` (sementara):

```blade
@extends('layouts.app')

@section('title', 'Beranda — MagangTrack')

@section('content')
    <div class="p-4 bg-white border rounded">
        <h1 class="h3">MagangTrack</h1>
        <p class="text-muted mb-0">Layout Bootstrap CDN siap. Lanjut ke tahap 02 untuk autentikasi.</p>
    </div>
@endsection
```

Di `routes/web.php`, sementara:

```php
<?php

use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return view('welcome-magang');
});
```

> Jangan pasang Tailwind, Laravel UI, Laravel Breeze, atau jalankan `npm install`.

---

### 6. Jalankan aplikasi

```bash
php artisan serve
```

Buka [http://127.0.0.1:8000](http://127.0.0.1:8000).  
Harus terlihat navbar gelap + teks MagangTrack dengan style Bootstrap.

---

### 7. Siapkan Git

```bash
git status
# pastikan .env TIDAK muncul sebagai file yang akan di-commit
git add .
git commit -m "chore: initial laravel + bootstrap cdn layout"
```

Jika remote sudah ada:

```bash
git remote -v
# jika belum ada origin:
# git remote add origin https://github.com/USERNAME/REPO.git
# git push -u origin main
```

---

### 8. Rapikan dokumentasi

Pastikan folder docs ikut di repo:

```text
docs/tahapan/01-persiapan-setup.md
docs/tahapan/02-autentikasi-role.md
... (sampai 09)
README.md
```

---

## Struktur folder yang diharapkan setelah tahap 01

```text
resources/views/
├── layouts/
│   └── app.blade.php
└── welcome-magang.blade.php

routes/web.php          ← route `/` ke welcome-magang
.env                    ← lokal, tidak di-commit
```

---

## Checklist selesai

- [ ] PHP 8.2+, Composer, MySQL, Git tersedia
- [ ] Project Laravel 11 terinstall
- [ ] Database `magangtrack` dibuat
- [ ] `.env` terisi & `php artisan key:generate` sukses
- [ ] `php artisan migrate` sukses
- [ ] Layout `layouts/app.blade.php` pakai Bootstrap CDN
- [ ] Halaman beranda tampil dengan style Bootstrap
- [ ] Tidak ada Tailwind / npm / Laravel UI / Breeze
- [ ] `.env` tidak ter-commit
- [ ] Commit awal sudah dibuat

---

## Kesalahan umum

1. **Memasang Breeze “biar cepat”** — jangan. Auth dibuat manual di tahap 02 agar sesuai aturan project (Bootstrap CDN, tanpa npm).
2. **`composer create-project` gagal karena folder tidak kosong** — pakai subfolder lalu pindahkan docs, atau bersihkan folder selain docs/README.
3. **Bootstrap tidak muncul** — cek koneksi internet (CDN), cek typo URL CDN, pastikan view benar-benar `@extends('layouts.app')`.
4. **Migrate error padahal MySQL jalan** — pastikan extension `pdo_mysql` aktif (`php -m | grep pdo_mysql`).

---

## Catatan

Belum perlu buat fitur bisnis.  
Fokus tahap 01: fondasi project agar tahap berikutnya lancar.

---

**Lanjut ke:** [Tahap 02 — Autentikasi & Role](02-autentikasi-role.md)
