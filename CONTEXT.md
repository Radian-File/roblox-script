# CONTEXT.md — Konteks Pengembangan Pabrik Growblox

> **Terakhir disusun:** 8 November 2026  
> **Sumber:** pembacaan statis source aktif `k .lua` dan laporan pengujian pengguna.  
> **Status:** roadmap Pabrik nomor 0-9 telah dipatch secara statis. Semua flow, storage, dan collector masih memerlukan pengujian runtime terotorisasi sebelum dinyatakan reliabel.

## 1. Tujuan pengguna

Pengguna ingin memperbaiki **Pabrik** yang terasa buggy dan sulit diprediksi. Target utamanya:

1. Membuat urutan kerja yang eksplisit: **BREAK → PLANT → WAIT_GROWTH → HARVEST → REPEAT**.
2. Memastikan fase **HARVEST memindai semua pohon target** sesuai area pengaturan (`WholeWorld` bila diaktifkan) dan tidak meninggalkan pohon matang yang layak dipanen karena inventory hampir penuh.
3. Menambahkan **Block Storage Point** untuk menaruh block hasil panen ketika inventory mendekati penuh, kemudian kembali melanjutkan Harvest.
4. Menambahkan **Auto Refill Blocks** untuk mengambil kembali block dari lokasi penyimpanan ketika fase Break kekurangan block.
5. Mempertahankan kompatibilitas fitur lain dan menghindari loop tanpa progress.

Pengguna ingin menyelesaikan perubahan kode melalui **Codex**. Filter platform tanam telah menjadi patch awal; pekerjaan berikutnya tetap harus mengikuti rencana bertahap dan persetujuan sebelum mengubah flow/state Pabrik.

## 2. Gejala yang dilaporkan pengguna

- Urutan fase tidak jelas: kadang `plant → break → harvest`, bukan flow yang diinginkan.
- Pabrik tampak menangani pohon yang **belum matang** dalam sebagian sesi.
- Harvest dapat berhenti sebelum seluruh pekerjaan selesai.
- Kapasitas inventory mengganggu siklus.

Ini adalah **laporan runtime pengguna**. Penyebab spesifik pohon muda terkena aksi masih perlu bukti log dan pengamatan di world.

## 3. Kondisi implementasi yang dikonfirmasi dari source

| Komponen | Fakta terkonfirmasi | Lokasi perkiraan |
| --- | --- | --- |
| Inisialisasi fase | `pabrikStep(o)` memakai `o.phase = o.phase or "panen"` | `k.lua:5731–5735` |
| Mode Pabrik normal | `pabrikLoop` membuat `o.breakMode = "always"` | `k.lua:5809–5811` |
| PANEN → TANAM | `fieldStep()` mengembalikan `idle`, atau block/seed mencapai `CAP-20` | `k.lua:5743–5756` |
| TANAM → BREAK | Pindah setelah pekerjaan tanam menjadi `idle` atau seed tidak ada | `k.lua:5759–5767` |
| BREAK → BREAK/TANAM | Setelah `pabrikBreak()`, masih ada block berarti tetap BREAK; selain itu TANAM | `k.lua:5769–5782` |
| TUNGGU | Dapat berpindah ke PANEN, BREAK, atau TANAM berdasarkan kondisi dan timer | `k.lua:5784–5800` |
| BREAK memanggil Auto Farm | `State.Override = {... mode="Place + Break"}` lalu `farmLoop` | `k.lua:5493–5544` |
| Watchdog BREAK | Menghentikan sesi jika seed mencapai ambang, drop perlu dilakukan, atau tidak ada perubahan jumlah Placed/Broken selama 8 detik | `k.lua:5515–5528` |
| Nilai `BreakSec` | `Pabrik.BreakSec = 60` dideklarasikan tetapi tidak ditemukan dipakai di kontrol sesi BREAK saat ini | `k.lua:194` |
| Metadata pohon | `treeReadyAt()` memakai `t.p + (it.grow or 0)` | `k.lua:4570–4578` |
| Filter jenis panen | `farmerScan()` mendukung `FC.only`, tetapi `pabrikCfg()` tidak mengisinya | `k.lua:5467–5490`, `4750` |
| Batas stock | `CAP=190`, PANEN mengecek `CAP-20=170`, default `Pabrik.DropAt=200`, `Keep=50` | `k.lua:187–208`, `5465`, `5746` |
| Drop yang tersedia | `dropItems()`, `dropTrip()`, `Pabrik.Zone` sudah ada, terutama dipakai untuk drop seed | `k.lua:5570–5685` |
| Zone collector | Sebagian collector mengecualikan radius sekitar titik `Pabrik.Zone` | `k.lua:2134–2161`, `3771–3794` |
| Konfigurasi Pabrik | `Config.add("Pabrik", ...)` menyimpan seed, area, posisi/grid break, drop seed | `k.lua:1166–1169` |

**Penting:** `Pabrik.start()` menonaktifkan beberapa fitur lain, tetapi tidak terlihat menonaktifkan `Build` dan `Vend` pada jalur start yang diperiksa (`k.lua:5872–5910`). Periksa kembali konsekuensi kontrol gerak bersama sebelum implementasi.

## 4. Hipotesis yang perlu diuji

