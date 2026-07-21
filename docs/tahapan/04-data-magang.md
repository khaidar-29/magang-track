# Tahap 04 — Data Magang (Penempatan)

**Urutan:** 04 dari 09  
**Tujuan:** Menghubungkan mahasiswa ke tempat magang, periode, dan pembimbing industri; mengatur status penempatan; isolasi data per role.

← Sebelumnya: [03 — Master Data](03-master-data.md) · [Indeks](../../README.md) · Berikutnya: [05 — Log Kegiatan](05-log-kegiatan.md) →

---

## Yang dihasilkan di tahap ini

- Tabel & model `Internship` (penempatan magang)
- Admin bisa create / edit / ubah status penempatan
- Assign **pembimbing industri** (bukan dosen)
- Status: `draft` → `active` → `completed` / `cancelled`
- Policy isolasi data
- Mahasiswa & pembimbing hanya melihat data yang relevan
- Seeder minimal 1 penempatan aktif untuk demo

Ini fondasi untuk log, dokumen, dan penilaian.

---

## Menu yang aktif di tahap ini

```text
├── Data Magang
│   ├── Daftar penempatan
│   ├── Tambah / edit penempatan     ← Admin
│   ├── Detail penempatan
│   └── Ubah status                  ← Admin
```

---

## Konsep data

Satu record `internships` artinya:

> Mahasiswa X magang di Perusahaan Y pada Periode Z, dibimbing oleh Pembimbing Industri A.

```text
users (mahasiswa)
      └── internships
            ├── companies
            ├── internship_periods
            └── users (pembimbing_industri)
```

### Field `internships`

| Field | Tipe | Keterangan |
|-------|------|------------|
| student_id | FK → users | Harus role `mahasiswa` |
| company_id | FK → companies | Tempat magang (aktif) |
| period_id | FK → internship_periods | Periode |
| industrial_supervisor_id | FK → users | Harus role `pembimbing_industri` |
| status | string | `draft` / `active` / `completed` / `cancelled` |
| started_at | date, nullable | Opsional; default bisa ikut period start |
| ended_at | date, nullable | Diisi saat `completed` (opsional) |
| notes | text, nullable | Catatan admin |
| timestamps | | |

**Tidak ada** `lecturer_id` / dosen.

---

## Mesin status

```text
        ┌──────────┐
        │  draft   │
        └────┬─────┘
             │ aktifkan
             ▼
        ┌──────────┐
   ┌────│  active  │────┐
   │    └──────────┘    │
   │ selesai            │ batalkan
   ▼                    ▼
┌───────────┐     ┌───────────┐
│ completed │     │ cancelled │
└───────────┘     └───────────┘
```

| Dari | Ke | Siapa |
|------|----|-------|
| draft | active | Admin |
| draft | cancelled | Admin |
| active | completed | Admin |
| active | cancelled | Admin |
| completed / cancelled | — | Tidak diubah lagi (kecuali admin “buka ulang” — opsional, boleh dilewati) |

Aturan bisnis:

1. Mahasiswa sebaiknya hanya punya **1 penempatan `active`** pada satu waktu (validasi di `store`/`update`/aktifkan).
2. Pembimbing industri hanya melihat internship yang `industrial_supervisor_id = auth()->id()`.
3. Mahasiswa hanya melihat `student_id = auth()->id()`.
4. Admin melihat semua.
5. Log & dokumen (tahap berikutnya) hanya untuk internship berstatus `active` (atau `completed` untuk lihat arsip — putuskan konsisten; disarankan: **buat log hanya jika `active`**).

---

## Keputusan teknis

| Topik | Keputusan |
|-------|-----------|
| Policy | `InternshipPolicy` untuk view/update |
| Controller | Bisa 1 `InternshipController` multi-role, atau pisah Admin vs lainnya |
| Dropdown form | Hanya user/company/period yang relevan & aktif |
| Validasi role FK | Pastikan student benar-benar mahasiswa, supervisor benar-benar pembimbing |

---

