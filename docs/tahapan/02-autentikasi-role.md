# Tahap 02 — Autentikasi & Role

**Urutan:** 02 dari 09  
**Estimasi:** 3–5 jam  
**Tujuan:** User bisa login/logout, sistem membedakan 3 role, tiap role punya dashboard awal, dan profil dasar berfungsi.

← Sebelumnya: [01 — Persiapan](01-persiapan-setup.md) · [Indeks](../../README.md) · Berikutnya: [03 — Master Data](03-master-data.md) →

---

## Yang dihasilkan di tahap ini

- Auth manual: login, logout (tanpa register publik)
- 3 role: `admin`, `mahasiswa`, `pembimbing_industri`
- Kolom `role` + `is_active` di tabel `users`
- Middleware pembatas akses per role
- Redirect setelah login sesuai role
- Dashboard kosong (sambutan) per role
- Halaman profil: lihat data, ubah nama/email, ubah password
- Seeder 3 akun demo

---

## Menu yang aktif di tahap ini

```text
├── Login / Logout
├── Dashboard (bedakan sambutan per role)
└── Profil
    ├── Lihat / ubah data
    └── Ubah password
```

---

## Keputusan teknis

| Topik | Keputusan |
|-------|-----------|
| Penyimpanan role | Kolom `users.role` (string) — **disarankan** |
| Spatie Permission | Opsional nanti; **jangan** dipasang di tahap ini kecuali sudah paham |
| Register publik | Tidak ada — user dibuat admin/seeder |
| User nonaktif | `is_active = false` → **tolak login** |
| Layout | Satu `layouts/app.blade.php`; menu nanti beda per role |

---

## Langkah kerja (urut)

### 1. Migration: role & is_active

```bash
php artisan make:migration add_role_and_is_active_to_users_table --table=users
```

Isi migration:

```php
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('role', 50)->default('mahasiswa')->after('email');
        $table->boolean('is_active')->default(true)->after('role');
    });
}

public function down(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn(['role', 'is_active']);
    });
}
```

```bash
php artisan migrate
```

---

### 2. Update model `User`

Di `app/Models/User.php`:

- Tambahkan ke `$fillable`: `role`, `is_active`
- Pastikan `password` di-cast `hashed` (Laravel 11 biasanya sudah)
- Tambah helper:

```php
public function isAdmin(): bool
{
    return $this->role === 'admin';
}

public function isMahasiswa(): bool
{
    return $this->role === 'mahasiswa';
}

public function isPembimbingIndustri(): bool
{
    return $this->role === 'pembimbing_industri';
}
```

Nilai role yang diizinkan hanya:

```text
admin | mahasiswa | pembimbing_industri
```

---

### 3. Buat auth manual — LoginController

```bash
php artisan make:controller Auth/LoginController
```

Contoh isi inti:

```php
<?php

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class LoginController extends Controller
{
    public function showLoginForm()
    {
        return view('auth.login');
    }

    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
        ]);

        $user = \App\Models\User::where('email', $credentials['email'])->first();

        if ($user && ! $user->is_active) {
            return back()->withErrors([
                'email' => 'Akun nonaktif. Hubungi admin.',
            ])->onlyInput('email');
        }

        if (Auth::attempt($credentials, $request->boolean('remember'))) {
            $request->session()->regenerate();

            return redirect()->intended($this->redirectTo(Auth::user()));
        }

        return back()->withErrors([
            'email' => 'Email atau password salah.',
        ])->onlyInput('email');
    }

    public function logout(Request $request)
    {
        Auth::logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect()->route('login');
    }

    protected function redirectTo($user): string
    {
        return match ($user->role) {
            'admin' => route('admin.dashboard'),
            'mahasiswa' => route('mahasiswa.dashboard'),
            'pembimbing_industri' => route('pembimbing.dashboard'),
            default => '/',
        };
    }
}
```

---

### 4. View login

