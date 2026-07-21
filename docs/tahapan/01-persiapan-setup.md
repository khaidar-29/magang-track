# Tahap 01 — Persiapan & Setup Project

**Urutan:** 01 dari 09  
**Tujuan:** Project Laravel siap jalan di lokal, repo rapi, database terkoneksi.

← [Kembali ke indeks](../../README.md) · Berikutnya: [02 — Autentikasi & Role](02-autentikasi-role.md) →

---

## Yang dihasilkan di tahap ini

- Project Laravel 11 terinstall
- Laravel Breeze (Blade + Tailwind) terpasang
- Database MySQL terkoneksi
- File `.env.example` siap dibagikan
- Struktur folder `docs/` sudah ada
- Bisa menjalankan `php artisan serve` tanpa error

---

## Prasyarat di komputer

- PHP 8.2+
- Composer
- Node.js + NPM
- MySQL
- Git

Cek cepat:

```bash
php -v
composer -V
node -v
mysql --version
git --version
```

---

## Langkah kerja (urut)

### 1. Buat project Laravel

```bash
composer create-project laravel/laravel .
```

Atau jika folder sudah terisi dokumen:

```bash
composer create-project laravel/laravel magangtrack
cd magangtrack
```

### 2. Pasang Breeze (Blade)

```bash
composer require laravel/breeze --dev
php artisan breeze:install blade
npm install
npm run build
```

### 3. Setup environment

```bash
cp .env.example .env
php artisan key:generate
```

Isi bagian database di `.env`:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=magangtrack
DB_USERNAME=root
DB_PASSWORD=
```

Buat database `magangtrack` di MySQL.

### 4. Migrasi awal

```bash
php artisan migrate
```

### 5. Jalankan aplikasi

```bash
php artisan serve
```

Buka `http://127.0.0.1:8000` — halaman default Breeze harus muncul.

### 6. Siapkan Git

```bash
git init
# pastikan .env ada di .gitignore (default Laravel sudah)
git add .
git commit -m "chore: initial laravel + breeze setup"
```

### 7. Rapikan dokumentasi

Pastikan struktur seperti ini:

```
project-magang/
├── README.md
└── docs/
    └── tahapan/
        ├── 01-persiapan-setup.md
        ├── 02-autentikasi-role.md
        └── ...
```

---

## Checklist selesai

- [ ] `php artisan serve` berjalan
- [ ] Halaman login/register Breeze muncul
- [ ] Koneksi database sukses (`migrate` tanpa error)
- [ ] `.env` tidak ter-commit
- [ ] Commit awal sudah dibuat

---

## Catatan

Belum perlu buat fitur bisnis di tahap ini.  
Fokusnya hanya fondasi project agar tahap berikutnya lancar.

---

**Lanjut ke:** [Tahap 02 — Autentikasi & Role](02-autentikasi-role.md)