## Langkah kerja (urut)

### 1. Migration + model

```bash
php artisan make:model Internship -m
php artisan make:policy InternshipPolicy --model=Internship
```

Migration:

```php
Schema::create('internships', function (Blueprint $table) {
    $table->id();
    $table->foreignId('student_id')->constrained('users')->cascadeOnDelete();
    $table->foreignId('company_id')->constrained()->cascadeOnDelete();
    $table->foreignId('period_id')->constrained('internship_periods')->cascadeOnDelete();
    $table->foreignId('industrial_supervisor_id')->constrained('users')->cascadeOnDelete();
    $table->string('status', 20)->default('draft');
    $table->date('started_at')->nullable();
    $table->date('ended_at')->nullable();
    $table->text('notes')->nullable();
    $table->timestamps();

    $table->index(['student_id', 'status']);
    $table->index(['industrial_supervisor_id', 'status']);
});
```

Model `Internship`: `$fillable`, casts tanggal, konstanta status opsional:

```php
public const STATUS_DRAFT = 'draft';
public const STATUS_ACTIVE = 'active';
public const STATUS_COMPLETED = 'completed';
public const STATUS_CANCELLED = 'cancelled';
```

---

### 2. Relasi model

**Internship**

```php
public function student() { return $this->belongsTo(User::class, 'student_id'); }
public function company() { return $this->belongsTo(Company::class); }
public function period() { return $this->belongsTo(InternshipPeriod::class, 'period_id'); }
public function industrialSupervisor() { return $this->belongsTo(User::class, 'industrial_supervisor_id'); }
```

**User**

```php
public function internshipsAsStudent()
{
    return $this->hasMany(Internship::class, 'student_id');
}

public function internshipsAsSupervisor()
{
    return $this->hasMany(Internship::class, 'industrial_supervisor_id');
}
```

**Company / InternshipPeriod**

```php
public function internships()
{
    return $this->hasMany(Internship::class);
}
```

---

### 3. Policy

Daftarkan policy (Laravel 11 biasanya auto-discover jika naming benar).

Contoh kemampuan:

| Ability | Admin | Mahasiswa pemilik | Pembimbing assign |
|---------|:-----:|:-----------------:|:-----------------:|
| viewAny | ✓ | ✓ (scoped) | ✓ (scoped) |
| view | ✓ | ✓ milik | ✓ bimbingan |
| create | ✓ | — | — |
| update | ✓ | — | — |
| updateStatus | ✓ | — | — |

Di controller `index`, **jangan** andalkan policy saja untuk filter list — query harus di-scope:

```php
$internships = match (auth()->user()->role) {
    'admin' => Internship::with([...])->latest(),
    'mahasiswa' => Internship::with([...])->where('student_id', auth()->id())->latest(),
    'pembimbing_industri' => Internship::with([...])
        ->where('industrial_supervisor_id', auth()->id())->latest(),
    default => Internship::query()->whereRaw('1=0'),
};
```

Di `show` / `edit`: `$this->authorize('view', $internship)`.

---

### 4. Form penempatan (Admin)

Field form:

- Mahasiswa (select user role mahasiswa, aktif)
- Tempat magang (company aktif)
- Periode
- Pembimbing industri (user role pembimbing_industri, aktif)
- Status awal (`draft` atau `active`)
- Catatan (opsional)

Validasi inti:

```php
'student_id' => ['required', 'exists:users,id'],
'company_id' => ['required', 'exists:companies,id'],
'period_id' => ['required', 'exists:internship_periods,id'],
'industrial_supervisor_id' => ['required', 'exists:users,id'],
'status' => ['required', 'in:draft,active'],
'notes' => ['nullable', 'string'],
```

Setelah validasi request, cek role di controller/service:

```php
$student = User::findOrFail($data['student_id']);
abort_unless($student->role === 'mahasiswa' && $student->is_active, 422);

$supervisor = User::findOrFail($data['industrial_supervisor_id']);
abort_unless($supervisor->role === 'pembimbing_industri' && $supervisor->is_active, 422);
```