Buat `resources/views/auth/login.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Login — MagangTrack')

@section('content')
<div class="row justify-content-center">
    <div class="col-md-5">
        <div class="card shadow-sm">
            <div class="card-body p-4">
                <h1 class="h4 mb-3">Login MagangTrack</h1>

                <form method="POST" action="{{ route('login') }}">
                    @csrf
                    <div class="mb-3">
                        <label class="form-label">Email</label>
                        <input type="email" name="email" value="{{ old('email') }}"
                               class="form-control @error('email') is-invalid @enderror" required autofocus>
                        @error('email') <div class="invalid-feedback">{{ $message }}</div> @enderror
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Password</label>
                        <input type="password" name="password"
                               class="form-control @error('password') is-invalid @enderror" required>
                        @error('password') <div class="invalid-feedback">{{ $message }}</div> @enderror
                    </div>
                    <div class="form-check mb-3">
                        <input class="form-check-input" type="checkbox" name="remember" id="remember">
                        <label class="form-check-label" for="remember">Ingat saya</label>
                    </div>
                    <button class="btn btn-primary w-100" type="submit">Masuk</button>
                </form>
            </div>
        </div>
    </div>
</div>
@endsection
```

---

### 5. Middleware role

```bash
php artisan make:middleware EnsureUserHasRole
```

Contoh `app/Http/Middleware/EnsureUserHasRole.php`:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserHasRole
{
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        $user = $request->user();

        if (! $user || ! in_array($user->role, $roles, true)) {
            abort(403, 'Anda tidak memiliki akses ke halaman ini.');
        }

        return $next($request);
    }
}
```

Daftarkan alias di `bootstrap/app.php` (Laravel 11):

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'role' => \App\Http\Middleware\EnsureUserHasRole::class,
    ]);
})
```

---

### 6. Route auth + dashboard

Di `routes/web.php` (pola):

```php
use App\Http\Controllers\Auth\LoginController;
use App\Http\Controllers\Admin\DashboardController as AdminDashboardController;
use App\Http\Controllers\Mahasiswa\DashboardController as MahasiswaDashboardController;
use App\Http\Controllers\Pembimbing\DashboardController as PembimbingDashboardController;
use App\Http\Controllers\ProfileController;

Route::get('/', function () {
    if (! auth()->check()) {
        return redirect()->route('login');
    }

    return match (auth()->user()->role) {
        'admin' => redirect()->route('admin.dashboard'),
        'mahasiswa' => redirect()->route('mahasiswa.dashboard'),
        'pembimbing_industri' => redirect()->route('pembimbing.dashboard'),
        default => redirect()->route('login'),
    };
});

Route::middleware('guest')->group(function () {
    Route::get('/login', [LoginController::class, 'showLoginForm'])->name('login');
    Route::post('/login', [LoginController::class, 'login']);
});

Route::post('/logout', [LoginController::class, 'logout'])
    ->middleware('auth')
    ->name('logout');

Route::middleware('auth')->group(function () {
    Route::get('/profil', [ProfileController::class, 'edit'])->name('profil.edit');
    Route::put('/profil', [ProfileController::class, 'update'])->name('profil.update');
    Route::put('/profil/password', [ProfileController::class, 'updatePassword'])->name('profil.password');

    Route::middleware('role:admin')->prefix('admin')->name('admin.')->group(function () {
        Route::get('/dashboard', [AdminDashboardController::class, 'index'])->name('dashboard');
    });

    Route::middleware('role:mahasiswa')->prefix('mahasiswa')->name('mahasiswa.')->group(function () {
        Route::get('/dashboard', [MahasiswaDashboardController::class, 'index'])->name('dashboard');
    });

    Route::middleware('role:pembimbing_industri')->prefix('pembimbing')->name('pembimbing.')->group(function () {
        Route::get('/dashboard', [PembimbingDashboardController::class, 'index'])->name('dashboard');
    });
});
```

> Tip: untuk redirect `/` setelah login, cukup panggil ulang logic `match` role di closure agar tidak perlu method publik tambahan.

Buat 3 controller dashboard sederhana yang return view sambutan (nama + role).

---

### 7. Navigasi di layout

Update navbar:

- Jika guest: tombol Login
- Jika auth: link Dashboard (sesuai role), Profil, form Logout (`POST` + `@csrf`)