| Gejala | Kandidat penyebab | Apa yang harus diamati |
| --- | --- | --- |
| Pohon muda terkena aksi | Break Grid beririsan dengan tile pohon; `tryBreak()` dijalankan tanpa gate kematangan yang sama dengan jalur Harvest | Koordinat Break Grid, ID item, mode aksi, kematangan saat pukulan |
| Pohon dianggap siap terlalu cepat | `it.grow` hilang sehingga fallback `0`, atau `Refs.Trees` stale/tidak lengkap | Waktu tanam, `it.grow`, waktu server, data tree entry |
| Harvest berhenti terlalu awal | Transisi saat block/seed `>=170` langsung ke TANAM | Inventory sebelum transisi, count pohon mature vs immature |
| BREAK loop/stall | Retain BREAK selagi block `>0`, watchdog hanya melihat Placed/Broken, atau setpoint drop tidak sinkron | `from/to phase`, jumlah hits, Placed/Broken, inventory, alasan stop |
| Drop terkumpul ulang | Area storage dekat lintasan gerak, collector lain atau native pickup masih aktif | Posisi player, posisi drop, collector aktif, perubahan inventory |
| Klaim aksi sukses tidak sesuai server | `plantSeed()` memanggil `sendSetTile()` tetapi tidak memeriksa hasilnya sebelum mengurangi inventory lokal dan menaikkan counter | Perubahan tile/inventory server setelah operasi, error jaringan |

Jangan menyatakan hipotesis sebagai penyebab pasti sebelum ada log atau tes yang memadai.

## 5. Desain target nomor 1-8 (telah dipatch secara statis, belum diuji runtime)

```mermaid
stateDiagram-v2
    [*] --> VALIDATE
    VALIDATE --> BREAK: siap & stok tersedia
    VALIDATE --> BOOTSTRAP: block tidak ada, seed tersedia (opsional)
    BREAK --> PLANT: batch break selesai
    PLANT --> WAIT_GROWTH: target tanam diproses
    WAIT_GROWTH --> HARVEST: ada pohon target matang
    HARVEST --> WAIT_GROWTH: masih ada pohon batch belum matang
    HARVEST --> BREAK: semua pohon target batch selesai & drop ditangani
    BREAK --> REFILL_BLOCK: block di bawah ambang & stok ada
    REFILL_BLOCK --> BREAK: pickup terkonfirmasi
    HARVEST --> STORE_BLOCK: block mendekati penuh
    STORE_BLOCK --> HARVEST: drop terkonfirmasi
    BREAK --> PAUSED: kegagalan tidak bisa dipulihkan
    PLANT --> PAUSED: kegagalan tidak bisa dipulihkan
    HARVEST --> PAUSED: kegagalan tidak bisa dipulihkan
    PAUSED --> VALIDATE: dipulihkan / restart terkontrol
```

Flow ini **rancangan**, bukan fungsi yang sudah ada di `k.lua`.

### Kontrak fase yang diusulkan

| Fase | Prasyarat | Selesai jika | Jangan lakukan |
| --- | --- | --- | --- |
| `VALIDATE` | Data world, seed, area, path & target tersedia | Semua validasi minimum lulus | Melanjutkan dengan referensi stale |
| `BREAK` | Block tersedia (inventory atau stock) dan grid valid | Batas batch yang disepakati tercapai; tidak hanya bergantung pada `blocks > 0` | Memukul pohon belum matang atau loop gagal tanpa batas |
| `REFILL_BLOCK` | Stok yang diketahui tersedia dan jalur ke storage dapat ditempuh | Jumlah block yang diambil tervalidasi | Mengambil semua item sekitar storage secara membabi buta |
| `PLANT` | Ada seed dan tempat tanam valid | Semua tile valid dalam batch telah diproses atau punya alasan skip | Menghitung penanaman gagal sebagai sukses |
| `WAIT_GROWTH` | Ada target batch yang belum matang | Minimal satu target matang dan data valid | Menganggap tree data hilang = matang |
| `HARVEST` | Ada target matang yang benar, path dapat ditempuh | Semua target matang ditangani; bila target belum matang tersisa, kembali menunggu | Keluar hanya karena inventory hampir penuh |
| `STORE_BLOCK` | Storage valid dan block mencapai ambang | Drop terkonfirmasi dan state Harvest dapat dipulihkan | Mengubah fase utama menjadi Plant/Break |
| `PAUSED/ERROR` | Terjadi kondisi yang tidak bisa dipastikan/ditangani | Pemulihan eksplisit | Spin tanpa progress dan tanpa status |

### Kebijakan 'Harvest seluruh world'

- Jika `Pabrik.WholeWorld = true`, pemindaian memakai batas world; jika false, area hanya wilayah lahan.
- Prioritaskan **hanya jenis seed Pabrik**; pohon lain tidak boleh masuk target yang sama tanpa keputusan desain baru.
- Pisahkan hitungan `ready`, `growing`, `unreachable`, `unknown` (data pohon tidak valid), dan `completed`.
- Pengguna menginginkan siklus batch yang dapat dituntaskan: **jangan mulai BREAK baru ketika masih ada pohon target dalam batch yang belum matang**, kecuali pengguna nanti memilih mode produksi yang overlapping.
- Bila masih ada pohon siap tetapi unreachable, jangan menandai fase selesai secara diam-diam: retry terbatas lalu tampilkan kondisi khusus.
- Simpan informasi batch/target agar pindah lokasi dan interupsi storage tidak membuat target hilang atau terproses dua kali.

## 6. Rencana Block Storage (belum ada)

### A. `Set Block Storage Point`

