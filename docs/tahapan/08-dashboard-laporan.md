# Tahap 08 — Dashboard & Laporan

**Urutan:** 08 dari 09  
**Estimasi:** 5–7 jam  
**Tujuan:** Dashboard berisi angka penting per role; halaman laporan berfilter; export Excel (PhpSpreadsheet) & PDF (DomPDF); notifikasi in-app sederhana.

← Sebelumnya: [07 — Penilaian](07-penilaian.md) · [Indeks](../../README.md) · Berikutnya: [09 — Polish & Testing](09-polish-testing.md) →

---

## Yang dihasilkan di tahap ini

- Dashboard statistik per role (ganti halaman sambutan kosong tahap 02)
- Halaman laporan: rekap log & rekap penilaian
- Filter laporan
- Export Excel via **PhpSpreadsheet**
- Export PDF via **DomPDF**
- Notifikasi in-app (disarankan dikerjakan)

---

## Menu yang aktif

```text
├── Dashboard
│   ├── Statistik ringkas
│   ├── Antrian kerja
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

- Total mahasiswa dengan internship `active`
- Total log minggu ini / bulan ini
- Jumlah log `submitted` (menunggu approval)
- Distribusi status internship (draft/active/completed/cancelled)
- 5 aktivitas / log terbaru (opsional)

### Mahasiswa

- Jumlah log minggu ini
- Log `rejected` yang perlu revisi
- Jumlah dokumen terunggah
- Nilai akhir (jika assessment final)
- Status penempatan aktif

### Pembimbing Industri

- Antrian log `submitted` bimbingan
- Jumlah mahasiswa bimbingan aktif
- Penilaian yang belum final
- Link cepat ke antrian approval

**Query tips:** hitung dengan `select count` / aggregate; hindari N+1; cache tidak wajib.

Contoh “minggu ini”:

```php
use Illuminate\Support\Carbon;

$start = Carbon::now()->startOfWeek();
$end = Carbon::now()->endOfWeek();

ActivityLog::whereBetween('activity_date', [$start, $end])->count();
```

---

## Laporan

### 1. Rekap log kegiatan

Filter GET:

- `period_id`
- `company_id`
- `student_id` (admin)
- `date_from` / `date_to`
- `status`

Kolom tabel: tanggal, nama mahasiswa, perusahaan, judul, status, jam kerja.

### 2. Rekap penilaian

Filter: periode, perusahaan, status final.

Kolom: mahasiswa, perusahaan, nilai akhir, status final, nama pembimbing.

### Hak akses laporan

| Role | Cakupan data |
|------|----------------|
| Admin | Semua |
| Pembimbing | Hanya bimbingan |
| Mahasiswa | Hanya milik sendiri |

Export memakai **filter yang sama** dengan yang sedang aktif di halaman.

---

## Package export

```bash
composer require phpoffice/phpspreadsheet
composer require barryvdh/laravel-dompdf
```

Opsional publish config DomPDF:

```bash
php artisan vendor:publish --provider="Barryvdh\DomPDF\ServiceProvider"
```

**Jangan** pasang `maatwebsite/excel`.

---

## Notifikasi in-app (disarankan)

### Event

| Event | Penerima |
|-------|----------|
| Log baru `submitted` | Pembimbing industri terkait |
| Log di-approve / di-reject | Mahasiswa |
| Nilai di-finalisasi | Mahasiswa |

### Tabel `notifications` (custom sederhana)

| Field | Tipe |
|-------|------|
| user_id | FK |
| title | string |
| message | text |
| url | string, nullable |
| is_read | boolean default false |
| timestamps | |

Atau pakai database notification Laravel (`php artisan notifications:table`) — pilih **satu**, jangan campur.

UI: ikon lonceng di navbar + badge jumlah belum dibaca + halaman daftar + mark as read.

Buat notifikasi di tempat event terjadi (setelah approve, reject, submit, finalize) agar tidak terlupa.

---

## Langkah kerja (urut)

### 1. Service / method statistik dashboard

Boleh buat class:

```text
app/Services/DashboardService.php
```

Method: `forAdmin()`, `forMahasiswa(User $user)`, `forPembimbing(User $user)`.

Controller dashboard tahap 02 diisi data dari service ini.

---

### 2. Rapikan Blade dashboard

- Kartu angka Bootstrap (`row` + `col` + `border rounded bg-white p-3`)
- Daftar singkat di bawah (antrian / log terbaru)
- Link “Lihat semua”

---

### 3. Halaman laporan + filter

```bash
php artisan make:controller ReportController
```

Route contoh:

```text
GET /laporan/log
GET /laporan/penilaian
GET /laporan/log/export-excel
GET /laporan/log/export-pdf
GET /laporan/penilaian/export-excel
GET /laporan/penilaian/export-pdf
```

Form filter method GET. Tombol export menyertakan query string yang sama.

Scope query mengikuti role (sama seperti index internship/log).

---

### 4. Export Excel (PhpSpreadsheet)

Contoh pola di controller:

```php
use PhpOffice\PhpSpreadsheet\Spreadsheet;
use PhpOffice\PhpSpreadsheet\Writer\Xlsx;
use Symfony\Component\HttpFoundation\StreamedResponse;

