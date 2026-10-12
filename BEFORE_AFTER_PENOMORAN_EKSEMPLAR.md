# Laporan Pekerjaan: 5-6 Oktober 2026

Dokumen ini menggabungkan laporan pekerjaan bertanggal dengan before/after penomoran eksemplar yang sudah ada. Tanggal pekerjaan didasarkan pada waktu perubahan file lokal (WIB); perubahan laporan baru dicatat dalam Git pada **7 Oktober 2026** melalui commit `c945639`. Tanggal commit tidak otomatis sama dengan tanggal pengerjaan.

## 5 Oktober 2026

Tidak ditemukan perubahan file proyek bertanggal 5 Oktober atau commit Git pada tanggal tersebut dalam data lokal yang tersedia. Karena itu belum ada pekerjaan yang dapat diatribusikan secara pasti ke tanggal ini. Bagian ini perlu dilengkapi bila tersedia catatan pekerjaan lain.

## 6 Oktober 2026

### 1. Penyempurnaan filter Laporan Anggota

File: `app/Modules/Laporan/LaporanAnggota/Controllers/LaporanAnggota.php:404` dan `app/Modules/Laporan/LaporanAnggota/Views/index.php:173`.

| Sebelum | Sesudah |
| --- | --- |
| Form menampilkan filter tanggal, bulan-tahun, dan tahun sebagai bagian terpisah. | Pengguna memilih satu periode: tanggal, bulan, atau tahun. Hanya input periode terpilih yang ditampilkan. |
| Rentang tanggal diterapkan hanya bila tanggal mulai **dan** akhir terisi. | Tanggal mulai/akhir bisa diterapkan sendiri-sendiri; batas akhir dihitung sampai sebelum hari berikutnya agar seluruh hari akhir tercakup. |
| Kriteria lanjutan tampil sebagai banyak input tetap. | Kriteria disusun dalam baris yang dapat ditambahkan dan dipilih, sehingga form lebih rapi. |

Alur preview dan ekspor memakai filter controller yang sama. Jejak perubahan lokal: controller **20.51 WIB**, view **21.00 WIB**.

### 2. Penyempurnaan filter Laporan Eksemplar

File: `app/Modules/Laporan/LaporanEksemplar/Controllers/LaporanEksemplar.php:375` dan `app/Modules/Laporan/LaporanEksemplar/Views/index.php:157`.

| Sebelum | Sesudah |
| --- | --- |
| Tanggal dibuat, bulan-tahun, dan tahun tersedia sebagai beberapa bagian filter terpisah. | Pilihan tanggal, bulan, atau tahun menentukan satu bagian periode yang aktif. |
| Rentang tanggal dibuat membutuhkan dua batas; batas akhir berisiko tidak mencakup data pada jam setelah tengah malam. | Batas awal/akhir dapat diisi terpisah dan batas akhir dihitung sebelum hari berikutnya. |
| Kriteria lain ditampilkan sebagai kumpulan field tetap. | Kriteria lanjutan disusun sebagai baris pilihan yang dapat ditambahkan sesuai kebutuhan. |

Filter lain, termasuk Tanggal Pengadaan, tetap tersedia. Jejak perubahan lokal: view **21.13 WIB**, controller **21.14 WIB**.

### Catatan verifikasi tanggal

Empat file laporan di atas menunjukkan waktu perubahan pada 6 Oktober 2026; commit `c945639` pada 7 Oktober juga mencakup Laporan Katalog. Laporan Katalog **tidak** dicatat sebagai pekerjaan 6 Oktober karena waktu perubahan file tersebut menunjukkan 7 Oktober. Riwayat percakapan lain tanpa penanda tanggal pasti tidak dipaksakan masuk ke 5 atau 6 Oktober.

## Lampiran: before/after penomoran eksemplar (7 Oktober 2026)

Perbandingan ini memakai versi sebelum dan sesudah commit `68229ba` (7 Oktober 2026). Fokus: pengaturan `NomorInduk = Manual`, sumber nomor barcode/RFID, dan preview pada form.

### Ringkasan

| Bagian | Sebelum | Sesudah |
| --- | --- | --- |
| Create: input manual | No. Induk, barcode, dan RFID tampak bisa diketik. | Hanya No. Induk yang bisa diketik; barcode/RFID `readonly`. |
| Create: sumber No. Induk | No. Induk diambil dari input. | Tetap dari input No. Induk. |
| Create: sumber `Item ID` | Memperkirakan ID dari `MAX(collections.ID) + urutan`; bisa keliru bila ada penyimpanan bersamaan. | Memakai ID hasil insert (`insertID()`), lalu memperbarui barcode/RFID berdasarkan sumber masing-masing. |
| Edit: barcode/RFID | Controller sudah menghitung nilai berdasarkan sumber, tetapi kedua input di form masih bisa diketik. | Controller tetap menghitung berdasarkan sumber; input `readonly` dan preview mengikuti perubahan No. Induk. |