- Opsi UI untuk menyimpan world serta koordinat titik drop block.
- Dipisahkan dari `Pabrik.DropWorld`, `DropX`, `DropY` yang saat ini ditujukan untuk seed.
- Pilih lokasi dekat BREAK namun **bukan** pada Break Grid, jalur karakter, atau tile yang membuat drop terkubur/otomatis terambil.
- Pastikan data storage tervalidasi setelah pindah world/rejoin.

### B. `Auto Store Blocks`

Saat HARVEST dan block mencapai ambang batas:

1. Suspend pekerjaan Harvest dan simpan konteks fase/target.
2. Lepas kontrol gerak sebelumnya, pergi ke Block Storage Point.
3. Drop hanya **ID block terpilih**, sejumlah `jumlah - keep`, dengan mekanisme yang telah diaudit.
4. Verifikasi perubahan inventory dan keberadaan drop bila datanya bisa diamati.
5. Kembali ke area pekerjaan; **resume HARVEST**.
6. Jika gagal: tampilkan error/retry terbatas, **jangan** lompat ke fase lain.

Ambang contoh dari diskusi: drop saat block `>=170`, sisakan `30`, jadi bila berisi 170 akan mencoba menyimpan 140. **Angka ini usulan, belum menjadi spesifikasi final**.

### C. `Auto Refill Blocks`

Saat BREAK dan block mendekati habis:

1. Suspend BREAK (bukan menandainya selesai).
2. Periksa apakah ada stok block yang benar di titik storage.
3. Menuju titik storage; ambil **hanya item ID target** sampai jumlah yang dibutuhkan.
4. Verifikasi kuantitas dan bahwa drop tidak terambil dua kali.
5. Kembali ke BREAK dan lanjutkan batch.
6. Jika stok kosong/hilang/tidak bisa dicapai, tampilkan alasan dan aturan fallback yang eksplisit.

Contoh ambang yang **belum diputuskan**: refill saat `<=10`, target inventory `150`, tidak melewati limit yang berlaku.

### D. Storage zone dan pickup

- Implementasi saat ini memakai `Pabrik.Zone` tunggal untuk mencegah pengambilan drop pada sebagian collector. Desain baru kemungkinan membutuhkan `SeedDropZone` dan `BlockStorageZone` terpisah atau daftar zona bertipe.
- Saat restock, beri izin **sementara** untuk mengambil block tertentu yang telah dipilih di storage, lalu kembalikan proteksi zone.
- Jarak aman dari Break Grid lebih penting daripada menaruh drop tepat di tile break, karena native pickup saat bersentuhan belum tentu dapat dinonaktifkan.
- **Jangan berasumsi** item drop selalu persisten, memiliki kepemilikan eksklusif, atau tidak bisa diambil orang lain.

## 7. Catatan atas keputusan yang masih terbuka

| Keputusan | Kondisi saat ini |
| --- | --- |
| Siklus batch harus menunggu seluruh pohon target matang? | **Arah preferensi:** ya; perlu konfirmasi perilaku untuk tree unreachable/missing data |
| Awal Pabrik ketika hanya ada seed dan tidak ada block? | Pertimbangkan `BOOTSTRAP` terpisah, jangan membingungkan urutan normal |
| Definisi BREAK selesai | Belum disepakati: target seed, jumlah block, waktu, atau kombinasi |
| Ambang Store/Refill dan Keep | Angka contoh ada, **belum final** |
| Storage di world sama/berbeda? | Preferensi praktis: dekat BREAK, tetapi di luar jalur dan grid aktif |
| Cara mengidentifikasi drop simpanan | Harus ditentukan berdasarkan data drop yang benar-benar tersedia |
| Stock drop hilang, dijarah, atau terambil otomatis | Perlu kebijakan fallback, retry dan notifikasi |
| Penanganan pohon mature tetapi tidak terjangkau | Retry terbatas, status jelas, tidak silently complete |
| Batch yang menunggu lama | Tetap `WAIT_GROWTH` atau dukung mode overlapping sebagai opsi terpisah nanti |

## 8. Risiko lintas modul

- **Shared `FC`:** Farmer dan Pabrik memakai mesin tanam/panen yang sama; perubahan filter dan scan harus diuji agar Farmer tidak rusak.
- **Shared `farmLoop`:** Pabrik BREAK mengisi `State.Override` dan memakai Auto Farm; perubahan status/loop berdampak pada Auto Farm biasa.
- **Shared movement:** `AutoInput` dipakai beberapa controller; perjalanan storage berpotensi bertabrakan dengan Build, Vend, Farmer, Clear, dan Guard.
- **Shared drop:** `Pabrik.Zone` juga dibaca oleh collector lain; modifikasi zone perlu mempertimbangkan Farmer/Auto Farm.
- **Keandalan server:** request place/break/collect/drop tidak boleh disamakan dengan bukti sukses; cek data aktual.
- **Perubahan world:** `Refs.World`, `Refs.Trees`, inventory, dan data drop bisa berubah ketika world/rejoin.
- **Cleanup:** pekerjaan background harus selesai atau batal ketika `Pabrik.stop()` atau `Session.Cleanup` dipanggil.

## 9. Rencana implementasi bertahap di Codex (jika diminta)

> **Catatan status:** uraian Tahap 0 di bawah adalah baseline dokumentasi historis. Sesudah baseline tersebut, nomor 9 (*Selected Plant Platform*) telah dipatch secara statis. Tahap 1-5 untuk flow, harvest/growth, storage, dan regresi tetap belum diimplementasikan atau disetujui.

