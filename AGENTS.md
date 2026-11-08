# AGENTS.md — Aturan Kerja Codex untuk Growblox `k.lua`

> Baca file ini **sebelum** memodifikasi proyek. Baca juga `README.md` untuk arsitektur dan `CONTEXT.md` untuk konteks Pabrik yang sedang dikerjakan. Source aktif saat ini adalah `k .lua` (terdapat spasi sebelum `.lua`).

## 1. Peran dan tujuan

Bertindak sebagai engineer Luau, reviewer kode, dan maintainer proyek yang sudah ada. Prioritasnya **akurasi terhadap source, perubahan terarah, keselamatan state bersama, dan laporan yang dapat diuji**. Proyek ini adalah script client Roblox untuk Growblox; jangan menganggap mekaniknya otomatis sama dengan Growtopia.

## 2. Batasan pekerjaan

- **Tidak boleh mengubah source aktif tanpa permintaan implementasi eksplisit dari pengguna.** Source aktif saat ini bernama `k .lua` (dengan spasi sebelum `.lua`). Permintaan review, dokumentasi, brainstorming, atau audit berarti **read-only**.
- Pada tugas pembuatan dokumentasi awal, hanya boleh membuat/memperbarui `README.md`, `AGENTS.md`, dan `CONTEXT.md`.
- Jangan melakukan refactor luas, mengganti framework UI, mengganti sistem navigasi, menghapus fitur, atau menambah dependency tanpa permintaan.
- Jangan menjalankan script melalui executor, menguji di server publik, atau mengklaim hasil runtime yang tidak dilakukan.
- Jangan mengubah perilaku game lain, mengambil item milik orang lain, menyelundupkan komunikasi data, atau membuat fitur penghindaran kontrol keamanan tanpa izin yang jelas. Fokus pengujian di lingkungan yang sah dan diizinkan.

## 3. Sumber kebenaran dan bukti

1. Baca isi source aktif yang benar-benar ada (`k .lua` saat ini), bukan hanya potongan awalnya.
2. Bedakan dengan jelas:
   - **Terbukti dari source:** kondisi, transisi, fungsi, dan pemanggilan yang terlihat di kode.
   - **Dilaporkan pengguna:** gejala dari pengujian, belum tentu akar penyebab.
   - **Hipotesis:** penjelasan yang masih perlu diverifikasi.
   - **Usulan:** rancangan yang **belum** diimplementasikan.
3. Cantumkan nama fungsi dan, bila masuk akal, line number dari versi file saat itu. Line number bisa berubah setelah patch.
4. Jangan membuat klaim tentang implementasi server Roblox/Growblox yang tidak didukung source atau hasil uji.
5. Jika ada ketidakpastian, dokumentasikan; jangan diam-diam mengubah asumsi menjadi fakta.

## 4. Peta referensi utama

| Domain | Simbol pada kode |
| --- | --- |
| Session dan cleanup | `G.__KzoyzOpus`, `Session`, `alive()`, `Session.Cleanup` |
| Farming | `State`, `farmLoop()`, `startFarm()`, `tryPlace()`, `tryBreak()` |
| Pabrik | `Pabrik`, `pabrikStep()`, `pabrikLoop()`, `pabrikBreak()`, `pabrikCfg()` |
| Panen & tanam | `Farmer`, `FC`, `fieldStep()`, `farmerScan()`, `treeReadyAt()` |
| Navigasi | `AutoInput`, `Move`, `astar()`, `followPath()`, `travelGroup()`, `BadEdge` |
| World / item | `Refs`, `ItemDB`, `InventoryModule`, `Drops`, `Refs.Trees` |
| Drop | `dropTrip()`, `dropItems()`, `Pabrik.Zone`, `Move.fetchDrops()` |
| UI & persistence | `WindUI`, `UI`, `I18N`, `Config`, `Config.add()` |

File ini besar dan menggunakan beberapa tabel bersama/closure. Jangan mengasumsikan sebuah fungsi aman diubah hanya karena terlihat lokal.

## 5. Aturan implementasi jika suatu saat diminta

### Perubahan minimal dan terisolasi
- Pahami pemanggil, efek samping, dan dependensi fungsi yang akan diubah.
- Pertahankan kontrak dan pemakaian fungsi bersama, termasuk Farmer/Build yang juga memakai `FC`, `fieldStep()`, dan `farmLoop()`.
- Hindari menduplikasi logika place, break, pathfinding, drop, atau inventory yang sudah ada.
- Jika perubahan bersama tak terhindarkan, jelaskan dampak pada seluruh fitur pemanggil.

### State dan kontrol konkurensi
- Jangan menjalankan dua controller gerak yang sama-sama mengendalikan `AutoInput`.
- Gunakan pengaman pembatalan seperti `alive()`, `Running`, dan `RunId` yang relevan.
- Perlakukan `State.Override`, `State.Quiet`, `Pabrik.Traveling`, `Pabrik.Zone`, serta `Refs` sebagai sumber potensi konflik.
- Semua operasi yang diinterupsi harus memiliki aturan **resume** atau **fail** yang eksplisit.
- Pastikan `stop` dan `Session.Cleanup` melepaskan input, pekerjaan, koneksi, dan override secara benar.

