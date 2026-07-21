# Tahap 08 — Dashboard & Laporan

**Urutan:** 08 dari 09  
**Tujuan:** Ringkasan data di dashboard dan export laporan yang berguna untuk monitoring.

← Sebelumnya: [07 — Penilaian](07-penilaian.md) · [Indeks](../../README.md) · Berikutnya: [09 — Polish & Testing](09-polish-testing.md) →

---

## Yang dihasilkan di tahap ini

- Dashboard berisi angka penting per role
- Halaman laporan dengan filter
- Export Excel
- Export PDF
- Notifikasi in-app sederhana (opsional tapi disarankan)

---

## Menu yang aktif di tahap ini

```
├── Dashboard
│   ├── Statistik ringkas
│   ├── Antrian kerja (approval / revisi)
│   └── Notifikasi terbaru
└── Laporan
    ├── Rekap log
    ├── Rekap penilaian
    ├── Export Excel
    └── Export PDF
```

---

## Isi dashboard per role

### Admin
- Total mahasiswa magang aktif
- Total log minggu ini / bulan ini
- Jumlah menunggu approval
- Distribusi status magang

### Mahasiswa
- Jumlah log minggu ini
- Log ditolak yang perlu direvisi
- Kelengkapan dokumen
- Nilai akhir (jika ada)

### Pembimbing Industri
- Antrian log `submitted`
- Jumlah mahasiswa bimbingan
- Penilaian yang belum final

### Dosen
- Progress mahasiswa bimbingan
- Penilaian yang belum final
- Dokumen laporan yang sudah diunggah

---

## Laporan yang dibuat

### 1. Rekap log kegiatan
Filter: periode, mahasiswa, status  
Kolom: tanggal, nama, judul, status, jam kerja

### 2. Rekap penilaian
Filter: periode / perusahaan  
Kolom: mahasiswa, nilai industri, nilai dosen, nilai akhir

### 3. Export
- Excel untuk rekap tabular
- PDF untuk lembar ringkasan / hasil penilaian

Package yang dipakai:

```bash
composer require phpoffice/phpspreadsheet
composer require barryvdh/laravel-dompdf
```

> Excel pakai **PhpSpreadsheet** langsung (bukan Maatwebsite Excel). PDF pakai **DomPDF**.

---

## Notifikasi in-app (sederhana)

### Event yang layak dibuatkan notifikasi
- Log baru menunggu approval → ke pembimbing
- Log di-approve / di-reject → ke mahasiswa
- Nilai sudah final → ke mahasiswa

### Tabel `notifications` (custom) atau pakai database notification Laravel
Minimal field jika custom:
- user_id
- title
- message
- is_read
- url
- created_at

Di navbar: ikon lonceng + jumlah belum dibaca.

---

## Langkah kerja (urut)

### 1. Query statistik dashboard

Buat method di controller/service yang menghitung angka sesuai role.  
Hindari query N+1.

### 2. Rapikan Blade dashboard

Tampilkan kartu angka + daftar singkat (mis. 5 log terbaru).  
Tambahkan link cepat ke halaman terkait.

### 3. Halaman laporan + filter

Form filter GET:
- period_id
- company_id
- student_id
- date_from / date_to
- status

### 4. Export Excel (PhpSpreadsheet)

Di controller/service, buat spreadsheet dengan `PhpOffice\PhpSpreadsheet\Spreadsheet`, isi header + baris data dari query berfilter, lalu unduh via `Xlsx` writer (`Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`).

Tombol “Export Excel” memakai filter yang sama dengan halaman laporan.

### 5. Export PDF (DomPDF)

Buat view Blade PDF sederhana (kop, judul, tabel, tanggal cetak), lalu generate dengan `Barryvdh\DomPDF\Facade\Pdf`.  
Pastikan readable saat diprint.

### 6. Notifikasi

- Create notification saat event terjadi
- List notifikasi
- Mark as read

---

## Checklist selesai

- [ ] Dashboard tiap role menampilkan data relevan
- [ ] Filter laporan berfungsi
- [ ] Export Excel berhasil diunduh
- [ ] Export PDF berhasil diunduh
- [ ] Notifikasi dasar muncul untuk approve/reject (jika dikerjakan)

---

## Cara uji cepat

1. Login admin → cek angka statistik masuk akal  
2. Filter laporan per periode → data berubah  
3. Export Excel & PDF → file terbuka benar  
4. Submit log baru → pembimbing mendapat notifikasi (jika fitur aktif)  

---

**Lanjut ke:** [Tahap 09 — Polish & Testing](09-polish-testing.md)