**Tahap 0 — hanya dokumentasi:** `README.md`, `AGENTS.md`, dan `CONTEXT.md`. **Tidak mengubah `k.lua`.**

**Tahap 1 — observability:** instrumentasi transisi fase, jumlah block/seed, alasan penundaan, tree state, dan kegagalan navigasi/aksi.

**Tahap 2 — state machine:** eksplisitkan `VALIDATE`, `BREAK`, `PLANT`, `WAIT_GROWTH`, `HARVEST`, `PAUSED/ERROR` dan aturan resume; pastikan siklus tidak berubah acak.

**Tahap 3 — keandalan Harvest:** filter target seed, proteksi tree muda, hasil panen/penanaman terverifikasi, retry bounded.

**Tahap 4 — Block Storage:** UI/konfigurasi titik, zone proteksi, drop/resume Harvest, refill/resume Break.

**Tahap 5 — regresi:** uji flow end-to-end, stop, world transition, inventory penuh/kosong, drop hilang, path gagal, tree stale.

Tahapan 1–5 **belum dikerjakan atau disetujui untuk dijalankan** berdasarkan permintaan dokumentasi saat ini.

## 10. Matriks uji yang disarankan

| Kasus | Ekspektasi |
| --- | --- |
| Start dengan block + seed valid | Mulai dari BREAK pada flow normal |
| Start tanpa block tetapi ada seed | Masuk fallback bootstrap yang eksplisit, jika disetujui |
| BREAK stok menipis | REFILL_BLOCK → kembali ke BREAK tanpa reset batch |
| HARVEST inventory mencapai ambang | STORE_BLOCK → kembali ke HARVEST pada target yang belum selesai |
| Semua pohon matang | Harvest semua target terjangkau dan terverifikasi |
| Pohon muda masih ada | Jangan dirusak; kembali `WAIT_GROWTH` |
| Pohon jenis lain ada di world | Tidak ikut dalam batch target Pabrik |
| Pohon matang unreachable | Retry bounded, tampilkan status; jangan hilangkan dari daftar |
| Data tree tidak tersedia | Jangan mengasumsikan matang; tampilkan status unverified |
| Drop storage terlalu dekat tile break | Tidak tercipta siklus drop-pickup otomatis |
| Stok drop hilang / ID berbeda | Tidak menambah counter stok palsu; fallback jelas |
| Inventory penuh atau kapasitas berubah | Tidak memproses item berlebih secara diam-diam |
| Stop pada navigasi, drop, atau Break | Gerakan/loop/override berhenti dengan aman |
| Rejoin atau pindah world | Referensi diperbarui, state tidak membuat aksi salah world |

## 11. Persyaratan laporan audit selanjutnya

Untuk setiap bug, dokumentasikan: **gejala → bukti fungsi/baris → hipotesis → kondisi reproduksi → tingkat keyakinan → risiko → rekomendasi → hasil uji**. Jika kemudian dilakukan patch, catat fungsi yang berubah serta tes yang benar-benar dijalankan.

### Ringkasan status

- **Ada saat ini:** Auto Farm, navigasi A*, Farmer, Pabrik dengan fase dinamis, drop seed, dan satu zona pengecualian drop.
- **Dilaporkan bermasalah:** urutan fase Pabrik, konsistensi Harvest, pengaruh inventory penuh.
- **Dikonfirmasi dari source:** awal default `panen`, transisi fase dinamis, threshold yang tidak konsisten, filter `FC.only` tidak diatur di `pabrikCfg()`.
- **Belum terverifikasi di runtime:** sumber pasti aksi pada pohon belum matang, sifat pickup/persistensi drop, efek penolakan server.
- **Direncanakan tetapi belum dibuat:** siklus Pabrik deterministik, Block Storage Point, Auto Store Blocks, Auto Refill Blocks, proteksi storage terpisah.

## 12. Pembaruan implementasi - Filter block penyangga Pabrik

**Status:** diimplementasikan secara statis pada `k .lua`; belum diuji runtime Roblox.

- `Pabrik.SupportFilter` dan `Pabrik.SupportId` adalah konfigurasi persisten. Pilihan disimpan sebagai `ItemId`, bukan nama tampilan.
- UI Pabrik menyediakan toggle **Enable Support Block Filter** dan dropdown **Plant Only Above**. Nilai dropdown dibangun dari `ItemDB` untuk block yang ditandai `solid` atau `platform`.
- `pabrikCfg()` memasang predicate `plantOk` hanya pada konfigurasi `FC` milik Pabrik. `farmerScan()` dan `farmerWorkInReach()` hanya menghormatinya bila predicate tersedia; Farmer dan Auto Splice tidak memasangnya sehingga perilakunya tidak berubah.
- `plantSpotOk()` tidak diubah. Sesudah filter penyangga lolos, seluruh validasi lokasi tanam yang sudah ada tetap dijalankan.
- Filter bekerja baik pada Whole World maupun Lahan karena diterapkan saat kandidat tanam dipilih, setelah pembatas area yang sudah ada.
- Bila scan PLANT tidak menemukan posisi yang cocok, fase mengikuti transisi Pabrik yang sudah ada tanpa retry loop tambahan dan status Pabrik menunjukkan `tidak ada posisi cocok saat ini`.
- Fitur ini tidak memodifikasi fase HARVEST, `treeReadyAt()`, atau pemeriksaan kematangan pohon.

