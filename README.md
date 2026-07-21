# MagangTrack

Website monitoring kegiatan magang mahasiswa berbasis **Laravel**.

Mahasiswa mencatat log harian, pembimbing industri menyetujui, lalu penilaian dan laporan digabung dalam satu sistem. Dokumentasi ini disusun **per tahap** agar bisa dikerjakan berurutan dari nol sampai siap demo.

---

## Ringkasan singkat

| Pertanyaan | Jawaban |
|------------|---------|
| Apa ini? | Sistem monitoring magang (log, dokumen, nilai, laporan) |
| Framework? | Laravel 11 + Blade |
| UI? | Bootstrap 5 via CDN (tanpa npm, tanpa Tailwind) |
| Auth? | Manual (controller + session), tanpa Laravel UI / Breeze |
| Database? | MySQL 8 |
| Export? | PhpSpreadsheet (Excel) + DomPDF (PDF) |
| Ada berapa role? | 3 — Admin, Mahasiswa, Pembimbing Industri |

---

## Stack resmi

| Bagian | Teknologi | Catatan |
|--------|-----------|---------|
| Backend | Laravel 11 | PHP 8.2+ |
| View | Blade | Extends layout bersama |
| UI | Bootstrap 5.3 (CDN) | **Jangan** pakai Tailwind / npm / Vite build |
| Auth | Manual | `LoginController` + `Auth::attempt()` |
| Role | Kolom `users.role` | Nilai: `admin`, `mahasiswa`, `pembimbing_industri` |
| Database | MySQL 8 | Nama DB contoh: `magangtrack` |
| File upload | Storage disk `local` (private) | Download lewat controller terproteksi |
| Export Excel | `phpoffice/phpspreadsheet` | Tanpa Maatwebsite Excel |
| Export PDF | `barryvdh/laravel-dompdf` | View Blade → PDF |

---

## Role pengguna

| Role | Kode di DB | Fungsi utama |
|------|------------|--------------|
| Admin | `admin` | Kelola user, master data, penempatan magang, monitoring global, laporan |
| Mahasiswa | `mahasiswa` | Isi log kegiatan, upload dokumen, lihat nilai & status magang |
| Pembimbing Industri | `pembimbing_industri` | Approve/reject log, unduh dokumen bimbingan, beri nilai |

Tidak ada role dosen. Penilaian hanya dari pembimbing industri.

### Hak akses ringkas

| Fitur | Admin | Mahasiswa | Pembimbing Industri |
|-------|:-----:|:---------:|:-------------------:|
| Master Data (user, perusahaan, periode) | CRUD | — | — |
| Penempatan magang | CRUD + status | Lihat milik sendiri | Lihat bimbingan |
| Log kegiatan | Lihat semua | CRUD milik sendiri | Approve / reject |
| Dokumen | Lihat / bantu kelola | Upload milik sendiri | Lihat / unduh bimbingan |
| Penilaian (aspek) | CRUD aspek | — | — |
| Input nilai | Unlock (opsional) | Lihat hasil | Input + finalisasi |
| Laporan & export | Ya | Terbatas (milik sendiri) | Terbatas (bimbingan) |
| Dashboard statistik | Global | Personal | Antrian bimbingan |

---

## Alur bisnis end-to-end

```text
1. Admin buat user, tempat magang, periode
2. Admin tempatkan mahasiswa + assign pembimbing industri (status active)
3. Mahasiswa isi log harian → submit
4. Pembimbing industri approve / reject (reject wajib catatan)
5. Mahasiswa upload dokumen (proposal, laporan, dll.)
6. Pembimbing industri input nilai per aspek → finalisasi
7. Mahasiswa lihat nilai akhir
8. Admin / pihak terkait lihat dashboard & export laporan
```

---

## Menu aplikasi (hasil akhir)

```text
MagangTrack
├── Login / Logout
├── Dashboard                 ← beda isi per role
├── Profil
│   ├── Lihat / ubah data
│   └── Ubah password
├── Master Data               ← Admin
│   ├── User
│   ├── Tempat Magang
│   └── Periode Magang
├── Data Magang               ← penempatan internship
├── Log Kegiatan
│   ├── Daftar / form log     ← Mahasiswa
│   └── Antrian approval      ← Pembimbing Industri
├── Dokumen
├── Penilaian
│   ├── Aspek penilaian       ← Admin
│   ├── Input nilai           ← Pembimbing Industri
│   └── Hasil nilai           ← Mahasiswa (read-only)
└── Laporan
    ├── Rekap log
    ├── Rekap penilaian
    ├── Export Excel
    └── Export PDF
```

