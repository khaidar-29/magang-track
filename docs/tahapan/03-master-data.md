# Tahap 03 — Master Data

**Urutan:** 03 dari 09  
**Estimasi:** 4–6 jam  
**Tujuan:** Admin bisa mengelola data dasar yang dibutuhkan modul berikutnya: user, tempat magang, dan periode magang.

← Sebelumnya: [02 — Auth & Role](02-autentikasi-role.md) · [Indeks](../../README.md) · Berikutnya: [04 — Data Magang](04-data-magang.md) →

---

## Yang dihasilkan di tahap ini

CRUD lengkap (Admin saja) untuk:

1. **User** — akun + role + aktif/nonaktif
2. **Tempat Magang** (`companies`)
3. **Periode Magang** (`internship_periods`)

Tanpa master data ini, penempatan & log tidak bisa jalan.

---

## Menu yang aktif di tahap ini

```text
├── Master Data          ← Admin saja
│   ├── User
│   ├── Tempat Magang
│   └── Periode Magang
```

Tambahkan link menu ini di navbar **hanya jika** `auth()->user()->isAdmin()`.

---

## Data yang dikelola

### A. User (`users`)

| Field | Tipe | Keterangan |
|-------|------|------------|
| name | string | Nama lengkap |
| email | string, unique | Email login |
| password | string (hashed) | Wajib saat create; opsional saat edit |
| role | string | `admin` / `mahasiswa` / `pembimbing_industri` |
| is_active | boolean | Default `true`. `false` = tidak bisa login |

Aturan tambahan:

- Admin tidak boleh menonaktifkan / menghapus **akun sendiri**.
- Idealnya tidak menghapus user yang sudah punya relasi magang — nonaktifkan saja (`is_active = false`) atau pakai SoftDeletes.
- Saat edit: jika password dikosongkan, jangan ubah password lama.

### B. Tempat Magang (`companies`)

| Field | Tipe | Keterangan |
|-------|------|------------|
| name | string | Nama instansi |
| address | text | Alamat |
| pic_name | string | Nama PIC |
| pic_contact | string | No. HP / email PIC |
| description | text, nullable | Opsional |
| is_active | boolean | Default `true` |
| deleted_at | timestamp, nullable | Soft delete |

### C. Periode Magang (`internship_periods`)

| Field | Tipe | Keterangan |
|-------|------|------------|
| name | string | Contoh: “Magang Genap 2025/2026” |
| start_date | date | Tanggal mulai |
| end_date | date | Tanggal selesai (`>= start_date`) |
| is_active | boolean | Periode yang dipakai penempatan baru |
| deleted_at | timestamp, nullable | Soft delete |

---

## Keputusan teknis

| Topik | Keputusan |
|-------|-----------|
| Namespace controller | `App\Http\Controllers\Admin\...` |
| Prefix route | `/admin/...` + middleware `auth` + `role:admin` |
| Soft delete | Ya untuk `companies` & `internship_periods` |
| User delete | Soft delete **atau** nonaktifkan; jangan hapus keras jika sudah berelasi |
| Listing | Pagination (10–15 per halaman) + pencarian sederhana |
| Validasi | Form Request per resource (disarankan) |

---

## Langkah kerja (urut)

### 1. Migration + model Company

```bash
php artisan make:model Company -m
```

Migration (inti):

```php
Schema::create('companies', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->text('address');
    $table->string('pic_name');
    $table->string('pic_contact');
    $table->text('description')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();
});
```

Model: pakai trait `SoftDeletes`, `$fillable` sesuai field.

---

### 2. Migration + model InternshipPeriod

```bash
php artisan make:model InternshipPeriod -m
```

```php
Schema::create('internship_periods', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->date('start_date');
    $table->date('end_date');
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();
});
```

Validasi bisnis: `end_date` harus setelah atau sama dengan `start_date`.

---

### 3. Relasi model (persiapan tahap 04)

Di `Company` dan `InternshipPeriod`, siapkan:

```php
public function internships()
{
    return $this->hasMany(Internship::class); // model dibuat di tahap 04
}
```

Boleh dikomentari dulu sampai tahap 04 jika class belum ada — atau buat method setelah tahap 04.

Di `User`, nanti (tahap 04) akan ada relasi sebagai student / supervisor.

---

### 4. Form Request

Contoh:

```bash
php artisan make:request Admin/StoreUserRequest
php artisan make:request Admin/UpdateUserRequest
php artisan make:request Admin/StoreCompanyRequest
php artisan make:request Admin/UpdateCompanyRequest
php artisan make:request Admin/StoreInternshipPeriodRequest
php artisan make:request Admin/UpdateInternshipPeriodRequest
```

Contoh rule user (store):

```php
return [
    'name' => ['required', 'string', 'max:100'],
    'email' => ['required', 'email', 'max:150', 'unique:users,email'],
    'password' => ['required', 'string', 'min:8', 'confirmed'],
    'role' => ['required', 'in:admin,mahasiswa,pembimbing_industri'],
    'is_active' => ['sometimes', 'boolean'],
];
```

Update user: `email` unique ignore current id; `password` nullable.

Periode:

```php
'start_date' => ['required', 'date'],
'end_date' => ['required', 'date', 'after_or_equal:start_date'],
```

---

### 5. Controller resource (Admin)