## 13. Status roadmap Pabrik saat ini

| No. | Fitur / perbaikan | Status |
| --- | --- | --- |
| 0 | Observability transisi fase dan inventory Pabrik | Diimplementasikan secara statis; belum diuji runtime |
| 1 | Flow deterministik `BREAK -> PLANT -> WAIT_GROWTH -> HARVEST -> REPEAT` | Diimplementasikan secara statis; belum diuji runtime |
| 2 | Harvest seluruh tree target yang dipindai | Diimplementasikan secara statis: filter seed target dan retry target unreachable; belum diuji runtime |
| 3 | Wait until trees mature | Diimplementasikan secara statis: tree growing ditunggu, metadata hilang menjadi `PAUSED`; belum diuji runtime |
| 4 | Set Block Storage Point | Diimplementasikan secara statis: world/koordinat persisten dan validasi dasar; belum diuji runtime |
| 5 | Auto Store Blocks | Diimplementasikan secara statis: suspend HARVEST, drop Item ID block di storage, verifikasi inventory, lalu resume HARVEST; belum diuji runtime |
| 6 | Auto Refill Blocks | Diimplementasikan secara statis: suspend BREAK, collect hanya Item ID block target, verifikasi target refill, lalu resume BREAK; belum diuji runtime |
| 7 | Block Storage Protection | Diimplementasikan secara statis: zona storage terpisah dikecualikan oleh collector script; native pickup belum terverifikasi |
| 8 | Perbaikan inventory threshold dan recovery | Threshold persisten dan pause aman diimplementasikan secara statis; Auto Store/Refill masih terpisah |
| 9 | Selected Plant Platform / Support Block Filter | Diimplementasikan secara statis; belum diuji runtime |
| 10 | Hardening lifecycle, konflik controller, I18N, dan checklist regresi | Diimplementasikan secara statis; belum diuji runtime |

Rencana detail dan urutan pengerjaan berada di `docs/implementation-plans.md`. Status pada dokumen tersebut mencatat patch statis; dokumen itu bukan bukti keberhasilan runtime.

## 14. Status hardening dan verifikasi

- `Pabrik.start()` kini menghentikan Build dan Vending agar tidak ada controller gerak kedua yang aktif.
- `Pabrik.stop()` dan `Session.Cleanup` mereset `Move.NoZone`, sehingga izin pickup storage sementara tidak bocor ke sesi berikutnya.
- Label konfigurasi storage/inventory mengikuti kamus I18N Indonesia/English.
- Pemeriksaan statis dilakukan untuk alur state, konfigurasi persisten, helper storage, dan call-site collector. Tidak ada executor, remote game, atau tes Roblox runtime yang dijalankan.
- Uji runtime terotorisasi yang masih wajib: success/failure place-break, kematangan tree, storage drop hilang/diambil pemain lain, refill partial, world transition, stop di tengah navigasi, dan script re-execution.

## 15. WindUI loader (8 November 2026)

- Loader WindUI pada `k .lua` memakai distribusi raw resmi Footagesus yang dipin ke tag `1.6.66`: `https://raw.githubusercontent.com/Footagesus/WindUI/1.6.66/dist/main.lua`. Endpoint asset rilis resmi untuk tag yang sama pernah mengembalikan `504 Gateway Timeout`; endpoint raw tag merespons `200` saat pemeriksaan read-only. Keduanya merujuk distribusi resmi Footagesus.
- Sebelum source remote dikompilasi, loader memeriksa kegagalan HTTP, tipe respons, respons kosong, serta ciri HTML/error page. Respons `504 Gateway Time-out` berbentuk HTML dilaporkan sebagai kegagalan terkendali dan tidak dieksekusi.
- Hasil `loadstring`, eksekusi library, dan keberadaan `CreateWindow` divalidasi. `CreateWindow` juga dibungkus `pcall`; bila gagal, sesi awal dibatalkan sebelum komponen GUI lanjutan dibuat.
- Perubahan ini hanya diverifikasi secara statis. Keberhasilan download dan kompatibilitas runtime dengan executor masih memerlukan pengujian terotorisasi.

## 16. Perbaikan batas inventory Pabrik (8 November 2026)

- Runtime melaporkan `attempt to perform arithmetic (sub) on nil and number` pada kontrol Inventory Pabrik yang menghitung `CAP - 1`. Error tersebut menghentikan konstruksi UI sebelum tab Koneksi dibuat.
- Batas aman Pabrik kini dimiliki oleh `Pabrik.InventoryCap` dengan fallback `190`; slider Pabrik dan seluruh pemeriksaan kapasitas seed Pabrik menghitung nilai ini langsung, bukan lewat `CAP` atau helper lokal lintas-scope.
- `InventoryCap` bukan pengaturan pengguna dan tidak dipersistenkan. Pemakaian `CAP` oleh Build tidak diubah dalam patch ini.
- Patch hanya diperiksa secara statis; buka UI Pabrik lalu tab Koneksi pada runtime terotorisasi untuk verifikasi.

## 17. Temuan runtime Pabrik yang tertunda (8 November 2026)

