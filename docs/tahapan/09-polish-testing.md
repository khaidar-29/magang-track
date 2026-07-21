# Tahap 09 — Polish, Testing & Stabilisasi

**Urutan:** 09 dari 09  
**Tujuan:** Website MagangTrack dirapikan, diuji, seeder demo lengkap, siap ditampilkan / dipakai demo end-to-end.

← Sebelumnya: [08 — Dashboard & Laporan](08-dashboard-laporan.md) · [Indeks](../../README.md)

---

## Yang dihasilkan di tahap ini

- UI lebih konsisten (Bootstrap)
- Bug utama beres
- Hardening validasi & otorisasi
- Seeder demo kaya untuk presentasi 5–10 menit
- Test case manual lulus
- (Opsional) feature test otomatis
- Cara install & akun demo di README akurat
- Aplikasi stabil untuk demo penuh

---

## Fokus pekerjaan

Di tahap ini **jangan menambah modul besar**.

Rapikan dan pastikan alur wajib dari tahap 01–08 berjalan mulus.

---

## 1. Perapian UI/UX

Kerjakan berurutan:

- [ ] Samakan layout, spacing, dan gaya tombol di semua halaman
- [ ] Navbar menampilkan menu sesuai role saja
- [ ] Empty state jelas (“Belum ada data”)
- [ ] Flash success/error konsisten
- [ ] Error validasi menempel di field (`is-invalid` + `invalid-feedback`)
- [ ] Konfirmasi sebelum hapus / batalkan / finalisasi
- [ ] Badge status berwarna konsisten (log, internship, nilai)
- [ ] Tabel responsif (`table-responsive`) di mobile
- [ ] Judul halaman (`@section('title')`) benar
- [ ] Loading state tombol submit (opsional: disable + teks “Menyimpan…”)

---

## 2. Hardening (keamanan & validasi)

- [ ] Semua input penting tervalidasi server-side (Form Request)
- [ ] Middleware `auth` + `role` di route sensitif
- [ ] Policy dipanggil di show/update/delete/download/review
- [ ] Upload tetap dibatasi tipe & ukuran (2MB)
- [ ] User `is_active = false` tidak bisa login
- [ ] Tidak ada IDOR: user A tidak akses data user B lewat ganti URL id
- [ ] Mass assignment aman (`$fillable` / `$guarded`)
- [ ] `.env` tidak ikut ter-push
- [ ] `APP_DEBUG=false` saat demo di server (jika di-deploy)

Uji IDOR cepat:

1. Login sebagai mahasiswa A, catat URL detail log miliknya  
2. Login mahasiswa B (atau buat user kedua), buka URL itu → harus 403  

---

## 3. Seeder demo final

Siapkan data cukup untuk demo 5–10 menit:

| Data | Jumlah saran |
|------|----------------|
| Admin | 1 |
| Mahasiswa | 2 |
| Pembimbing industri | 1–2 |
| Tempat magang | 2 |
| Periode aktif | 1 |
| Internship active | minimal 2 (masing-masing mahasiswa) |
| Activity logs | campur draft/submitted/approved/rejected |
| Documents | 1–2 per mahasiswa |
| Assessment | 1 final + 1 draft (opsional) |
| Notifications | beberapa contoh |

Password seragam: `password`

Akun inti yang harus selalu ada:

| Email | Role |
|-------|------|
| admin@demo.test | admin |
| mahasiswa@demo.test | mahasiswa |
| industri@demo.test | pembimbing_industri |

Jalankan:

```bash
php artisan migrate:fresh --seed
```

> Hanya di local/demo. Jangan di production yang berisi data nyata.

Pastikan `DatabaseSeeder` memanggil semua seeder berurutan (users → companies → periods → internships → logs → documents → aspects → assessments → notifications).

---

## 4. Pengujian manual (wajib)

Jalankan satu per satu. Centang setelah lulus.

