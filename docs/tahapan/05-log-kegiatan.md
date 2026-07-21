# Tahap 05 — Log Kegiatan & Approval

**Urutan:** 05 dari 09  
**Tujuan:** Mahasiswa mencatat kegiatan harian; pembimbing menyetujui atau menolak.

Ini modul inti website. Kerjakan lebih teliti.

← Sebelumnya: [04 — Data Magang](04-data-magang.md) · [Indeks](../../README.md) · Berikutnya: [06 — Dokumen](06-dokumen.md) →

---

## Yang dihasilkan di tahap ini

- CRUD log kegiatan untuk mahasiswa
- Alur status: `draft` → `submitted` → `approved` / `rejected`
- Approval oleh pembimbing industri
- Revisi jika ditolak
- Filter, search, pagination
- Upload lampiran bukti per log (opsional di tahap ini, boleh disederhanakan)

---

## Menu yang aktif di tahap ini

```
├── Log Kegiatan
│   ├── Daftar log
│   ├── Tambah log
│   ├── Detail / edit log
│   └── Antrian approval          ← Pembimbing
```

---

## Field log kegiatan

| Field | Keterangan |
|-------|------------|
| internship_id | Penempatan magang terkait |
| date | Tanggal kegiatan |
| title | Judul singkat |
| description | Uraian kegiatan |
| work_hours | Jam kerja (opsional) |
| status | draft / submitted / approved / rejected |
| reviewer_id | Pembimbing yang review |
| reviewed_at | Waktu review |
| review_note | Catatan approve/reject |

---

## Alur status (wajib dipahami)

```text
[Mahasiswa] buat log
     │
     ▼
  draft ──submit──► submitted
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
      approved                  rejected
                                   │
                                   └── mahasiswa revisi ──submit──► submitted
```

### Aturan
1. Log hanya untuk internship berstatus `active`.
2. Tanggal log harus dalam rentang periode magang.
3. `approved` tidak bisa diedit mahasiswa.
4. `rejected` wajib punya `review_note`.
5. Hanya pembimbing industri yang di-assign yang boleh approve/reject.

---

## Langkah kerja (urut)

### 1. Migration + model

```bash
php artisan make:model ActivityLog -m
```

Jika lampiran dipisah:

```bash
php artisan make:model ActivityLogAttachment -m
```

### 2. Relasi

- `Internship` hasMany `ActivityLog`
- `ActivityLog` belongsTo `Internship`
- `ActivityLog` belongsTo reviewer (`User`)

### 3. Policy

Mahasiswa: create/update milik sendiri (jika status mengizinkan)  
Pembimbing: review milik bimbingan  
Dosen/Admin: bisa lihat (read)

### 4. Fitur mahasiswa

Halaman:
- Daftar log milik sendiri
- Form tambah
- Form edit (hanya `draft` / `rejected`)
- Tombol **Submit**
- Detail + lihat catatan penolakan

Validasi:
- title, date, description wajib
- work_hours numerik jika diisi

### 5. Fitur pembimbing

Halaman antrian:
- List status `submitted`
- Tombol Approve
- Tombol Reject + wajib isi catatan

### 6. Filter & pencarian

Filter minimal:
- by status
- by tanggal (from–to)
- keyword judul

### 7. (Opsional tahap ini) Lampiran bukti

Upload 1 file per log:
- tipe: pdf/jpg/png
- max size: 2MB

Jika waktu mepet, lampiran boleh digeser ke tahap 06.

### 8. Notifikasi sederhana (opsional ringan)

Cukup flash message dulu:
- “Log berhasil disubmit”
- “Log ditolak, silakan revisi”

In-app notification table bisa ditambah di tahap 08.

---

## Checklist selesai

- [ ] Mahasiswa bisa buat & submit log
- [ ] Pembimbing bisa approve
- [ ] Pembimbing bisa reject + catatan
- [ ] Mahasiswa bisa revisi log rejected
- [ ] Log approved terkunci dari edit
- [ ] Filter status berfungsi
- [ ] Role lain tidak bisa mengacak data log orang

---

## Cara uji cepat (alur penuh)

1. Login mahasiswa → buat log → simpan draft  
2. Submit log → status `submitted`  
3. Logout, login pembimbing industri  
4. Approve 1 log → status `approved`  
5. Reject 1 log lain dengan catatan  
6. Login mahasiswa → perbaiki log rejected → submit ulang  
7. Coba edit log yang sudah approved → harus gagal  

---

**Lanjut ke:** [Tahap 06 — Dokumen](06-dokumen.md)