public function exportLogExcel(Request $request): StreamedResponse
{
    $rows = $this->logQuery($request)->get(); // query berfilter + scoped

    $spreadsheet = new Spreadsheet();
    $sheet = $spreadsheet->getActiveSheet();
    $sheet->fromArray(['Tanggal', 'Mahasiswa', 'Judul', 'Status', 'Jam'], null, 'A1');

    $r = 2;
    foreach ($rows as $log) {
        $sheet->fromArray([
            $log->activity_date->format('Y-m-d'),
            $log->internship->student->name,
            $log->title,
            $log->status,
            $log->work_hours,
        ], null, 'A'.$r);
        $r++;
    }

    $writer = new Xlsx($spreadsheet);

    return response()->streamDownload(function () use ($writer) {
        $writer->save('php://output');
    }, 'rekap-log.xlsx', [
        'Content-Type' => 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    ]);
}
```

---

### 5. Export PDF (DomPDF)

Buat view `resources/views/reports/log-pdf.blade.php` (HTML sederhana: judul, tanggal cetak, tabel). Hindari CSS Flex/Grid rumit — DomPDF lebih cocok dengan tabel klasik.

```php
use Barryvdh\DomPDF\Facade\Pdf;

public function exportLogPdf(Request $request)
{
    $rows = $this->logQuery($request)->get();

    $pdf = Pdf::loadView('reports.log-pdf', [
        'rows' => $rows,
        'generatedAt' => now(),
    ])->setPaper('a4', 'landscape');

    return $pdf->download('rekap-log.pdf');
}
```

---

### 6. Notifikasi

- Migration + model `AppNotification` (atau nama `Notification` custom hati-hati bentrok)
- Helper `notify(User $user, string $title, string $message, ?string $url)`
- Panggil dari Action log/assessment
- Controller: index, markRead, markAllRead
- Navbar badge

---

## Checklist selesai

- [ ] Dashboard admin menampilkan angka masuk akal
- [ ] Dashboard mahasiswa & pembimbing relevan
- [ ] Filter laporan log berfungsi
- [ ] Filter laporan penilaian berfungsi
- [ ] Data laporan ter-scope per role
- [ ] Export Excel terunduh & bisa dibuka
- [ ] Export PDF terunduh & terbaca
- [ ] Package yang terpasang: PhpSpreadsheet + DomPDF (bukan Maatwebsite)
- [ ] Notifikasi dasar muncul untuk submit/approve/reject/final nilai (jika dikerjakan)
- [ ] Query dashboard tidak N+1 parah

---

## Cara uji cepat

1. Login admin → cek angka statistik vs data aktual di DB  
2. Filter laporan per periode → isi tabel berubah  
3. Export Excel & PDF → buka file, kolom sesuai  
4. Login mahasiswa → laporan hanya miliknya  
5. Submit log baru → pembimbing mendapat notifikasi (jika fitur aktif)  

---

## Kesalahan umum

1. **Export tanpa authorize/scope** — kebocoran data.
2. **Memasang Maatwebsite “karena tutorial”** — tidak sesuai stack project.
3. **View PDF pakai class Bootstrap kompleks** — layout pecah; sederhanakan HTML.
4. **Hitung statistik di Blade dengan query per kartu** — pindahkan ke service, 1–beberapa query saja.
5. **Lupa eager load saat export** — error relasi null / lambat.

---

**Lanjut ke:** [Tahap 09 — Polish & Testing](09-polish-testing.md)