- **Terkonfirmasi dari stack trace:** updater status Pabrik gagal di `k .lua:11418` saat memanggil `pabrikInventoryPolicy()`, yang pada closure UI terbaca `nil`. Dampaknya, `Status Pabrik` tetap `-` dan tidak lagi menampilkan fase maupun transisi.
- **Dilaporkan pengguna, belum terverifikasi sebagai transisi gagal:** setelah BREAK tampak selesai, Pabrik tidak terlihat memasuki PLANT. Source menyatakan BREAK yang sukses memanggil transisi ke PLANT; karena updater status sudah mati, tampilan fase tidak dapat dipakai sebagai bukti state internal.
- Kemungkinan yang perlu dibedakan setelah observability pulih: BREAK sebenarnya gagal/PAUSED; seed hasil BREAK tidak tersedia sehingga PLANT langsung menuju WAIT_GROWTH; atau tidak ada tile tanam valid (area, Whole World, atau Support Block Filter).
- **Arah desain yang dicatat:** `Finalisasi BREAK` adalah kondisi antara BREAK dan PLANT. BREAK hanya boleh lanjut apabila inventory block target nol dan drop block target pada area kerja sudah diperiksa/ditangani dengan retry terbatas. Drop tersisa yang tidak dapat dijangkau harus menghasilkan PAUSED/RECOVERY, bukan langsung PLANT.

## 18. Perbaikan observability dan finalisasi BREAK (8 November 2026)

- Updater `Status Pabrik` tidak lagi memanggil helper Pabrik lokal yang pada runtime pengguna terbaca `nil`. Nilai inventory dan nama storage untuk tampilan dihitung langsung dari tabel `Pabrik`; perbaikan ini ditujukan khusus error `k .lua:11418` dan belum terverifikasi runtime.
- **BREAK selesai** kini berarti: inventory Item ID block target kosong, data `Drops.list` tersedia, dan tidak ada lagi drop block target pada snapshot Grid BREAK. Jika drop masih ada, Pabrik mencoba mengambil hanya drop Item ID tersebut di area grid dengan maksimal dua percobaan.
- Jika pungutan memulihkan inventory, Pabrik melanjutkan BREAK lebih dulu (tanpa langsung dialihkan oleh ambang refill pada satu langkah tersebut). Jika target drop tetap ada atau data drop tidak tersedia, fase menjadi `PAUSED`; PLANT tidak dipakai sebagai fallback.
- BREAK yang berhenti karena timeout tetapi masih memiliki block tidak dianggap selesai: ia tetap pada BREAK. Semua perilaku ini hanya diperiksa statis; posisi drop, pembaruan `Drops.list`, dan pickup server harus diuji pada lingkungan terotorisasi.

## 19. Stabilisasi startup, branding, dan koneksi game (8 November 2026)

### Istilah operasional

- **UI siap** berarti seluruh kontrol WindUI telah selesai dibuat dan loop status background boleh dimulai. Ini tidak otomatis berarti referensi internal game sudah ditemukan.
- **Koneksi Inventory** berarti `Refs.Inv` berhasil menunjuk object inventory client yang memiliki `slots` dan API baca yang sesuai. Status `Inventori: TIDAK` bukan bukti koneksi internet atau server terputus.
- **Quick scan** mencari `Refs.Inv` dan `Refs.World` melalui upvalue koneksi event yang tersedia. **Deep scan** adalah fallback executor melalui snapshot `getgc(true)` untuk menemukan referensi game tambahan.

### Perubahan statis

- Pembuatan kontrol UI sekarang diberi yield antar-tab dan loop status menunggu `UI.Ready`. Tujuannya agar popup bahasa tidak diikuti pekerjaan background pada GUI yang belum selesai dibuat.
- Re-execution membersihkan GUI grid/fly lama serta label watermark karakter lama. Watermark user-facing memakai nama `Rocky`; nama karakter asli disembunyikan hanya saat label pengganti berhasil dipasang dan dipulihkan saat cleanup.
- Referensi Inventory hasil quick scan tidak lagi dibuang oleh `findRefs()`. Quick scan dan deep scan memakai validator yang sama; ketergantungan wajib pada field `capacity` dihapus karena field tersebut bukan kontrak yang stabil.
- Bila reference inti belum ditemukan, reconnect menjalankan maksimal dua quick retry dengan budget 80 closure per percobaan. Retry tidak melakukan deep scan baru dan melaporkan kegagalan melalui warning.
- Reconnect otomatis hanya menjalankan quick discovery dan tidak mengambil snapshot `getgc(true)`. Satu snapshot GC baru boleh diambil melalui tombol **Hubungkan ke game**; `findRefs()` mengenali reference serta drop dalam traversal yang sama, pemrosesan memberi yield setiap 100 object, lalu snapshot dilepas agar tidak menahan seluruh object GC.

### Batas verifikasi

- Patch ini telah diperiksa dengan `git diff --check` dan penelusuran call-site. Tidak ada compiler Luau lokal yang tersedia, dan tidak ada script yang dijalankan melalui executor.
- Pengambilan snapshot `getgc(true)` sendiri tetap dapat menahan thread sesuai performa executor/perangkat; batching hanya mengurangi beban pemrosesan setelah snapshot tersedia.
- Uji runtime terotorisasi masih diperlukan untuk memastikan: UI tampil setelah memilih bahasa, watermark lama hilang pada re-execution, nama karakter diganti Rocky, `Inventori: OK`, dropdown inventory terisi, tombol reconnect tidak membuat task yatim, dan tidak ada freeze/crash pada perangkat target.

## 20. Istilah startup dan kandidat Support Block (8 November 2026)

