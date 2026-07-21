# Tahap 07 — Penilaian

**Urutan:** 07 dari 09  
**Tujuan:** Admin mengelola aspek penilaian; pembimbing industri mengisi skor per aspek lalu finalisasi; mahasiswa melihat nilai akhir (read-only).

← Sebelumnya: [06 — Dokumen](06-dokumen.md) · [Indeks](../../README.md) · Berikutnya: [08 — Dashboard & Laporan](08-dashboard-laporan.md) →

---

## Yang dihasilkan di tahap ini

- Master aspek penilaian (`assessment_aspects`) berbobot
- Form input nilai oleh **pembimbing industri**
- Perhitungan `total_score` / nilai akhir di backend
- Status draft vs final (`is_final`)
- Nilai final terkunci
- Mahasiswa melihat hasil tanpa bisa edit

Tidak ada nilai dosen. Satu sumber nilai: pembimbing industri.

---

## Menu yang aktif

```text
├── Penilaian
│   ├── Aspek penilaian          ← Admin
│   ├── Input nilai              ← Pembimbing Industri
│   └── Hasil nilai              ← Mahasiswa / terkait (read-only)
```

---

## Desain penilaian

### Aspek default (seeder)

| Aspek | Bobot (`weight`) | max_score |
|-------|-----------------:|----------:|
| Disiplin & kehadiran | 20 | 100 |
| Kualitas pekerjaan | 25 | 100 |
| Komunikasi | 20 | 100 |
| Inisiatif | 15 | 100 |
| Dokumentasi / laporan | 20 | 100 |
| **Total bobot** | **100** | |

### Rumus nilai akhir (wajib dipakai)

Setiap aspek dinilai 0 … `max_score` (biasanya 0–100).

```text
nilai_akhir = Σ (skor_aspek / max_score * 100 * weight) / Σ weight
```

Jika semua `max_score = 100`, menyederhanakan jadi:

```text
nilai_akhir = Σ (skor_aspek * weight) / Σ weight
```

Bulatkan ke **2 desimal** di backend.

Contoh: skor 80,90,70,85,75 dengan bobot di atas → hitung berbobot.

---

## Tabel

### `assessment_aspects`

| Field | Tipe | Keterangan |
|-------|------|------------|
| name | string | Nama aspek |
| description | text, nullable | |
| weight | unsignedInteger | Bobot |
| max_score | unsignedInteger | Default 100 |
| is_active | boolean | Aspek nonaktif tidak muncul di form baru |
| timestamps | | |

### `assessments`

| Field | Tipe | Keterangan |
|-------|------|------------|
| internship_id | FK | Unique disarankan (1 assessment per internship) |
| assessor_id | FK users | Pembimbing yang menilai |
| total_score | decimal(5,2), nullable | Dihitung backend |
| is_final | boolean | Default false |
| notes | text, nullable | Catatan akhir |
| finalized_at | timestamp, nullable | |
| timestamps | | |

Constraint disarankan:

```php
$table->unique('internship_id');
```

### `assessment_scores`

| Field | Tipe | Keterangan |
|-------|------|------------|
| assessment_id | FK | |
| aspect_id | FK | |
| score | decimal(5,2) | 0 … max_score aspek |
| unique(assessment_id, aspect_id) | | |

---

## Aturan bisnis

1. Hanya pembimbing yang di-assign (`industrial_supervisor_id`) yang boleh input nilai.
2. Satu assessment per internship (update skor selama belum final).
3. Semua aspek **aktif** harus terisi sebelum finalisasi.
4. Setelah `is_final = true`, skor terkunci. Unlock hanya admin (opsional).
5. Mahasiswa hanya melihat hasil; tidak boleh POST skor.
6. Idealnya penilaian hanya untuk internship `active` atau `completed`.

---

## Langkah kerja (urut)

### 1. Migration + model

```bash
php artisan make:model AssessmentAspect -m
php artisan make:model Assessment -m
php artisan make:model AssessmentScore -m
php artisan make:policy AssessmentPolicy --model=Assessment
```

Buat migration sesuai skema di atas. Jangan lupa kolom **`weight`**.

---

### 2. Relasi

```php
// Assessment
public function internship() { return $this->belongsTo(Internship::class); }
public function assessor() { return $this->belongsTo(User::class, 'assessor_id'); }
public function scores() { return $this->hasMany(AssessmentScore::class); }

// AssessmentScore
public function aspect() { return $this->belongsTo(AssessmentAspect::class, 'aspect_id'); }
public function assessment() { return $this->belongsTo(Assessment::class); }

// Internship
public function assessment() { return $this->hasOne(Assessment::class); }
```