---

## Cara mengerjakan (wajib berurutan)

Kerjakan **tahap 01 → 09**. Jangan loncat. Centang checklist di akhir tiap file sebelum lanjut.

| Tahap | Fokus | File |
|------:|-------|------|
| 01 | Persiapan & setup Laravel + layout Bootstrap CDN | [01-persiapan-setup.md](docs/tahapan/01-persiapan-setup.md) |
| 02 | Login manual, role, middleware, dashboard kosong, profil | [02-autentikasi-role.md](docs/tahapan/02-autentikasi-role.md) |
| 03 | Master data: user, perusahaan, periode | [03-master-data.md](docs/tahapan/03-master-data.md) |
| 04 | Penempatan magang + policy isolasi data | [04-data-magang.md](docs/tahapan/04-data-magang.md) |
| 05 | Log kegiatan + approval (modul inti) | [05-log-kegiatan.md](docs/tahapan/05-log-kegiatan.md) |
| 06 | Upload & download dokumen | [06-dokumen.md](docs/tahapan/06-dokumen.md) |
| 07 | Penilaian berbobot + kunci final | [07-penilaian.md](docs/tahapan/07-penilaian.md) |
| 08 | Dashboard, laporan, export Excel/PDF, notifikasi | [08-dashboard-laporan.md](docs/tahapan/08-dashboard-laporan.md) |
| 09 | Polish UI, hardening, seeder demo, testing | [09-polish-testing.md](docs/tahapan/09-polish-testing.md) |

Setiap file tahap berisi:
- tujuan & hasil yang diharapkan
- menu yang aktif di tahap itu
- skema data / field
- langkah kerja berurutan (dengan contoh kode)
- checklist selesai
- cara uji cepat
- kesalahan umum & tips

---

## Model data (peta singkat)

```text
users
  ├── role: admin | mahasiswa | pembimbing_industri
  └── is_active

companies                  ← tempat magang
internship_periods         ← periode magang

internships                ← penempatan
  ├── student_id → users
  ├── company_id → companies
  ├── period_id → internship_periods
  ├── industrial_supervisor_id → users
  └── status: draft | active | completed | cancelled

activity_logs              ← log harian
  ├── internship_id
  ├── status: draft | submitted | approved | rejected
  └── reviewer_id, review_note, reviewed_at

documents                  ← file dokumen magang
  ├── internship_id
  ├── type, title, file_path, original_name, mime, size

assessment_aspects         ← master aspek + bobot
assessments                ← satu penilaian per internship (ideal)
assessment_scores          ← skor per aspek

notifications              ← opsional (tahap 08)
```

Diagram relasi lebih detail ada di tahap 04 dan 07.

---

## Akun demo (setelah seeder)

| Email | Password | Role |
|-------|----------|------|
| `admin@demo.test` | `password` | Admin |
| `mahasiswa@demo.test` | `password` | Mahasiswa |
| `industri@demo.test` | `password` | Pembimbing Industri |

Di tahap 09, seeder diperluas (lebih dari 1 mahasiswa, dll.) untuk demo yang lebih kaya.

---

## Install cepat (setelah project selesai dibuat di tahap 01)

```bash
composer install
cp .env.example .env
php artisan key:generate
# isi DB_* di .env, buat database magangtrack di MySQL
php artisan migrate --seed
php artisan serve
```

Buka: [http://127.0.0.1:8000](http://127.0.0.1:8000)

**Tidak ada** `npm install` / `npm run build`. Bootstrap dari CDN.

Package export (dipasang di tahap 08):

```bash
composer require phpoffice/phpspreadsheet
composer require barryvdh/laravel-dompdf
```

---

## Aturan pengerjaan

1. Ikuti urutan tahap 01 → 09.
2. Satu tahap selesai (checklist hijau) baru lanjut.
3. Commit per fitur / per tahap, jangan menumpuk di akhir.
4. Prioritas: fitur wajib dulu, bonus belakangan (lihat tahap 09).
5. Jangan ganti stack tanpa alasan kuat (tetap Bootstrap CDN, auth manual, PhpSpreadsheet + DomPDF).
6. Isolasi data antar role wajib diuji di setiap modul.

---

## Mulai dari sini

👉 **[Tahap 01 — Persiapan & Setup Project](docs/tahapan/01-persiapan-setup.md)**