Cek 1 active per mahasiswa:

```php
$existsActive = Internship::where('student_id', $student->id)
    ->where('status', 'active')
    ->when($internship ?? null, fn ($q) => $q->where('id', '!=', $internship->id))
    ->exists();

if ($data['status'] === 'active' && $existsActive) {
    return back()->withInput()->with('error', 'Mahasiswa sudah punya penempatan aktif.');
}
```

---

### 5. Halaman daftar & detail

**Index**

- Kolom: mahasiswa, perusahaan, periode, pembimbing, status, aksi
- Filter opsional: status, periode, perusahaan
- Eager load: `student`, `company`, `period`, `industrialSupervisor` (hindari N+1)

**Detail**

Tampilkan semua ringkasan. Halaman ini nanti dipakai lagi di log/nilai (link cepat).

---

### 6. Aksi ubah status

Route contoh:

```text
PATCH /internships/{internship}/status
```

Body: `status` = `active` | `completed` | `cancelled`

Saat `completed`, set `ended_at` = hari ini (jika kosong).  
Validasi transisi sesuai tabel mesin status di atas.

Konfirmasi di UI (Bootstrap modal atau `confirm()` sederhana) sebelum batalkan/selesaikan.

---

### 7. Proteksi hapus master data (update dari tahap 03)

Di `CompanyController@destroy` / `InternshipPeriodController@destroy`:

```php
if ($company->internships()->exists()) {
    return back()->with('error', 'Masih dipakai penempatan. Nonaktifkan saja.');
}
```

---

### 8. Seeder penempatan contoh

Hubungkan:

- `mahasiswa@demo.test`
- 1 company dari seeder tahap 03
- 1 period dari seeder tahap 03
- `industri@demo.test`
- status: `active`

```bash
php artisan make:seeder InternshipSeeder
php artisan db:seed --class=InternshipSeeder
```

---

## Route saran

```text
GET    /internships                  → index (semua role, scoped)
GET    /internships/create           → admin
POST   /internships                  → admin
GET    /internships/{id}             → show (authorized)
GET    /internships/{id}/edit        → admin
PUT    /internships/{id}             → admin
PATCH  /internships/{id}/status      → admin
```

Bisa di-prefix `/admin` untuk create/edit, sementara index/show di path bersama — pilih satu pola dan konsisten.

---

## Checklist selesai

- [ ] Migration `internships` jalan
- [ ] Relasi model lengkap
- [ ] Admin bisa tambah & edit penempatan
- [ ] Validasi role student & supervisor
- [ ] Validasi satu penempatan `active` per mahasiswa
- [ ] Ubah status sesuai mesin status
- [ ] Policy / authorize di show
- [ ] Index ter-scope per role
- [ ] Mahasiswa & pembimbing lain tidak melihat data yang bukan miliknya
- [ ] Seeder 1 internship aktif siap untuk tahap 05

---

## Cara uji cepat

1. Login admin → buat penempatan mahasiswa demo + industri demo → status `active`  
2. Login mahasiswa → melihat penempatannya di Data Magang  
3. Login industri → mahasiswa tersebut muncul di daftar bimbingan  
4. Buat user pembimbing lain → login → mahasiswa demo **tidak** muncul  
5. Coba aktifkan penempatan kedua untuk mahasiswa yang sama → ditolak  

---

## Kesalahan umum

1. **Dropdown menampilkan semua user** — filter by role + `is_active`.
2. **Lupa eager load** — halaman index lambat / N+1.
3. **Policy ada tapi index tidak di-scope** — user masih “lihat semua id” lalu 403 di detail; lebih baik list sudah terfilter.
4. **cascadeOnDelete terlalu agresif** — jika takut data hilang, ganti `restrictOnDelete` untuk FK penting.
5. **Status string bebas** — selalu validasi `in:draft,active,completed,cancelled`.

---

**Lanjut ke:** [Tahap 05 — Log Kegiatan](05-log-kegiatan.md)
