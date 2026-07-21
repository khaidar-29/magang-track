# Tahap 02 — Autentikasi & Role

**Urutan:** 02 dari 09  
**Tujuan:** User bisa login, sistem membedakan role, dan tiap role punya layout/dashboard awal.

← Sebelumnya: [01 — Persiapan](01-persiapan-setup.md) · [Indeks](../../README.md) · Berikutnya: [03 — Master Data](03-master-data.md) →

---

## Yang dihasilkan di tahap ini

- 4 role: `admin`, `mahasiswa`, `pembimbing_industri`, `dosen`
- Middleware pembatas akses per role
- Redirect setelah login sesuai role
- Halaman profil dasar (lihat & ubah data singkat)
- Seeder user dummy untuk testing

---

## Menu yang aktif di tahap ini

```
├── Login / Logout
├── Dashboard (kosong dulu, beda sambutan per role)
└── Profil
    ├── Lihat profil
    └── Ubah password
```

---

## Langkah kerja (urut)

### 1. Tentukan penyimpanan role

Pilihan sederhana (disarankan di awal):

- Tambah kolom `role` di tabel `users`  
  nilai: `admin | mahasiswa | pembimbing_industri | dosen`

Atau pakai package:

```bash
composer require spatie/laravel-permission
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

> Untuk belajar, kolom `role` di `users` sudah cukup. Spatie bisa ditambah nanti jika perlu permission lebih detail.

### 2. Buat migration kolom role (jika pakai cara sederhana)

```bash
php artisan make:migration add_role_to_users_table --table=users
```

Isi: kolom `role` string, default `mahasiswa`.

Jalankan:

```bash
php artisan migrate
```

### 3. Update model `User`

- Tambahkan `role` ke `$fillable`
- Buat helper sederhana, contoh: `isAdmin()`, `isMahasiswa()`, dll.

### 4. Buat middleware role

```bash
php artisan make:middleware EnsureUserHasRole
```

Logic singkat:
- Ambil role yang diizinkan dari route
- Jika user tidak cocok → abort `403`

Daftarkan middleware di Laravel (bootstrap/app.php atau alias middleware sesuai versi).

### 5. Atur redirect setelah login

Di `LoginResponse` / `AuthenticatedSessionController` (Breeze):
- `admin` → `/admin/dashboard`
- `mahasiswa` → `/mahasiswa/dashboard`
- `pembimbing_industri` → `/pembimbing/dashboard`
- `dosen` → `/dosen/dashboard`

### 6. Buat route & controller dashboard per role

Contoh route:

```text
/admin/dashboard
/mahasiswa/dashboard
/pembimbing/dashboard
/dosen/dashboard
```

Tiap halaman cukup tampilkan nama user + role dulu.

### 7. Proteksi route

Contoh:
- Route admin hanya `role:admin`
- Route mahasiswa hanya `role:mahasiswa`
- dst.

Uji: login sebagai mahasiswa, buka URL admin → harus ditolak.

### 8. Halaman profil dasar

- Lihat nama, email, role
- Form ubah nama / email
- Form ubah password (pakai fitur Breeze jika sudah ada)

### 9. Seeder user dummy

Buat 4 akun:

| Email | Role | Password |
|-------|------|----------|
| admin@demo.test | admin | password |
| mahasiswa@demo.test | mahasiswa | password |
| industri@demo.test | pembimbing_industri | password |
| dosen@demo.test | dosen | password |

```bash
php artisan db:seed
```

---

## Checklist selesai

- [ ] 4 akun dummy bisa login
- [ ] Tiap role masuk ke dashboard masing-masing
- [ ] Mahasiswa tidak bisa akses halaman admin
- [ ] Logout berfungsi
- [ ] Profil bisa dibuka dan password bisa diganti

---

## Cara uji cepat

1. Login `admin@demo.test` → masuk dashboard admin  
2. Logout  
3. Login `mahasiswa@demo.test` → masuk dashboard mahasiswa  
4. Tempel URL `/admin/dashboard` → muncul 403 / redirect  

---

**Lanjut ke:** [Tahap 03 — Master Data](03-master-data.md)