```bash
php artisan make:controller Admin/UserController --resource
php artisan make:controller Admin/CompanyController --resource
php artisan make:controller Admin/InternshipPeriodController --resource
```

Pola tiap controller:

| Method | Fungsi |
|--------|--------|
| `index` | List + search + paginate |
| `create` | Form tambah |
| `store` | Simpan |
| `edit` | Form ubah |
| `update` | Update |
| `destroy` | Soft delete / nonaktifkan |

Contoh search user di `index`:

```php
$query = User::query()->latest();

if ($search = $request->string('q')->toString()) {
    $query->where(function ($q) use ($search) {
        $q->where('name', 'like', "%{$search}%")
          ->orWhere('email', 'like', "%{$search}%");
    });
}

if ($role = $request->string('role')->toString()) {
    $query->where('role', $role);
}

$users = $query->paginate(10)->withQueryString();
```

Proteksi hapus/nonaktif:

```php
if ($user->id === auth()->id()) {
    return back()->with('error', 'Tidak bisa menonaktifkan akun sendiri.');
}
```

Proteksi hapus company yang masih dipakai (nanti setelah tahap 04):

```php
if ($company->internships()->exists()) {
    return back()->with('error', 'Tidak bisa hapus: masih dipakai penempatan magang. Nonaktifkan saja.');
}
```

Untuk tahap 03 (belum ada internship), soft delete biasa sudah cukup; tambahkan cek relasi di tahap 04/09.

---

### 6. Route admin

Di dalam group `auth` + `role:admin` + prefix `admin`:

```php
Route::resource('users', UserController::class)->except(['show']);
Route::resource('companies', CompanyController::class)->except(['show']);
Route::resource('periods', InternshipPeriodController::class)->except(['show']);
```

URL contoh:

```text
/admin/users
/admin/companies
/admin/periods
```

---

### 7. Blade CRUD (Bootstrap)

Untuk tiap resource, buat minimal:

```text
resources/views/admin/users/index.blade.php
resources/views/admin/users/create.blade.php
resources/views/admin/users/edit.blade.php
```

(dan setara untuk `companies`, `periods`)

Pola UI:

- `index`: tabel + form search GET + tombol “Tambah” + pagination `{{ $items->links() }}`
- `create` / `edit`: form vertical Bootstrap, tampilkan `@error` per field
- Flash `success` / `error` sudah ada di layout tahap 01

Untuk pagination Bootstrap 5, di `AppServiceProvider::boot()`:

```php
use Illuminate\Pagination\Paginator;
Paginator::useBootstrapFive();
```

---

### 8. Seeder data contoh

Buat seeder:

```bash
php artisan make:seeder CompanySeeder
php artisan make:seeder InternshipPeriodSeeder
```

Contoh data:

**Companies**

| name | pic_name |
|------|----------|
| PT Teknologi Nusantara | Budi Santoso |
| CV Digital Kreatif | Siti Rahma |

**Periods**

| name | start_date | end_date |
|------|------------|----------|
| Magang Genap 2025/2026 | 2026-01-06 | 2026-06-30 |

Panggil dari `DatabaseSeeder` bersama `UserSeeder`.

```bash
php artisan db:seed
```

---

## Struktur file yang diharapkan

```text
app/Http/Controllers/Admin/
├── UserController.php
├── CompanyController.php
└── InternshipPeriodController.php

app/Http/Requests/Admin/
├── StoreUserRequest.php
├── UpdateUserRequest.php
└── ...

app/Models/
├── Company.php
└── InternshipPeriod.php

resources/views/admin/
├── users/
├── companies/
└── periods/
```

---

## Checklist selesai

- [ ] Tabel `companies` & `internship_periods` ter-migrate (soft deletes)
- [ ] CRUD User (create dengan role, edit, nonaktifkan) jalan
- [ ] Tidak bisa menonaktifkan diri sendiri
- [ ] CRUD Tempat Magang jalan
- [ ] CRUD Periode Magang jalan (`end_date >= start_date`)
- [ ] Route hanya bisa diakses admin
- [ ] Search + pagination di index
- [ ] Validasi error tampil di form
- [ ] Seeder company + period tersedia
- [ ] Menu Master Data hanya muncul untuk admin

---

## Cara uji cepat

1. Login admin → buka `/admin/users` → tambah user mahasiswa baru  
2. Edit user → kosongkan password → simpan → password lama tetap bisa dipakai  
3. Nonaktifkan user → coba login sebagai user itu → ditolak  
4. Tambah 1 perusahaan + 1 periode  
5. Login mahasiswa → buka `/admin/companies` → harus 403  

---

## Kesalahan umum

1. **Password ter-overwrite jadi kosong saat edit** — hanya update password jika field terisi.
2. **Lupa `Paginator::useBootstrapFive()`** — link pagination jelek / tidak Bootstrap.
3. **Role bebas diketik** — harus `in:admin,mahasiswa,pembimbing_industri`.
4. **Soft delete tanpa trait di model** — record tetap “hilang” dari query default? Pastikan `use SoftDeletes` di model.
5. **Mahasiswa masih melihat menu Master Data** — bungkus menu dengan `@if(auth()->user()->isAdmin())`.

---

**Lanjut ke:** [Tahap 04 — Data Magang](04-data-magang.md)