| ID | Skenario | Hasil diharapkan |
|----|----------|------------------|
| TC-01 | Login tiap role | Masuk dashboard yang benar |
| TC-02 | Mahasiswa buka URL admin | 403 |
| TC-03 | Tambah & submit log | Status `submitted` |
| TC-04 | Approve log | Status `approved`, terkunci |
| TC-05 | Reject tanpa catatan | Validasi gagal |
| TC-06 | Revisi log rejected | Bisa submit ulang |
| TC-07 | Edit log approved | Tidak boleh |
| TC-08 | Upload dokumen valid | Tersimpan, bisa diunduh |
| TC-09 | Upload file ilegal / terlalu besar | Ditolak |
| TC-10 | Input nilai pembimbing + finalisasi | Nilai akhir benar (rumus berbobot) |
| TC-11 | Export Excel/PDF | File terunduh & isi benar |
| TC-12 | Isolasi bimbingan | Pembimbing A tidak lihat mahasiswa B |
| TC-13 | Satu mahasiswa dua internship active | Ditolak |
| TC-14 | User nonaktif login | Ditolak |
| TC-15 | Ubah password di profil | Login dengan password baru berhasil |

Centang:

- [ ] TC-01
- [ ] TC-02
- [ ] TC-03
- [ ] TC-04
- [ ] TC-05
- [ ] TC-06
- [ ] TC-07
- [ ] TC-08
- [ ] TC-09
- [ ] TC-10
- [ ] TC-11
- [ ] TC-12
- [ ] TC-13
- [ ] TC-14
- [ ] TC-15

---

## 5. Testing otomatis (opsional tapi bagus)

Buat beberapa feature test untuk alur inti:

- login role & redirect
- mahasiswa tidak akses admin
- submit log
- approve log oleh pembimbing yang benar
- pembimbing salah ditolak
- hitung nilai berbobot

```bash
php artisan make:test ActivityLogApprovalTest
php artisan test
```

Contoh assert sederhana:

```php
$this->actingAs($mahasiswa)
    ->post(route('mahasiswa.logs.store'), [...])
    ->assertRedirect();

$this->assertDatabaseHas('activity_logs', [
    'title' => 'Setup environment',
    'status' => 'draft',
]);
```

---

## 6. README instalasi MagangTrack

Pastikan README project menjelaskan cara menjalankan aplikasi:

- [ ] Stack yang dipakai (Bootstrap CDN, auth manual, PhpSpreadsheet, DomPDF)
- [ ] Cara install (`composer install`, `.env`, `migrate --seed`, `serve`) — tanpa npm
- [ ] Akun demo (email + password)
- [ ] Role yang ada (admin, mahasiswa, pembimbing industri)

---

## 7. Demo end-to-end (latihan presentasi)

Latihan sampai lancar tanpa baca catatan terlalu sering:

1. **Admin** login → tunjukkan master data → buka penempatan mahasiswa  
2. **Mahasiswa** login → buat log → submit → upload dokumen  
3. **Pembimbing** login → approve/reject → input nilai → finalisasi  
4. **Mahasiswa** lihat nilai & status log  
5. **Admin** dashboard + export laporan Excel/PDF  

Jika salah satu putus, perbaiki dulu sebelum menambah fitur bonus.

Durasi ideal demo: **5–10 menit**.

---

## 8. Commit & repo rapi

- [ ] `.env` tidak ada di remote
- [ ] Tidak ada token/password mentah di kode atau history commit
- [ ] Pesan commit jelas per fitur
- [ ] Branch utama bisa di-clone lalu `migrate --seed` jalan

---

## Checklist selesai project

- [ ] Semua tahap 01–08 beres secara fungsional
- [ ] Alur inti tidak error
- [ ] TC manual lulus (minimal TC-01 s/d TC-12)
- [ ] Seeder demo siap (`migrate:fresh --seed`)
- [ ] UI cukup rapi untuk ditampilkan
- [ ] Export Excel & PDF jalan
- [ ] Isolasi role teruji
- [ ] Cara install & akun demo di README benar

---

## Setelah tahap 09 (backlog bonus — jangan ganggu demo)

Kerjakan hanya jika inti sudah stabil:

- Notifikasi email (SMTP)
- Chart di dashboard
- Import data dari Excel
- Multi-file lampiran per log
- Soft delete user + restore
- Activity log audit admin
- Deploy ke hosting

---

## Definition of done (versi 1 siap demo)

Project dianggap selesai v1 jika:

1. Tiga role bisa login dan hanya melihat data yang berhak  
2. Alur log → approval → dokumen → nilai → laporan berjalan  
3. Export Excel & PDF berhasil  
4. Seeder demo cukup untuk presentasi  
5. Clone → install → `migrate --seed` → aplikasi jalan
---

## Selesai

Kembali ke indeks utama: **[README.md](../../README.md)**

Selamat — jika checklist di atas hijau, MagangTrack siap didemo.