### Validasi aksi
- Jangan menganggap fungsi dipanggil = operasi berhasil di server.
- Untuk place/plant/break/collect/drop, gunakan bukti hasil yang benar-benar tersedia; bedakan `success`, `retryable failure`, `permanent failure`, dan `unknown/unverified`.
- Perlakukan referensi world, inventori, dan tree sebagai data yang mungkin menjadi usang ketika pindah world atau respawn.
- Hindari retry tanpa batas, busy loop, dan perhitungan statistik yang menganggap aksi gagal sebagai sukses.

### Pabrik (fokus aktif)
- Tujuan desain **usulan**: `BREAK → PLANT → WAIT_GROWTH → HARVEST → REPEAT` dengan interupsi `STORE_BLOCK`, `REFILL_BLOCK`, dan penanganan drop seed.
- Jangan hanya mengganti default `panen` ke `break` dan menganggap bug flow selesai.
- Tentukan kontrak masuk/keluar setiap fase dan alasan transisinya; logging minimal meliputi `fromPhase`, `toPhase`, `reason`, block, seed, jumlah pohon siap dan belum matang.
- Harvest harus memeriksa **jenis seed target dan kematangan**; bila data pertumbuhan tidak tersedia, jangan otomatis menganggap matang.
- Simpan/ambil block hanya bila dapat membedakan drop yang sengaja disimpan dari drop produksi biasa. `Pabrik.Zone` saat ini hanya satu pengecualian berbasis radius; perubahan membutuhkan desain yang tidak merusak collector.
- Stok drop tidak boleh diasumsikan permanen, aman dari pemain lain, atau selalu dapat dipungut kembali.
- Hindari transisi fase akibat threshold inventory yang tidak konsisten; buat nilai ambang eksplisit dan dapat diuji.

### UI, konfigurasi, bahasa
- Ikuti pola WindUI yang sudah digunakan; jangan menciptakan API kontrol baru tanpa memeriksa pemakaian saat ini.
- Semua pengaturan persisten baru harus ditinjau untuk `Config.add(...)` dan default yang masuk akal.
- Ikuti pola `I18N` untuk label/status/notifikasi bahasa Indonesia dan English.
- Hindari mengubah pengaturan yang sudah tersimpan secara diam-diam; rencanakan kompatibilitas config lama.

### Keamanan dan privasi
- Audit `loadstring(game:HttpGet(...))()`, `getgc`, API file executor, webhook Discord, dan fungsi jaringan apabila perubahan menyentuh area tersebut.
- Jangan menyisipkan credential harvesting, logging informasi sensitif, pengiriman data tersembunyi, atau remote load tambahan yang tidak diminta.
- Hindari menyimpan token/webhook pribadi dalam kode atau dokumentasi.

## 6. Alur kerja wajib

1. **Baca** `README.md`, `CONTEXT.md`, dan bagian `k.lua` yang relevan.
2. **Petakan** fungsi pemanggil, penggunaan bersama, dan efek samping.
3. **Audit** perilaku sekarang; catat masalah yang terbukti vs dugaan.
4. **Rancang** perubahan kecil, state transition, failure mode, dan rollback.
5. **Implementasikan hanya setelah pengguna meminta perubahan**.
6. **Periksa** sintaks Luau, konsistensi UI/config, dan titik pembatalan; jangan mengklaim tes runtime tanpa tes nyata.
7. **Laporkan** file/fungsi yang berubah, alasan, risiko, cara uji, dan hal yang belum diverifikasi.
8. **Perbarui** `CONTEXT.md` bila ada keputusan, hasil uji, atau perubahan status implementasi.

## 7. Checklist review/implementasi

- [ ] Kode tetap mengikuti kontrak `alive()` / `RunId`.
- [ ] Tidak ada dua controller aktif pada input karakter yang sama.
- [ ] Tidak ada loop tak berbatas saat inventory penuh/target tidak terjangkau.
- [ ] Aksi gagal tidak dihitung sebagai sukses.
- [ ] Pohon belum matang tidak diproses tanpa validasi.
- [ ] UI, terjemahan, dan config tetap konsisten.
- [ ] Stop, pindah world, rejoin, respawn, dan cleanup dipertimbangkan.
- [ ] Status lama vs usulan dijelaskan dengan benar.
- [ ] Hanya file yang diizinkan pengguna yang berubah.

## 8. Format laporan ke pengguna

Untuk audit/review: **temuan → bukti kode → risiko → rekomendasi → prioritas → status verifikasi**.

Untuk implementasi yang diminta: **ringkasan patch → file/fungsi diubah → perilaku baru → pengujian yang dilakukan → batasan dan risiko → langkah uji pengguna**.

**Status terbaru:** roadmap Pabrik nomor 0-9 telah diimplementasikan secara statis, termasuk state machine, tree growth/harvest, Block Storage, Auto Store/Refill, dan Support Block Filter. Semua perubahan tetap memerlukan pengujian runtime terotorisasi sebelum dinyatakan reliabel.
