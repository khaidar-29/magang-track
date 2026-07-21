# Tahap 06 — Dokumen

**Urutan:** 06 dari 09  
**Tujuan:** Mahasiswa bisa mengunggah dokumen magang; pihak terkait bisa mengunduh sesuai hak akses.

← Sebelumnya: [05 — Log Kegiatan](05-log-kegiatan.md) · [Indeks](../../README.md) · Berikutnya: [07 — Penilaian](07-penilaian.md) →

---

## Yang dihasilkan di tahap ini

- Upload dokumen per jenis
- Validasi tipe & ukuran file
- Penyimpanan lewat Laravel Storage
- Download terproteksi
- Daftar dokumen per mahasiswa / per internship

---

## Menu yang aktif di tahap ini

```
├── Dokumen
│   ├── Daftar dokumen
│   ├── Upload dokumen
│   └── Unduh dokumen
```

---

## Jenis dokumen (awal)

| Kode | Nama |
|------|------|
| proposal | Proposal magang |
| acceptance_letter | Surat penerimaan |
| weekly_report | Laporan mingguan |
| final_report | Laporan akhir |
| certificate | Sertifikat |
| other | Lainnya |

Boleh disimpan sebagai enum/string di kolom `type`.

---

## Field `documents`

| Field | Keterangan |
|-------|------------|
| internship_id | Penempatan terkait |
| uploaded_by | User pengunggah |
| type | Jenis dokumen |
| title | Judul tampilan |
| file_path | Path di storage |
| file_name | Nama asli file |
| mime_type | MIME |
| file_size | Ukuran byte |
| notes | Opsional |

---

## Aturan

1. Hanya mahasiswa pemilik internship yang boleh upload (admin boleh bantu jika perlu).
2. Pembimbing industri & dosen bimbingan boleh lihat/unduh.
3. Batasi ekstensi: `pdf`, `jpg`, `jpeg`, `png`, `doc`, `docx`.
4. Batasi ukuran: misalnya max **2MB** atau **5MB**.
5. Jangan taruh file sensitif langsung di folder public tanpa kontrol akses.

---

## Langkah kerja (urut)

### 1. Migration + model

```bash
php artisan make:model Document -m
```

### 2. Relasi

- `Internship` hasMany `Document`
- `Document` belongsTo `Internship`
- `Document` belongsTo uploader (`User`)

### 3. Form upload

Field form:
- jenis dokumen
- judul
- file
- catatan (opsional)

Validasi contoh:

```text
type: required
title: required|string|max:150
file: required|file|mimes:pdf,jpg,jpeg,png,doc,docx|max:2048
```

### 4. Simpan file

Pakai storage disk `local` (private) lalu unduh via controller:

```text
storage/app/documents/...
```

Simpan metadata ke tabel `documents`.

### 5. Daftar dokumen

Tampilkan:
- judul, jenis, tanggal upload, ukuran
- tombol unduh
- tombol hapus (hanya pemilik / admin, dan hanya jika diizinkan)

### 6. Download terproteksi

Buat route download yang:
1. Cek auth
2. Cek policy (apakah user berhak)
3. Baru `Storage::download(...)`

Jangan expose path mentah ke user tanpa cek.

### 7. Hapus dokumen

Hapus record + hapus file fisik di storage.  
Pakai konfirmasi di UI.

---

## Checklist selesai

- [ ] Mahasiswa bisa upload dokumen valid
- [ ] File ilegal / oversized ditolak
- [ ] Pembimbing bisa unduh dokumen mahasiswa bimbingannya
- [ ] Orang lain tidak bisa unduh dokumen sembarang
- [ ] Hapus dokumen menghapus file di storage

---

## Cara uji cepat

1. Login mahasiswa → upload PDF laporan  
2. Coba upload file `.exe` → harus gagal  
3. Login pembimbing → unduh dokumen tersebut berhasil  
4. Login mahasiswa lain → tidak bisa unduh dokumen orang lain  

---

**Lanjut ke:** [Tahap 07 — Penilaian](07-penilaian.md)
