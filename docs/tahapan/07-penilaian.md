# Tahap 07 — Penilaian

**Urutan:** 07 dari 09  
**Tujuan:** Pembimbing industri dan dosen memberi nilai; sistem menghitung nilai akhir.

← Sebelumnya: [06 — Dokumen](06-dokumen.md) · [Indeks](../../README.md) · Berikutnya: [08 — Dashboard & Laporan](08-dashboard-laporan.md) →

---

## Yang dihasilkan di tahap ini

- Master aspek penilaian
- Form input nilai oleh industri & dosen
- Perhitungan nilai akhir berbobot
- Mahasiswa bisa melihat hasil akhir (bukan form input)
- Kunci nilai final agar tidak berubah sembarangan

---

## Menu yang aktif di tahap ini

```
├── Penilaian
│   ├── Aspek penilaian          ← Admin
│   ├── Input nilai industri     ← Pembimbing industri
│   ├── Input nilai dosen        ← Dosen
│   └── Hasil nilai akhir        ← Mahasiswa / semua terkait
```

---

## Desain penilaian

### Aspek contoh

| Aspek | Bobot default (opsional per aspek) |
|-------|-------------------------------------|
| Disiplin & kehadiran | 20 |
| Kualitas pekerjaan | 25 |
| Komunikasi | 20 |
| Inisiatif | 15 |
| Dokumentasi / laporan | 20 |

> Boleh juga aspek tanpa bobot per item, lalu rata-rata sederhana. Yang penting konsisten.

### Komponen nilai akhir

| Komponen | Bobot |
|----------|------:|
| Nilai Pembimbing Industri | 60% |
| Nilai Dosen Pembimbing | 40% |

Rumus:

```text
nilai_akhir = (nilai_industri * 0.60) + (nilai_dosen * 0.40)
```

Bobot bisa disimpan di config/pengaturan agar mudah diubah.

---

## Tabel yang dibutuhkan

### `assessment_aspects`
- name
- description
- max_score (mis. 100)
- is_active

### `assessments`
- internship_id
- assessor_id
- assessor_role (`pembimbing_industri` / `dosen`)
- total_score
- is_final (boolean)
- notes

### `assessment_scores`
- assessment_id
- aspect_id
- score

---

## Aturan

1. Hanya pembimbing yang di-assign yang boleh menilai mahasiswa tersebut.
2. Satu jenis penilai (industri/dosen) idealnya satu assessment per internship.
3. Setelah `is_final = true`, nilai terkunci (unlock hanya admin jika perlu).
4. Mahasiswa hanya melihat ringkasan hasil, tidak boleh edit skor.
5. Nilai per aspek tidak boleh melebihi `max_score`.

---

## Langkah kerja (urut)

### 1. Migration + model

```bash
php artisan make:model AssessmentAspect -m
php artisan make:model Assessment -m
php artisan make:model AssessmentScore -m
```

### 2. Seeder aspek penilaian

Isi 5 aspek default agar form langsung siap dipakai.

### 3. CRUD aspek (Admin)

Admin bisa tambah/nonaktifkan aspek.  
Jangan hapus keras aspek yang sudah dipakai di skor lama.

### 4. Form input nilai

Halaman untuk pembimbing/dosen:
- pilih mahasiswa bimbingan
- isi skor per aspek
- catatan akhir
- tombol simpan draft / finalisasi

Hitung `total_score` di backend (jangan percaya perhitungan dari frontend saja).

### 5. Hitung nilai akhir

Saat keduanya final (atau salah satu dulu ditampilkan sebagian):
- ambil total industri & dosen
- terapkan bobot
- tampilkan nilai akhir di detail internship / halaman hasil

### 6. Policy & isolasi data

Uji ketat:
- pembimbing A tidak menilai mahasiswa bimbingan B
- mahasiswa tidak bisa POST ke endpoint penilaian

### 7. Tampilan hasil untuk mahasiswa

Tampilkan:
- nilai industri
- nilai dosen
- nilai akhir
- status (belum lengkap / final)

---

## Checklist selesai

- [ ] Admin punya daftar aspek penilaian
- [ ] Pembimbing industri bisa input & finalisasi nilai
- [ ] Dosen bisa input & finalisasi nilai
- [ ] Nilai akhir terhitung sesuai bobot
- [ ] Nilai final terkunci
- [ ] Mahasiswa hanya bisa melihat hasil

---

## Cara uji cepat

1. Login industri → isi skor mahasiswa demo → finalisasi  
2. Login dosen → isi skor → finalisasi  
3. Cek nilai akhir = (industri×0.6) + (dosen×0.4)  
4. Coba ubah nilai yang sudah final → ditolak  
5. Login mahasiswa → hasil tampil, form input tidak ada  

---

**Lanjut ke:** [Tahap 08 — Dashboard & Laporan](08-dashboard-laporan.md)
