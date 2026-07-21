# Tahap 05 — Log Kegiatan & Approval

**Urutan:** 05 dari 09  
**Estimasi:** 6–8 jam  
**Tujuan:** Mahasiswa mencatat kegiatan harian; pembimbing industri menyetujui atau menolak. Ini **modul inti** MagangTrack.

← Sebelumnya: [04 — Data Magang](04-data-magang.md) · [Indeks](../../README.md) · Berikutnya: [06 — Dokumen](06-dokumen.md) →

---

## Yang dihasilkan di tahap ini

- Tabel `activity_logs`
- Mahasiswa: buat, edit (draft/rejected), submit log
- Pembimbing industri: antrian `submitted`, approve, reject (+ catatan wajib)
- Status workflow lengkap
- Filter & pencarian
- Policy isolasi ketat
- Flash message feedback

**Lampiran bukti per log:** tidak wajib di tahap ini. Upload file digarap di tahap 06 sebagai dokumen tingkat internship (lebih sederhana & cukup untuk project).

**Notifikasi in-app:** cukup flash dulu; notifikasi database di tahap 08.

---

## Menu yang aktif

```text
├── Log Kegiatan
│   ├── Daftar log / form          ← Mahasiswa
│   ├── Detail log
│   └── Antrian approval           ← Pembimbing Industri
```

Admin boleh punya halaman “semua log” (read-only) — opsional.

---

## Field `activity_logs`

| Field | Tipe | Keterangan |
|-------|------|------------|
| internship_id | FK | Penempatan terkait |
| title | string | Judul singkat kegiatan |
| activity_date | date | Tanggal kegiatan |
| description | text | Uraian kegiatan |
| work_hours | decimal(4,1), nullable | Jam kerja hari itu (contoh 8.0) |
| status | string | `draft` / `submitted` / `approved` / `rejected` |
| reviewer_id | FK users, nullable | Pembimbing yang mereview |
| review_note | text, nullable | Wajib jika reject |
| reviewed_at | timestamp, nullable | Waktu review |
| submitted_at | timestamp, nullable | Waktu submit |
| timestamps | | |

---

## Alur status

```text
 draft ──submit──► submitted ──approve──► approved
                      │
                      └──reject──► rejected ──edit+submit──► submitted
```

| Status | Mahasiswa boleh | Pembimbing boleh |
|--------|-----------------|------------------|
| draft | edit, hapus, submit | — |
| submitted | lihat saja | approve, reject |
| approved | lihat saja (terkunci) | lihat |
| rejected | edit, submit ulang | lihat |

### Aturan bisnis

1. Log hanya boleh dibuat jika internship status = **`active`**.
2. `activity_date` harus berada dalam rentang periode magang (`period.start_date` … `period.end_date`).
3. Hanya pembimbing yang di-assign di internship itu yang boleh approve/reject.
4. Reject **wajib** `review_note`.
5. Log `approved` tidak bisa diubah mahasiswa.
6. Saat approve/reject: set `reviewer_id`, `reviewed_at`; saat submit: set `submitted_at`, status `submitted`, kosongkan review lama jika dari rejected.

---

## Langkah kerja (urut)

### 1. Migration + model

```bash
php artisan make:model ActivityLog -m
php artisan make:policy ActivityLogPolicy --model=ActivityLog
```

```php
Schema::create('activity_logs', function (Blueprint $table) {
    $table->id();
    $table->foreignId('internship_id')->constrained()->cascadeOnDelete();
    $table->string('title');
    $table->date('activity_date');
    $table->text('description');
    $table->decimal('work_hours', 4, 1)->nullable();
    $table->string('status', 20)->default('draft');
    $table->foreignId('reviewer_id')->nullable()->constrained('users')->nullOnDelete();
    $table->text('review_note')->nullable();
    $table->timestamp('submitted_at')->nullable();
    $table->timestamp('reviewed_at')->nullable();
    $table->timestamps();

    $table->index(['internship_id', 'status']);
    $table->index(['activity_date']);
});
```

---

### 2. Relasi

```php
// ActivityLog
public function internship() { return $this->belongsTo(Internship::class); }
public function reviewer() { return $this->belongsTo(User::class, 'reviewer_id'); }

// Internship
public function activityLogs() { return $this->hasMany(ActivityLog::class); }
```

---

### 3. Policy matrix

| Ability | Mahasiswa pemilik | Pembimbing assign | Admin |
|---------|:-----------------:|:-----------------:|:-----:|
| viewAny | ✓ scoped | ✓ scoped | ✓ |
| view | ✓ | ✓ | ✓ |
| create | ✓ (punya internship active) | — | — |
| update | ✓ jika draft/rejected | — | — |
| delete | ✓ jika draft | — | — |
| submit | ✓ jika draft/rejected | — | — |
| review | — | ✓ jika submitted | — |