**Catatan:** Sebelum perubahan, create sempat membaca input barcode/RFID manual, tetapi nilainya kemudian ditimpa oleh hasil perhitungan berdasarkan pengaturan sumber. Jadi input yang diketik tampak bisa diubah, namun bukan nilai akhir yang disimpan.

### Controller: create

`app/Modules/SubModule/Eksemplar/Controllers/EksemplarFormController.php:23`

Sebelum (potongan alur):

```php
$lastCollectionId = (int) ($this->db->table('collections')
    ->selectMax('ID', 'last_id')->get()->getRow()->last_id ?? 0);

$noInduk = $post['NoInduk' . $idx] ?? '';
$nomorBarcode = $post['NomorBarcode' . $idx] ?? '';
$rfid = $post['RFID' . $idx] ?? '';

$itemId = str_pad((string) ($lastCollectionId + $i), 11, '0', STR_PAD_LEFT);
$nomorBarcode = ($barcodeSource === 'Item ID') ? $itemId : $noInduk;
$rfid = ($rfidSource === 'Item ID') ? $itemId : $noInduk;
```

Sesudah (potongan alur):

```php
$lock = $this->db->query('SELECT GET_LOCK(?, 30) AS acquired', [$lockName])->getRow();
$this->db->transBegin();

$noInduk = $post['NoInduk' . $idx] ?? '';
$this->db->table('collections')->insert($save);
$itemId = $this->db->insertID();
$itemNumber = str_pad((string) $itemId, 11, '0', STR_PAD_LEFT);
$nomorBarcode = ($barcodeSource === 'Item ID') ? $itemNumber : $noInduk;
$rfid = ($rfidSource === 'Item ID') ? $itemNumber : $noInduk;
$this->db->table('collections')->where('ID', $itemId)->update([
    'NomorBarcode' => $nomorBarcode,
    'RFID' => $rfid,
]);

$this->db->transCommit();
```

Potongan sesudah disingkat untuk dokumentasi; kode asli juga memeriksa hasil lock, insert, update, dan transaksi, lalu melepas lock.

### Controller: edit

`app/Modules/SubModule/Eksemplar/Controllers/EksemplarFormController.php:224`

Sebelum dan sesudah, edit menentukan nilai akhir dari sumber yang dikonfigurasi, bukan dari input barcode/RFID. Perubahan controller menyiapkan sumber sekali untuk dipakai oleh tampilan dan penyimpanan:

```php
$this->data['barcodeUsesItemId'] = $this->getNumberSource('FormatNomorBarcode') === 'Item ID';
$this->data['rfidUsesItemId'] = $this->getNumberSource('FormatNomorRFID') === 'Item ID';

$noInduk = isset($post['NoInduk0']) ? $post['NoInduk0']
    : (isset($post['NoInduk']) ? $post['NoInduk'] : $eksemplar->NoInduk);
$itemId = str_pad((string) $eksemplar->ID, 11, '0', STR_PAD_LEFT);
$nomorBarcode = $this->data['barcodeUsesItemId'] ? $itemId : $noInduk;
$rfid = $this->data['rfidUsesItemId'] ? $itemId : $noInduk;
```

### Tampilan: create dan edit

`app/Modules/SubModule/Eksemplar/Views/add.php:554` dan `app/Modules/SubModule/Eksemplar/Views/update.php:503`

Sebelum: input barcode/RFID pada mode manual bisa diketik; perubahan No. Induk tidak memperbarui preview pada form edit.

Sesudah: input barcode/RFID `readonly`. Pada create, jika sumbernya No. Induk, preview mengikuti input No. Induk; jika sumbernya `Item ID`, placeholder menjelaskan nomor dibuat saat disimpan. Pada edit, preview memperbarui kedua nilai sesuai sumber masing-masing setiap kali No. Induk berubah.

| Sumber yang dipilih | Nilai tersimpan |
| --- | --- |
| Barcode = `Item ID` | ID eksemplar, diisi nol hingga 11 digit. |
| Barcode = No. Induk | No. Induk eksemplar. |
| RFID = `Item ID` | ID eksemplar, diisi nol hingga 11 digit. |
| RFID = No. Induk | No. Induk eksemplar. |

Sumber barcode dan RFID dipilih terpisah. Pada mode manual, field dengan sumber `Item ID` menampilkan placeholder, bukan angka perkiraan. Preview nomor pada mode otomatis bisa berubah bila eksemplar lain tersimpan lebih dulu; ID final baru diketahui setelah insert berhasil.