---

### 3. Seeder aspek

```bash
php artisan make:seeder AssessmentAspectSeeder
```

Isi 5 aspek default (tabel di atas). Panggil dari `DatabaseSeeder`.

---

### 4. CRUD aspek (Admin)

Resource sederhana:

- Index daftar aspek
- Create / edit name, description, weight, max_score, is_active
- Jangan hapus keras aspek yang sudah punya `assessment_scores` — nonaktifkan saja

Validasi: `weight` >= 1; total bobot aktif idealnya 100 (boleh soft-warning di UI, atau enforce ketat).

---

### 5. Form input nilai (Pembimbing)

Alur UI:

1. Pilih mahasiswa bimbingan (internship list)
2. Form skor untuk tiap aspek aktif
3. Catatan akhir
4. Tombol **Simpan draft** / **Finalisasi**

Pseudo store/update:

```php
DB::transaction(function () use ($request, $internship) {
    $assessment = Assessment::firstOrCreate(
        ['internship_id' => $internship->id],
        ['assessor_id' => auth()->id(), 'is_final' => false]
    );

    abort_if($assessment->is_final, 403, 'Nilai sudah final.');

    $aspects = AssessmentAspect::where('is_active', true)->get();
    $weights = 0;
    $weighted = 0;

    foreach ($aspects as $aspect) {
        $score = (float) $request->input('scores.'.$aspect->id);
        abort_if($score < 0 || $score > $aspect->max_score, 422);

        AssessmentScore::updateOrCreate(
            ['assessment_id' => $assessment->id, 'aspect_id' => $aspect->id],
            ['score' => $score]
        );

        $weights += $aspect->weight;
        $weighted += ($score / $aspect->max_score) * 100 * $aspect->weight;
    }

    $total = $weights > 0 ? round($weighted / $weights, 2) : 0;

    $assessment->update([
        'assessor_id' => auth()->id(),
        'total_score' => $total,
        'notes' => $request->notes,
        'is_final' => $request->boolean('finalize'),
        'finalized_at' => $request->boolean('finalize') ? now() : null,
    ]);
});
```

Hitung **selalu di server**, jangan percaya angka dari hidden input frontend.

---

### 6. Unlock oleh admin (opsional)

```text
POST /admin/assessments/{assessment}/unlock
```

Set `is_final = false`, `finalized_at = null`.  
Berguna jika salah input.

---

### 7. Tampilan hasil mahasiswa

Tampilkan:

- Status: Belum dinilai / Draft / Final
- Tabel aspek + skor (jika sudah ada)
- Nilai akhir (`total_score`)
- Catatan pembimbing (jika final / jika diizinkan)

Tidak ada form input di sisi mahasiswa.

---

### 8. Policy

| Ability | Pembimbing assign | Mahasiswa pemilik | Admin |
|---------|:-----------------:|:-----------------:|:-----:|
| upsert draft | ✓ | — | — |
| finalize | ✓ | — | — |
| view result | ✓ | ✓ | ✓ |
| unlock | — | — | ✓ |

---

## Checklist selesai

- [ ] 3 tabel penilaian ter-migrate
- [ ] Aspek punya kolom `weight` & seeder 5 aspek
- [ ] Admin CRUD aspek
- [ ] Pembimbing bisa simpan draft skor
- [ ] Finalisasi mengunci nilai
- [ ] `total_score` dihitung berbobot di backend
- [ ] Unique 1 assessment per internship
- [ ] Mahasiswa hanya lihat hasil
- [ ] Pembimbing lain tidak bisa menilai mahasiswa bukan bimbingannya
- [ ] Skor di luar 0…max_score ditolak

---

## Cara uji cepat

1. Login industri → pilih mahasiswa demo → isi skor semua aspek → finalisasi  
2. Cek `total_score` sesuai rumus berbobot  
3. Coba ubah lagi → ditolak / form terkunci  
4. Login mahasiswa → halaman hasil menampilkan nilai  
5. Login pembimbing lain → tidak bisa akses form nilai mahasiswa itu  

---

## Kesalahan umum

1. **Aspek tanpa `weight`** — rumus berbobot mustahil; pastikan kolom ada.
2. **Menghitung di JavaScript saja** — harus diulang di backend.
3. **Finalisasi meski aspek kosong** — validasi semua aspek aktif terisi.
4. **Banyak assessment per internship** — pakai `unique(internship_id)` + `firstOrCreate`.
5. **Mahasiswa bisa tebak URL POST** — policy + middleware role wajib.

---

**Lanjut ke:** [Tahap 08 — Dashboard & Laporan](08-dashboard-laporan.md)
