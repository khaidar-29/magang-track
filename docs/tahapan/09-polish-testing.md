# Tahap 09 — Polish, Testing & Stabilisasi

**Urutan:** 09 dari 09  
**Tujuan:** Website dirapikan, diuji, dan siap dipakai demo / dipakai sehari-hari.

← Sebelumnya: [08 — Dashboard & Laporan](08-dashboard-laporan.md) · [Indeks](../../README.md)

---

## Yang dihasilkan di tahap ini

- UI lebih konsisten & ramah dipakai
- Bug utama sudah diperbaiki
- Test case manual lulus
- Seeder demo lengkap
- README instalasi final akurat
- Aplikasi stabil untuk demo end-to-end

---

## Fokus pekerjaan (bukan fitur baru besar)

Di tahap ini **jangan menambah modul besar**.  
Rapikan yang sudah ada.

---

## 1. Perapian UI/UX

Kerjakan berurutan:

- [ ] Samakan layout, spacing, dan tombol di semua halaman
- [ ] Tambah empty state (“Belum ada data”)
- [ ] Flash message sukses/error jelas
- [ ] Form error menempel di field yang salah
- [ ] Konfirmasi sebelum hapus
- [ ] Loading state pada tombol submit (opsional)
- [ ] Cek tampilan mobile sederhana (responsive dasar)

---

## 2. Hardening (keamanan & validasi)

- [ ] Semua input penting tervalidasi server-side
- [ ] Policy/middleware dipasang di route sensitif
- [ ] Upload file tetap dibatasi tipe & ukuran
- [ ] Pastikan tidak ada data orang lain bocor antar role
- [ ] `.env` tidak ikut ter-push

---

## 3. Seeder demo final

Siapkan data yang cukup untuk demo 5–10 menit:

- 1 admin
- 2 mahasiswa
- 1–2 pembimbing industri
- 1 dosen
- 2 tempat magang
- 1 periode aktif
- beberapa log (campur status)
- 1–2 dokumen
- penilaian contoh

Password demo seragam, misalnya: `password`

---

## 4. Pengujian manual (wajib)

Jalankan satu per satu:

| ID | Skenario | Hasil diharapkan |
|----|----------|------------------|
| TC-01 | Login tiap role | Masuk dashboard yang benar |
| TC-02 | Mahasiswa buka URL admin | Ditolak |
| TC-03 | Tambah & submit log | Status `submitted` |
| TC-04 | Approve log | Status `approved` |
| TC-05 | Reject tanpa catatan | Validasi gagal |
| TC-06 | Revisi log rejected | Bisa submit ulang |
| TC-07 | Edit log approved | Tidak boleh |
| TC-08 | Upload dokumen valid | Tersimpan |
| TC-09 | Upload file ilegal | Ditolak |
| TC-10 | Input nilai industri + dosen | Nilai akhir benar |
| TC-11 | Export Excel/PDF | File terunduh |
| TC-12 | Isolasi bimbingan | Pembimbing A tidak lihat mahasiswa B |

Centang di sini setelah lulus:
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

---

## 5. Testing otomatis (opsional tapi bagus)

Buat beberapa feature test untuk alur inti:
- login role
- submit log
- approve log
- hitung nilai akhir

```bash
php artisan test
```

---

## 6. Dokumentasi final di README utama

Pastikan README root berisi:
- cara install yang benar
- akun demo
- link ke semua tahap
- stack yang benar-benar dipakai

---

## 7. Demo end-to-end (latihan)

Latihan alur ini sampai lancar:

1. Admin menempatkan mahasiswa  
2. Mahasiswa isi log + upload dokumen  
3. Pembimbing approve/reject  
4. Industri & dosen input nilai  
5. Lihat dashboard + export laporan  

Jika salah satu putus, perbaiki dulu sebelum menambah fitur bonus.

---

## Checklist selesai project

- [ ] Semua tahap 01–08 sudah beres
- [ ] Alur inti tidak error
- [ ] Test case manual lulus
- [ ] Seeder demo siap
- [ ] UI cukup rapi untuk ditampilkan
- [ ] Repo GitHub rapi (commit bertahap)

---

## Setelah tahap 09

Website dianggap **selesai versi 1**.

Pengembangan tambahan (boleh belakangan):
- email notification
- chart dashboard lebih lengkap
- import data Excel
- PWA / mobile-friendly lebih jauh

Jangan kerjakan bonus jika checklist di atas belum selesai.

---

**Selesai.** Kembali ke [README utama](../../README.md)
