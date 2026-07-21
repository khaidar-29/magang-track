# Tahap 06 — Upload Dokumen

**Urutan:** 06 dari 09  
**Estimasi:** 3–5 jam  
**Tujuan:** Mahasiswa mengunggah dokumen magang; pembimbing/admin bisa melihat & mengunduh dengan kontrol akses; file disimpan privat (bukan langsung di `public/`).

← Sebelumnya: [05 — Log Kegiatan](05-log-kegiatan.md) · [Indeks](../../README.md) · Berikutnya: [07 — Penilaian](07-penilaian.md) →

---

## Yang dihasilkan di tahap ini

- Tabel `documents`
- Upload dokumen terkait **internship** (bukan per-log)
- Validasi tipe & ukuran file
- Penyimpanan di disk privat
- Download lewat route terproteksi
- Hapus dokumen (DB + file)
- Policy isolasi

---

## Menu yang aktif

```text
├── Dokumen
│   ├── Daftar dokumen
│   ├── Upload dokumen          ← Mahasiswa (+ Admin boleh bantu)
│   ├── Download
│   └── Hapus                   ← Pemilik / Admin
```

---

## Jenis dokumen

| Kode `type` | Label tampilan |
|-------------|----------------|
| `proposal` | Proposal magang |
| `acceptance_letter` | Surat penerimaan |
| `weekly_report` | Laporan mingguan |
| `final_report` | Laporan akhir |
| `certificate` | Sertifikat |
| `other` | Lainnya |

Aturan jumlah file:

- Boleh **banyak file** per internship (termasuk type sama), **atau**
- Satu file per `type` (upload baru menimpa)  

**Keputusan project ini:** boleh banyak file; bedakan lewat `title` + tanggal upload. Lebih fleksibel untuk laporan mingguan.

---

## Field `documents`

| Field | Tipe | Keterangan |
|-------|------|------------|
| internship_id | FK | Penempatan |
| uploaded_by | FK users | Siapa yang upload |
| type | string | Lihat tabel jenis |
| title | string | Judul dokumen |
| file_path | string | Path relatif di storage |
| original_name | string | Nama file asli |
| mime | string | MIME type |
| size | unsignedInteger | Byte |
| timestamps | | |

---

## Aturan akses

| Aksi | Admin | Mahasiswa pemilik | Pembimbing assign |
|------|:-----:|:-----------------:|:-----------------:|
| Lihat daftar | ✓ semua | ✓ milik | ✓ bimbingan |
| Upload | ✓ (boleh bantu) | ✓ | — |
| Download | ✓ | ✓ | ✓ |
| Hapus | ✓ | ✓ milik sendiri | — |

Aturan lain:

1. Upload hanya jika internship `active` (atau `completed` untuk arsip — **disarankan: active & completed**).
2. Ekstensi: `pdf`, `jpg`, `jpeg`, `png`, `doc`, `docx`.
3. Ukuran max: **2048 KB (2MB)**.
4. Jangan simpan di `public/` tanpa proteksi. Pakai `storage/app/private/documents/...` atau disk `local`.
5. Saat hapus: hapus record **dan** file fisik.

---

## Langkah kerja (urut)

### 1. Migration + model + policy

```bash
php artisan make:model Document -m
php artisan make:policy DocumentPolicy --model=Document
```

```php
Schema::create('documents', function (Blueprint $table) {
    $table->id();
    $table->foreignId('internship_id')->constrained()->cascadeOnDelete();
    $table->foreignId('uploaded_by')->constrained('users')->cascadeOnDelete();
    $table->string('type', 50);
    $table->string('title');
    $table->string('file_path');
    $table->string('original_name');
    $table->string('mime', 100);
    $table->unsignedInteger('size');
    $table->timestamps();

    $table->index(['internship_id', 'type']);
});
```

---

### 2. Relasi

```php
// Document
public function internship() { return $this->belongsTo(Internship::class); }
public function uploader() { return $this->belongsTo(User::class, 'uploaded_by'); }

// Internship
public function documents() { return $this->hasMany(Document::class); }
```