- **Koneksi otomatis ringan** berarti startup menunggu `Character` dan `GameGui.World.Camera.PlayerView` secara terbatas, lalu hanya menjalankan quick scan tanpa `getgc(true)`. Reference yang belum ditemukan tetap ditampilkan apa adanya dan tidak memicu full scan tersembunyi.
- **Kandidat Support Block Pabrik** adalah gabungan Item ID pilihan tersimpan, block valid dalam inventory, dan block unik yang terdeteksi pada area kerja world saat ini. Kandidat bukan lagi seluruh isi `ItemDB`.
- Pada mode **Whole World**, kandidat world berasal dari seluruh grid. Pada mode **Lahan**, kandidat world dibatasi koordinat lahan. Block world hanya menjadi kandidat bila tile di atasnya kosong atau berisi tree, sedangkan validasi final penanaman tetap dilakukan saat fase PLANT.
- Pilihan tersimpan tetap dipertahankan walaupun block tersebut sedang tidak ditemukan di world atau inventory. Tidak ditemukannya kandidat tidak memberi izin untuk mengganti atau menghancurkan support block secara otomatis.
- **Watermark holder** adalah container GUI untuk nama karakter/panel yang di-cache. Pencarian descendant dilakukan berkala, sedangkan animasi per-frame hanya memakai holder cache.
- Seluruh perilaku pada bagian ini baru diverifikasi secara statis. Crash executor saat `getgc(true)` mengambil snapshot, bentuk aktual `Refs.World.fg`, refresh dropdown WindUI, dan discovery tree on-demand masih memerlukan pengujian runtime terotorisasi.

## 21. Status reference game (8 November 2026)

- **Primary reference** adalah closure atau table internal game yang ditemukan dari module resmi yang sudah dimuat client, upvalue connection, atau snapshot GC dan lolos kontrak sesuai consumer script.
- **`OK`** berarti reference tersedia dan lolos pemeriksaan operasional saat status dibaca. Khusus World, kontraknya memakai fingerprint `fg/bg/w/h/spawn` dan nama world aktif. Khusus posisi, `state()` harus memberi `x/y` numerik serta tidak `active == false`. `facing`, `vx`, `vy`, dan `grounded` dicatat sebagai capability gerak dari state yang sama, bukan syarat discovery koordinat. Panel detail mencantumkan sumber dan tipe capability tersebut.
- **`FALLBACK`** berarti primary `SetTile` atau `Punch` belum ditemukan, tetapi jalur game yang sudah dikenal source (`Shared.Net.tile`, `Shared.Net.act`, atau legacy `WorldRemotes.SetTile`) tersedia dan tervalidasi bentuk fungsinya. Status ini tidak disamakan dengan primary `OK`.
- **`VIEW`** berarti primary `state()` belum ditemukan tetapi koordinat visual `PlayerView` dapat dibaca. Fallback ini cukup untuk sebagian fitur berbasis tile, tetapi tidak menyediakan seluruh state gerak.
- **`TUNGGU`** berarti object sudah ditemukan tetapi belum operasional, misalnya nama World belum aktif atau State masih inactive. **`TUNDA`** berarti deep discovery belum diperlukan; status ini berlaku untuk Trees, Drop, dan Players sesuai fitur pemakainya. `TIDAK` baru dipakai setelah discovery yang relevan benar-benar dicoba dan gagal.
- **Stale reference** adalah reference yang pernah valid tetapi tidak lagi cocok setelah respawn atau perpindahan world. Reconnect kini membuang World, State, Players, Trees, dan Drop sesi lama sebelum quick scan. Deep scan mempertahankan kandidat aktif dari RunService dan tidak menimpanya dengan kandidat GC pertama. Respawn mereset State lalu menjalankan maksimal dua retry ringan berbudget 80 closure.
- **Feature readiness** adalah pemeriksaan capability yang dipakai bersama oleh panel dan entry point automation. Pabrik memerlukan World aktif, Inventory, State gerak, `WorldMenu.isDown`, jalur place, jalur punch, Drop, dan metadata Trees sebelum mulai. Kegagalan dilaporkan sebagai blocker spesifik; pesan umum “Kamu belum masuk world” tidak lagi dipakai sebagai pengganti kegagalan State.
- **Full discovery manual** berarti `getgc(true)` hanya dipanggil setelah pengguna menekan **Koneksi > Hubungkan ke game**. Preflight automation dan Penjaga tidak menjalankan full scan tersembunyi; jika capability wajib belum ditemukan, keduanya meminta pengguna menjalankan tombol tersebut. Satu traversal snapshot mencari reference inti, Drop, dan Players; Trees dicari setelah World tersedia. Retry delayed-loading dan respawn tetap memakai jalur ringan tanpa snapshot GC baru.
- **Recovery scan** hanya memeriksa struktur candidate reference; discovery Players tidak memanggil `list()` milik game. Laporan crash sebelumnya berkorelasi dengan deep scan startup tanpa syarat dan scan closure berulang. Pola tersebut sudah dihapus, tetapi hubungan sebab-akibat runtime tetap perlu dikonfirmasi di executor terotorisasi.
- Deep drop discovery mengenali field yang dipakai collector (`uid`, `frame`, `x`, `y`, dan `id`) dan tidak lagi mewajibkan metadata legacy `born`.
- Perubahan ini diverifikasi secara statis dan melalui penelusuran dependency/call-site. Nilai status aktual, signature closure versi game sekarang, respons remote, serta ketahanan setelah respawn/world transition masih wajib diuji pada runtime terotorisasi.

