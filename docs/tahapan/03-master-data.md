# Tahap 03 — Master Data

**Urutan:** 03 dari 09  
**Tujuan:** Admin bisa mengelola data dasar yang dibutuhkan modul berikutnya.

← Sebelumnya: [02 — Auth & Role](02-autentikasi-role.md) · [Indeks](../../README.md) · Berikutnya: [04 — Data Magang](04-data-magang.md) →

---

## Yang dihasilkan di tahap ini

CRUD untuk:
- User (kelola akun + role + aktif/nonaktif)
- Tempat Magang (`companies`)
- Periode Magang (`internship_periods`)

Tanpa master data ini, modul penempatan & log tidak bisa jalan.

---

## Menu yang aktif di tahap ini

```
├── Master Data          ← Admin saja
│   ├── User
│   ├── Tempat Magang
│   └── Periode Magang
```

---

## Data yang dikelola

### A. User
| Field | Keterangan |
|-------|------------|
| name | Nama lengkap |
| email | Unik |
| password | Hash |
| role | admin / mahasiswa / pembimbing_industri / dosen |
| is_active | true/false |

### B. Tempat Magang
| Field | Keterangan |
|-------|------------|
| name | Nama instansi |
| address | Alamat |
| pic_name | Nama PIC |
| pic_contact | No. HP / email PIC |
| description | Opsional |

### C. Periode Magang
| Field | Keterangan |
|-------|------------|
| name | Contoh: Genap 2025/2026 |
| start_date | Tanggal mulai |
| end_date | Tanggal selesai |
| is_active | Periode sedang dipakai atau tidak |

---

## Langkah kerja (urut)

### 1. Buat migration + model

```bash
php artisan make:model Company -m
php artisan make:model InternshipPeriod -m
```

Untuk user: migration tambahan jika perlu `is_active`, `phone`, dll.

### 2. Definisikan relasi model

Contoh nanti dipakai tahap 04:
- `Company` hasMany `Internship`
- `InternshipPeriod` hasMany `Internship`

Sementara relasi boleh ditulis dulu atau ditunda sampai tahap 04.

### 3. Buat Form Request / validasi

Contoh aturan:
- email user unik
- `end_date` >= `start_date`
- field wajib tidak boleh kosong

### 4. Buat controller resource (Admin)

```bash
php artisan make:controller Admin/UserController --resource
php artisan make:controller Admin/CompanyController --resource
php artisan make:controller Admin/InternshipPeriodController --resource
```

### 5. Buat route admin

Prefix contoh: `/admin/...`  
Middleware: `auth` + `role:admin`

Route yang dibutuhkan per resource:
- `index` (daftar + search)
- `create` / `store`
- `edit` / `update`
- `destroy` (lebih aman soft delete)

### 6. Buat Blade CRUD

Untuk setiap resource, minimal:
- Tabel daftar + tombol tambah
- Form create
- Form edit
- Tombol hapus (dengan konfirmasi)
- Flash message sukses/gagal
- Empty state jika data kosong

### 7. Proteksi hapus

Jangan hard-delete sembarangan jika data sudah dipakai.  
Untuk tahap ini: soft delete atau cek relasi sebelum hapus.

### 8. Seeder data contoh

Isi 2–3 tempat magang dan 1 periode aktif agar siap uji di tahap 04.

---

## Checklist selesai

- [ ] Admin bisa CRUD user
- [ ] Admin bisa CRUD tempat magang
- [ ] Admin bisa CRUD periode magang
- [ ] Validasi form berjalan
- [ ] Role lain tidak bisa buka menu master data
- [ ] Search / pagination dasar ada di halaman index

---

## Cara uji cepat

1. Login admin  
2. Tambah tempat magang baru → muncul di daftar  
3. Edit periode → tanggal berubah  
4. Nonaktifkan 1 user → user itu tidak bisa dipakai (atau ditolak login, sesuai aturan yang kamu pilih)  
5. Login mahasiswa → URL `/admin/companies` ditolak  

---

**Lanjut ke:** [Tahap 04 — Data Magang](04-data-magang.md)
