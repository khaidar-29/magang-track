# MagangTrack

Website monitoring kegiatan magang mahasiswa berbasis **Laravel**.

Mahasiswa mencatat log harian, pembimbing menyetujui, lalu penilaian dan laporan digabung dalam satu sistem.

---

## Stack

| Bagian | Teknologi |
|--------|-----------|
| Backend | Laravel 11 |
| Auth | Manual (controller + Blade) |
| UI | Blade + Bootstrap 5 (CDN) |
| Database | MySQL 8 |
| Export | PhpSpreadsheet + DomPDF |

---

## Role pengguna

| Role | Fungsi singkat |
|------|----------------|
| Admin | Kelola user, master data, monitoring global |
| Mahasiswa | Isi log, upload dokumen, lihat nilai |
| Pembimbing Industri | Approve log, beri nilai industri |
| Dosen Pembimbing | Monitor progress, beri nilai akademik |

---

## Cara mengerjakan project ini

Kerjakan **berurutan dari tahap 01 → 09**.  
Jangan loncat tahap. Centang checklist di setiap README sebelum lanjut.

| Tahap | Fokus | File |
|------:|-------|------|
| 01 | Persiapan & setup project | [docs/tahapan/01-persiapan-setup.md](docs/tahapan/01-persiapan-setup.md) |
| 02 | Login, role, layout dasar | [docs/tahapan/02-autentikasi-role.md](docs/tahapan/02-autentikasi-role.md) |
| 03 | Master data | [docs/tahapan/03-master-data.md](docs/tahapan/03-master-data.md) |
| 04 | Penempatan magang | [docs/tahapan/04-data-magang.md](docs/tahapan/04-data-magang.md) |
| 05 | Log kegiatan & approval | [docs/tahapan/05-log-kegiatan.md](docs/tahapan/05-log-kegiatan.md) |
| 06 | Upload dokumen | [docs/tahapan/06-dokumen.md](docs/tahapan/06-dokumen.md) |
| 07 | Penilaian | [docs/tahapan/07-penilaian.md](docs/tahapan/07-penilaian.md) |
| 08 | Dashboard & laporan | [docs/tahapan/08-dashboard-laporan.md](docs/tahapan/08-dashboard-laporan.md) |
| 09 | Perapian, testing, jalankan stabil | [docs/tahapan/09-polish-testing.md](docs/tahapan/09-polish-testing.md) |

---

## Menu aplikasi (hasil akhir)

```
MagangTrack
├── Login / Logout
├── Dashboard
├── Profil
├── Master Data          ← Admin
│   ├── User
│   ├── Tempat Magang
│   └── Periode Magang
├── Data Magang
├── Log Kegiatan
├── Dokumen
├── Penilaian
└── Laporan
```

Detail tiap menu ada di README tahap yang terkait.

---

## Install cepat (setelah project dibuat di tahap 01)

```bash
composer install
cp .env.example .env
php artisan key:generate
# isi DB di .env
php artisan migrate --seed
php artisan serve
```

Buka: `http://127.0.0.1:8000`

---

## Aturan pengerjaan

1. Ikuti urutan tahap.
2. Satu tahap selesai dulu (checklist hijau), baru lanjut.
3. Commit per fitur, jangan menumpuk di akhir.
4. Prioritas: fitur wajib dulu, bonus belakangan.