Contoh form logout:

```blade
<form action="{{ route('logout') }}" method="POST" class="d-inline">
    @csrf
    <button type="submit" class="btn btn-outline-light btn-sm">Logout</button>
</form>
```

---

### 8. Halaman profil

```bash
php artisan make:controller ProfileController
```

Fitur wajib:

1. Tampilkan nama, email, role (role read-only)
2. Ubah nama & email (email unik kecuali milik sendiri)
3. Ubah password: wajib `current_password`, `password` + `confirmed`, min 8 karakter

Validasi password update (inti):

```php
$request->validate([
    'current_password' => ['required', 'current_password'],
    'password' => ['required', 'confirmed', 'min:8'],
]);
```

---

### 9. Seeder user demo

Buat / edit `database/seeders/UserSeeder.php` atau `DatabaseSeeder`:

```php
use App\Models\User;
use Illuminate\Support\Facades\Hash;

User::updateOrCreate(
    ['email' => 'admin@demo.test'],
    [
        'name' => 'Admin Demo',
        'password' => Hash::make('password'),
        'role' => 'admin',
        'is_active' => true,
    ]
);

User::updateOrCreate(
    ['email' => 'mahasiswa@demo.test'],
    [
        'name' => 'Mahasiswa Demo',
        'password' => Hash::make('password'),
        'role' => 'mahasiswa',
        'is_active' => true,
    ]
);

User::updateOrCreate(
    ['email' => 'industri@demo.test'],
    [
        'name' => 'Pembimbing Industri Demo',
        'password' => Hash::make('password'),
        'role' => 'pembimbing_industri',
        'is_active' => true,
    ]
);
```

```bash
php artisan db:seed
```

| Email | Role | Password |
|-------|------|----------|
| admin@demo.test | admin | password |
| mahasiswa@demo.test | mahasiswa | password |
| industri@demo.test | pembimbing_industri | password |

---

### 10. Halaman 403 (opsional tapi bagus)

Buat `resources/views/errors/403.blade.php` yang extends layout, teks ramah: “Akses ditolak”.

---

## Isolasi yang wajib diuji

| Aksi | Hasil diharapkan |
|------|------------------|
| Login admin | Masuk `/admin/dashboard` |
| Login mahasiswa | Masuk `/mahasiswa/dashboard` |
| Login industri | Masuk `/pembimbing/dashboard` |
| Mahasiswa buka `/admin/dashboard` | 403 |
| User `is_active=false` login | Ditolak |
| Logout | Session hilang, kembali ke login |
| Sudah login buka `/login` | Redirect ke dashboard (middleware `guest`) |

---

## Checklist selesai

- [ ] Migration `role` + `is_active` jalan
- [ ] Helper role di model `User`
- [ ] Login / logout manual berfungsi
- [ ] User nonaktif tidak bisa login
- [ ] Middleware `role` terdaftar di `bootstrap/app.php`
- [ ] 3 dashboard route terproteksi
- [ ] Redirect setelah login sesuai role
- [ ] Profil: ubah data + ubah password
- [ ] 3 akun demo bisa login
- [ ] Mahasiswa tidak bisa akses URL admin

---

## Cara uji cepat

1. Login `admin@demo.test` / `password` → dashboard admin  
2. Logout  
3. Login `mahasiswa@demo.test` → dashboard mahasiswa  
4. Tempel URL `/admin/dashboard` → 403  
5. Ubah password di profil → logout → login dengan password baru  

---

## Kesalahan umum

1. **Lupa `session()->regenerate()` setelah login** — rawan session fixation.
2. **Logout pakai GET** — harus `POST` + CSRF.
3. **Middleware alias belum didaftarkan** — error “Target class [role] does not exist”.
4. **Password disimpan plain text di seeder** — wajib `Hash::make()` atau cast `hashed` + plain saat create via model dengan benar.
5. **Memasang Spatie di awal** — menambah kompleksitas; kolom `role` sudah cukup untuk project ini.

---

**Lanjut ke:** [Tahap 03 — Master Data](03-master-data.md)