---

### 3. Form upload + validasi

```php
'title' => ['required', 'string', 'max:150'],
'type' => ['required', 'in:proposal,acceptance_letter,weekly_report,final_report,certificate,other'],
'file' => ['required', 'file', 'mimes:pdf,jpg,jpeg,png,doc,docx', 'max:2048'],
```

Form Blade: `enctype="multipart/form-data"`.

---

### 4. Simpan file (private)

Contoh di `store`:

```php
$file = $request->file('file');
$path = $file->store('documents/'.$internship->id, 'local');

Document::create([
    'internship_id' => $internship->id,
    'uploaded_by' => auth()->id(),
    'type' => $request->type,
    'title' => $request->title,
    'file_path' => $path,
    'original_name' => $file->getClientOriginalName(),
    'mime' => $file->getClientMimeType(),
    'size' => $file->getSize(),
]);
```

Pastikan folder writable. Disk `local` default mengarah ke `storage/app/private` di Laravel baru, atau `storage/app` di beberapa versi — cek `config/filesystems.php` dan sesuaikan.

**Jangan** pakai `storePublicly` untuk dokumen sensitif.

---

### 5. Daftar dokumen

Index scoped mirip internship:

- Mahasiswa: dokumen internship miliknya
- Pembimbing: dokumen internship bimbingan
- Admin: semua + filter

Tampilkan: judul, jenis, nama file, ukuran (KB), tanggal, tombol unduh/hapus.

Format ukuran:

```php
number_format($document->size / 1024, 1) . ' KB'
```

---

### 6. Download terproteksi

```text
GET /documents/{document}/download
```

```php
public function download(Document $document)
{
    $this->authorize('download', $document);

    return Storage::disk('local')->download(
        $document->file_path,
        $document->original_name
    );
}
```

Jangan expose path mentah ke user.

---

### 7. Hapus dokumen

```php
public function destroy(Document $document)
{
    $this->authorize('delete', $document);

    Storage::disk('local')->delete($document->file_path);
    $document->delete();

    return back()->with('success', 'Dokumen dihapus.');
}
```

Konfirmasi di UI sebelum hapus.

---

### 8. Policy singkat

```php
public function download(User $user, Document $document): bool
{
    $internship = $document->internship;

    return $user->isAdmin()
        || $internship->student_id === $user->id
        || $internship->industrial_supervisor_id === $user->id;
}
```

Mirip untuk `view`, `delete` (delete: admin atau student pemilik).

---

## Checklist selesai

- [ ] Migration `documents` jalan
- [ ] Upload berhasil untuk tipe yang diizinkan
- [ ] File > 2MB ditolak
- [ ] Ekstensi ilegal ditolak
- [ ] File tersimpan di disk privat (bukan URL publik langsung)
- [ ] Download hanya untuk yang berhak
- [ ] Hapus menghilangkan file + record
- [ ] Pembimbing hanya lihat dokumen bimbingan
- [ ] Form memakai `multipart/form-data`

---

## Cara uji cepat

1. Login mahasiswa → upload PDF laporan → muncul di daftar  
2. Klik unduh → file terbuka/terunduh  
3. Coba upload `.exe` / file 10MB → ditolak  
4. Login industri → bisa unduh dokumen mahasiswa bimbingan  
5. Login pembimbing lain → tidak bisa unduh (403)  
6. Hapus dokumen → file hilang dari storage  

---

## Kesalahan umum

1. **Lupa `enctype` di form** — file tidak terkirim.
2. **Simpan di `public/` lalu link langsung** — siapa saja yang tahu URL bisa unduh.
3. **Hapus DB saja** — storage penuh file yatim.
4. **`mimes` vs `mimetypes`** — ikuti rule Laravel; uji dengan file nyata.
5. **Path traversal di nama file** — pakai `store()` + `original_name` hanya untuk label download, bukan path.

---

**Lanjut ke:** [Tahap 07 — Penilaian](07-penilaian.md)