## 22. Alur koneksi hybrid manual (8 November 2026)

- **Mode `RINGAN OTOMATIS`** berjalan saat startup dan respawn. Startup menunggu karakter dan `PlayerView`; bila `Tiles` belum siap, fallback visual diberi kesempatan satu detik tanpa menahan primary state discovery selama seluruh timeout. Setelah itu script mencari Inventory, World, State, dan fallback aksi melalui module/upvalue yang sudah tersedia. Jalur ini tidak memanggil `getgc(true)` dan tetap memiliki maksimal dua retry ringan bila reference inti belum operasional.
- **Mode `FULL MANUAL`** dimulai hanya dari tombol **Hubungkan ke game**. Tombol tersebut mereset reference sesi lama, menjalankan quick discovery, mengambil paling banyak satu snapshot `getgc(true)`, lalu mencoba reference inti, Drop, Players, Trees, dan Animasi. Mode terakhir ditampilkan pada panel detail Koneksi.
- Entry point automation tetap melakukan feature readiness. Bila reference wajib belum tersedia dan full scan belum dijalankan, pesan blocker mengarahkan pengguna ke tombol **Hubungkan ke game**; automation tidak mencoba deep scan sendiri.
- Tombol manual tidak menjamin seluruh status menjadi `OK`. Status hanya menjadi `OK` bila object aktual ditemukan dan lolos validator consumer; `FALLBACK`, `TUNGGU`, atau `TIDAK` tetap dipertahankan agar panel tidak memberi false positive.
- Alasan desain: laporan pengguna menunjukkan freeze/crash berkorelasi dengan discovery GC selama startup atau saat fitur mulai. Memindahkan snapshot GC ke tindakan eksplisit menjaga startup ringan, tetap menyediakan discovery lengkap saat diperlukan, dan membuat waktu beban berat dapat diprediksi pengguna.
- Verifikasi yang telah dilakukan hanya statis: `deepRefreshRefs()` memiliki satu call-site dari reconnect manual, preflight/Penjaga tidak lagi memanggilnya, dan startup memanggil `reconnect(..., false)`. Runtime terotorisasi masih wajib menguji waktu full scan, hasil `Posisi`, perpindahan world, respawn, serta apakah struktur game terbaru masih cocok dengan validator.

## 23. Pemulihan reference posisi dan watermark karakter (8 November 2026)

- Log runtime pengguna setelah full scan masih melaporkan `state() aktif dengan x/y/facing/vx tidak ditemukan`. Ini membuktikan penghapusan syarat `grounded` saja belum cukup: kontrak lama `facing/vx` masih membuang kandidat posisi atau tidak dapat membedakan tidak adanya holder state.
- Status `Posisi: OK` tetap tidak di-hardcode. `Refs.State` hanya disimpan bila `state()` berhasil, memberi `x/y` numerik, dan tidak inactive. `facing/vx/vy/grounded` tidak lagi menjadi syarat koordinat; tipe field tersebut dilaporkan sebagai capability agar kekurangan state gerak terlihat saat pengujian.
- Discovery mencatat jumlah table dengan fungsi `state()` selama quick/full traversal tanpa mengambil snapshot tambahan. Jika tidak ada reference yang diterima, alasan kini membedakan antara tidak ada holder sama sekali dan holder ditemukan tetapi hasil terakhir invalid.
- Log runtime berikutnya menunjukkan `tidak ada table dengan fungsi state() pada quick/full scan`. Audit source menemukan `Refs.State` hanya dihapus oleh reconnect dan `CharacterAdded`, sehingga tidak ada loop biasa yang terus menimpa hasil discovery. Kegagalan terjadi pada asumsi bentuk holder: scanner hanya memakai `rawget(v, "state")`.
- Discovery State sekarang memeriksa method langsung, metatable, dan table `metatable.__index`, termasuk `getrawmetatable` bila executor menyediakannya. Method diprobe tanpa `self` lalu dengan owner sebagai `self`; hanya hasil `x/y` valid yang disimpan sebagai accessor tanpa argumen untuk menjaga kontrak consumer lama. Fungsi GC yang tidak terkait tidak dipanggil.
- Build runtime `2026-11-08-state-metatable-1` dicetak pada awal eksekusi agar salinan executor/URL queue lama dapat dibedakan dari source terbaru.
- Watermark `Rocky` hanya overlay lokal dan tidak mengubah `LocalPlayer.Name`, `DisplayName`, atau `Refs.State`. Resolver sekarang mengutamakan `NameTag`/`BitmapText` di subtree `PlayerView` karakter lokal, mempertahankan fallback layout Camera lama, serta mencari `BitmapText` secara rekursif untuk hide/restore.
- Holder watermark harus berupa `GuiObject`. Jika holder karakter atau panel belum ditemukan setelah lima detik, warning satu kali menjelaskan bagian yang gagal; resolver tetap mencoba ulang setiap detik dan melakukan rebind setelah object GUI diganti/respawn.
- Perubahan telah diperiksa secara statis dengan `git diff --check` dan penelusuran seluruh call-site `readStateRef()`. Tidak ada compiler Luau lokal atau pengujian executor. Runtime terotorisasi masih diperlukan untuk memastikan `Posisi: OK`, gerakan A*, nama Rocky tampil, nama asli pulih saat cleanup, dan label tidak menempel pada pemain lain.