Helper di model/policy: ambil internship → bandingkan `student_id` / `industrial_supervisor_id`.

---

### 4. Fitur mahasiswa

Controller saran: `Mahasiswa\ActivityLogController`

Halaman:

1. **Index** — daftar log milik internship mahasiswa (filter status, tanggal, keyword judul)
2. **Create / Store** — form tambah (status awal `draft`)
3. **Edit / Update** — hanya draft/rejected
4. **Submit** — `PATCH` / `POST` aksi submit
5. **Show** — detail + catatan penolakan jika ada
6. **Destroy** — hanya draft

Validasi store/update:

```php
'title' => ['required', 'string', 'max:150'],
'activity_date' => ['required', 'date'],
'description' => ['required', 'string'],
'work_hours' => ['nullable', 'numeric', 'min:0', 'max:24'],
```

Setelah validasi, cek:

- user punya internship `active` (ambil yang active; jika belum ada → error)
- tanggal dalam periode
- authorize create/update

Tombol di UI:

- Simpan draft
- Simpan & Submit (atau tombol Submit terpisah di detail)

---

### 5. Fitur pembimbing

Controller saran: `Pembimbing\ActivityLogReviewController`

**Antrian**

```php
ActivityLog::with(['internship.student', 'internship.company'])
    ->where('status', 'submitted')
    ->whereHas('internship', fn ($q) =>
        $q->where('industrial_supervisor_id', auth()->id())
    )
    ->latest('submitted_at')
    ->paginate(10);
```

**Approve**

```php
$log->update([
    'status' => 'approved',
    'reviewer_id' => auth()->id(),
    'reviewed_at' => now(),
    'review_note' => null,
]);
```

**Reject**

```php
$data = $request->validate([
    'review_note' => ['required', 'string', 'min:5'],
]);

$log->update([
    'status' => 'rejected',
    'reviewer_id' => auth()->id(),
    'reviewed_at' => now(),
    'review_note' => $data['review_note'],
]);
```

UI reject: modal Bootstrap dengan textarea catatan.

---

### 6. Route contoh

```text
# Mahasiswa
GET    /mahasiswa/logs
GET    /mahasiswa/logs/create
POST   /mahasiswa/logs
GET    /mahasiswa/logs/{log}
GET    /mahasiswa/logs/{log}/edit
PUT    /mahasiswa/logs/{log}
POST   /mahasiswa/logs/{log}/submit
DELETE /mahasiswa/logs/{log}

# Pembimbing
GET    /pembimbing/logs
GET    /pembimbing/logs/{log}
POST   /pembimbing/logs/{log}/approve
POST   /pembimbing/logs/{log}/reject
```

Semua dalam middleware `auth` + `role:...` + authorize policy.

---

### 7. Filter & pencarian

Parameter GET: `status`, `date_from`, `date_to`, `q` (judul).

Gunakan `when()` di query builder. Pagination `withQueryString()`.

---

### 8. UX wajib

- Badge warna status (draft=secondary, submitted=warning, approved=success, rejected=danger)
- Empty state: “Belum ada log”
- Flash success: “Log berhasil disubmit”, “Log disetujui”, dll.
- Di detail rejected: tampilkan `review_note` mencolok

---

## Checklist selesai

- [ ] Migration `activity_logs` jalan
- [ ] Mahasiswa bisa buat & edit draft
- [ ] Submit mengubah status ke `submitted`
- [ ] Pembimbing melihat antrian hanya bimbingan sendiri
- [ ] Approve mengunci log
- [ ] Reject wajib catatan; mahasiswa bisa revisi & submit ulang
- [ ] Tidak bisa edit log approved
- [ ] Tidak bisa buat log jika tidak ada internship active
- [ ] Tanggal di luar periode ditolak
- [ ] Isolasi: pembimbing A tidak review mahasiswa B
- [ ] Filter status/tanggal/keyword berfungsi

---

## Cara uji cepat (alur penuh)

1. Login mahasiswa → buat log → simpan draft → submit  
2. Login industri → antrian muncul → reject tanpa catatan → gagal  
3. Reject dengan catatan → sukses  
4. Login mahasiswa → lihat catatan → edit → submit ulang  
5. Login industri → approve  
6. Login mahasiswa → tombol edit hilang / ditolak  
7. Login pembimbing lain → log tidak muncul  

---

## Kesalahan umum

1. **Approve tanpa cek supervisor assign** — celah keamanan serius.
2. **Submit dari status approved** — harus dicegah.
3. **Internship completed masih boleh create log** — ikuti aturan hanya `active`.
4. **N+1 di antrian** — selalu `with([...])`.
5. **Reject note nullable di DB tapi wajib di app** — tetap enforce di validasi request.

---

**Lanjut ke:** [Tahap 06 — Dokumen](06-dokumen.md)
