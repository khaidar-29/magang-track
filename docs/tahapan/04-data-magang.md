# Tahap 04 — Data Magang (Penempatan)

**Urutan:** 04 dari 09  
**Tujuan:** Menghubungkan mahasiswa ke tempat magang, periode, dan pembimbing.

← Sebelumnya: [03 — Master Data](03-master-data.md) · [Indeks](../../README.md) · Berikutnya: [05 — Log Kegiatan](05-log-kegiatan.md) →

---

## Yang dihasilkan di tahap ini

- Tabel penempatan magang (`internships`)
- Admin bisa menempatkan mahasiswa
- Assign pembimbing industri & dosen
- Status magang: `draft` → `active` → `completed` / `cancelled`
- Tiap role melihat data sesuai haknya

---

## Menu yang aktif di tahap ini

```
├── Data Magang
│   ├── Daftar penempatan
│   ├── Tambah / edit penempatan     ← Admin
│   └── Detail penempatan
```

---

## Konsep data

Satu record `internships` artinya:

> Mahasiswa X magang di Perusahaan Y pada Periode Z, dibimbing oleh Industri A dan Dosen B.

### Field utama `internships`

| Field | Keterangan |
|-------|------------|
| student_id | User role mahasiswa |
| company_id | Tempat magang |
| period_id | Periode |
| industrial_supervisor_id | User role pembimbing industri |
| lecturer_id | User role dosen |
| status | draft / active / completed / cancelled |
| started_at | Opsional |
| ended_at | Opsional |
| notes | Opsional |

---

## Aturan bisnis tahap ini

1. Mahasiswa aktif sebaiknya hanya punya **1 penempatan aktif** per waktu.
2. Pembimbing industri hanya melihat mahasiswa yang di-assign kepadanya.
3. Dosen hanya melihat mahasiswa bimbingannya.
4. Mahasiswa hanya melihat data magangnya sendiri.
5. Ubah status harus jelas (misalnya dari `active` ke `completed`).

---

## Langkah kerja (urut)

### 1. Migration + model `Internship`

```bash
php artisan make:model Internship -m
```

Tambahkan foreign key ke `users`, `companies`, `internship_periods`.

### 2. Relasi model

- `User` hasMany / hasOne `Internship` (sebagai student)
- `Company` hasMany `Internship`
- `InternshipPeriod` hasMany `Internship`
- `Internship` belongsTo student, company, period, industrialSupervisor, lecturer

### 3. Controller & policy

```bash
php artisan make:controller InternshipController --resource
php artisan make:policy InternshipPolicy --model=Internship
```

Policy mengatur siapa boleh lihat/ubah record tertentu.

### 4. Form penempatan (Admin)

Form create/edit berisi:
- Pilih mahasiswa
- Pilih tempat magang
- Pilih periode
- Pilih pembimbing industri
- Pilih dosen
- Pilih status awal (`draft` atau `active`)

Validasi: semua field wajib relevan harus terisi.

### 5. Halaman daftar & detail

**Admin:** lihat semua  
**Pembimbing / Dosen:** filter hanya bimbingan mereka  
**Mahasiswa:** hanya miliknya

Detail menampilkan ringkasan penempatan (dipakai lagi di tahap log/nilai).

### 6. Aksi ubah status

Tombol sederhana:
- Aktifkan
- Selesaikan
- Batalkan

Catat waktu selesai jika status `completed`.

### 7. Seeder penempatan contoh

Buat minimal 1 internship aktif yang menghubungkan:
- `mahasiswa@demo.test`
- 1 company
- 1 period
- `industri@demo.test`
- `dosen@demo.test`

---

## Checklist selesai

- [ ] Admin bisa menambah penempatan magang
- [ ] Relasi mahasiswa–perusahaan–pembimbing tersimpan benar
- [ ] Status bisa diubah
- [ ] Isolasi data per role berjalan
- [ ] Ada minimal 1 data demo aktif untuk tahap berikutnya

---

## Cara uji cepat

1. Login admin → buat penempatan untuk mahasiswa demo  
2. Login mahasiswa → melihat detail magangnya  
3. Login pembimbing industri → mahasiswa tersebut muncul di daftar bimbingan  
4. Login dosen lain (jika ada) → mahasiswa itu **tidak** muncul  

---

**Lanjut ke:** [Tahap 05 — Log Kegiatan](05-log-kegiatan.md)
