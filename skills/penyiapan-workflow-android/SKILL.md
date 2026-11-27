---
name: penyiapan-workflow-android
description: Gunakan saat pengguna meminta membuat, memperbaiki, atau memeriksa GitHub Actions maupun EAS Build untuk APK Android Expo/React Native, khususnya Development Build, target ABI arm64-v8a, konfigurasi SDK, cache, dan artefak APK.
---

# Penyiapan Workflow Android

## Tujuan

Membuat pipeline Android yang menghasilkan APK valid dan dapat dipasang pada perangkat, dengan konfigurasi yang sesuai dengan versi Expo, React Native, Android Gradle Plugin (AGP), Gradle, Java, dan Android SDK pada proyek.

Skill ini mencakup dua jalur yang berbeda:
- **GitHub Actions + Gradle:** runner GitHub membangun APK dari proyek Android hasil Expo Prebuild.
- **EAS Build:** layanan Expo membangun APK menggunakan profil pada `eas.json`.

Jangan mencampur konfigurasi atau menganggap kedua jalur tersebut menjalankan perintah yang sama.

## Pemicu penggunaan

Gunakan skill ini ketika pengguna meminta:
- Membuat atau memperbaiki workflow build Android di GitHub Actions.
- Menghasilkan APK Expo Development Build, bukan sekadar menjalankan Expo Go.
- Membatasi APK ke ABI tertentu, terutama `arm64-v8a`.
- Menyamakan workflow dengan repositori referensi yang terbukti berhasil.
- Memeriksa kegagalan SDK, Gradle, Java, prebuild, cache, atau unggah artefak APK.

## Prinsip wajib

1. Baca aturan repo, memori proyek, dan instruksi skill terkait sebelum mengubah file.
2. Periksa struktur repo dan branch aktif. Jika repo tidak memiliki branch `master`, gunakan `main`; jangan mengasumsikan nama branch.
3. Periksa workflow referensi yang diminta pengguna secara langsung. Jangan menganggap konfigurasi repositori referensi sama dengan konfigurasi proyek target.
4. Jangan mengubah file atau membuat commit jika pengguna hanya meminta analisis. Jika pengguna meminta perbaikan, batasi perubahan pada file yang diperlukan.
5. Jangan mengarang versi Java, Node.js, Bun, Android SDK, NDK, Build Tools, Gradle, atau AGP. Ambil dari konfigurasi proyek, dokumentasi resmi yang sesuai versinya, atau kebutuhan build yang terdeteksi.
6. Jangan menetapkan API Android terbaru secara paksa. Versi `compileSdk` harus mengikuti Expo SDK dan konfigurasi native proyek.
7. Jangan membuat symlink SDK seperti `android-37` ke `android-37.0` sebagai solusi umum. Gunakan nama paket SDK yang benar-benar tersedia. Jika paket tidak tersedia, tentukan versi yang didukung proyek dan runner sebelum mengubah konfigurasi.
8. Jangan menganggap `eas.json` mengendalikan perintah Gradle di GitHub Actions. Konfigurasi EAS hanya berlaku ketika jalur build memang menggunakan EAS.
9. Jangan menyatakan APK berhasil hanya karena langkah Gradle selesai. Pastikan berkas APK benar-benar ada, tidak kosong, dan sesuai ABI serta jenis build yang diminta.
10. Jangan menjalankan build lokal jika aturan proyek melarangnya. Gunakan GitHub Actions atau layanan build yang memang ditetapkan proyek.

## Alur pemeriksaan

### 1. Identifikasi proyek dan jalur build

Periksa setidaknya:
- `package.json` dan lockfile (`bun.lock`, `bun.lockb`, `package-lock.json`, atau `yarn.lock`).
- `app.json` atau `app.config.js/ts`.
- `eas.json`, jika tersedia.
- `.github/workflows/*.yml`.
- Direktori `android/`, termasuk `build.gradle`, `settings.gradle`, `gradle.properties`, dan Gradle Wrapper, jika sudah ada.

Tentukan apakah proyek memakai Continuous Native Generation (CNG) atau menyimpan proyek native Android secara langsung. Jangan menjalankan Prebuild dengan `--clean` sebelum memastikan direktori native boleh dibuat ulang dan tidak menyimpan perubahan manual.

### 2. Tentukan jenis APK

Bedakan dengan tegas:
- **Expo Go:** aplikasi Expo Go yang menjalankan JavaScript proyek; bukan APK aplikasi mandiri milik proyek.
- **Development Build:** APK aplikasi proyek yang menyertakan `expo-dev-client`, sehingga dapat terhubung ke Metro dan mendukung native module proyek.
- **Preview/internal APK:** APK mandiri untuk pengujian atau distribusi internal; tidak otomatis merupakan Development Build.
- **Production:** build untuk distribusi produksi. Jangan menyamakannya dengan debug APK.

Untuk Development Build:
- Pastikan `expo-dev-client` menjadi dependency proyek dan kompatibel dengan Expo SDK yang digunakan.
- Untuk EAS, periksa profil yang memiliki `developmentClient: true` dan distribusi internal.
- Untuk GitHub Actions + Gradle, pastikan dependency tersebut terpasang sebelum Prebuild dan build.
- Jangan menganggap nama profil `development` saja cukup untuk menjadikan APK sebagai Development Build.

### 3. Tentukan ABI

Jika pengguna meminta arm64 saja:
- Gunakan `arm64-v8a` secara konsisten pada perintah Gradle atau konfigurasi native yang benar-benar mengendalikan ABI.
- Periksa apakah konfigurasi React Native dan `abiFilters` Gradle saling bertentangan.
- Jangan menyimpulkan APK arm64-only hanya dari nama artefak atau nama profil.
- Verifikasi isi APK dengan memeriksa direktori native library, misalnya `lib/arm64-v8a/`, dan pastikan ABI lain tidak ikut terpaket jika permintaannya arm64-only.
- Jangan menambahkan ABI lain tanpa alasan kompatibilitas yang jelas dan persetujuan pengguna.

Contoh parameter Gradle untuk React Native (sesuaikan dengan versi plugin dan konfigurasi proyek):

```bash
./gradlew :app:assembleDebug -PreactNativeArchitectures=arm64-v8a
```

Parameter tersebut bukan pengganti pemeriksaan hasil APK. Jangan menambahkan properti Gradle lain yang belum diverifikasi dukungannya.

### 4. Siapkan GitHub Actions + Gradle

Urutan umum workflow:
1. Jalankan hanya pada trigger yang diminta pengguna. Untuk workflow build manual, gunakan `workflow_dispatch`; jangan menambah trigger `push` atau `pull_request` tanpa kebutuhan.
2. Checkout branch atau commit yang benar. Gunakan `submodules: recursive` hanya jika proyek memang memakai submodule.
3. Siapkan runtime JavaScript sesuai lockfile dan versi proyek.
4. Aktifkan cache package manager berdasarkan lockfile dan cache Gradle berdasarkan file konfigurasi Gradle. Cache harus memiliki key yang berubah ketika dependency atau konfigurasi build berubah.
5. Siapkan Java sesuai kompatibilitas AGP/Gradle. Gunakan versi action yang masih didukung; jangan menyalin versi action lama dari workflow referensi tanpa pemeriksaan.
6. Siapkan Android SDK. Instal hanya platform, Build Tools, atau NDK yang diperlukan proyek dan tersedia pada runner.
7. Instal dependency secara reproducible menggunakan lockfile, misalnya `bun install --frozen-lockfile` atau `npm ci`. Pilih perintah sesuai package manager proyek.
8. Jalankan Expo Prebuild hanya jika dibutuhkan oleh CNG. Hindari `--clean` jika ada native customization yang harus dipertahankan.
9. Jalankan Gradle Wrapper dari direktori Android atau gunakan path task Gradle yang eksplisit.
10. Temukan APK dan validasi sebelum mengunggahnya.
11. Unggah artefak hanya jika build berhasil dan APK tersedia. Gunakan `if-no-files-found: error` agar kegagalan path tidak tersembunyi.

Jangan menyalin workflow referensi secara mentah. Pertahankan pola yang terbukti berguna, tetapi sesuaikan package manager, Java, SDK, ABI, task Gradle, dan path artefak dengan proyek target.

### 5. Siapkan EAS Build

Jika pengguna memilih EAS:
- Pastikan `eas.json` valid dan profil yang dipanggil memang tersedia.
- Untuk Development Build Android, periksa `developmentClient: true` dan `distribution: internal`.
- Pastikan hasil Android berupa APK jika harus dipasang langsung pada perangkat. Periksa konfigurasi `android.buildType` atau perilaku profil development yang digunakan.
- Jika ABI dibatasi, pastikan konfigurasi atau perintah Gradle yang digunakan EAS benar-benar menerapkan pembatasan tersebut.
- Periksa autentikasi CI melalui secret `EXPO_TOKEN`. Jangan menulis token ke file, log, atau argumen yang dapat terekspos.
- Jika GitHub Actions perlu menyimpan APK sebagai artefak, pastikan workflow mengunduh hasil build EAS dan mengunggah file APK yang benar. Memulai build EAS saja tidak otomatis menghasilkan artefak lokal di runner.

Rujukan konfigurasi resmi: https://docs.expo.dev/build/

### 6. Android SDK dan kegagalan versi

Saat terjadi kegagalan SDK:
1. Ambil nilai `compileSdk`, `targetSdk`, Build Tools, NDK, AGP, dan Gradle dari konfigurasi proyek yang benar-benar digunakan.
2. Periksa daftar paket SDK yang tersedia pada runner atau dokumentasi image runner.
3. Bedakan nama API level dengan nama paket SDK. Jangan menebak suffix versi paket.
4. Instal paket yang diperlukan dengan nama persis yang tersedia.
5. Jika versi yang diwajibkan belum tersedia, cari jalur kompatibel yang didukung Expo SDK/AGP. Jangan menurunkan atau menaikkan SDK secara sepihak.
6. Jelaskan bukti, dampak, dan pilihan perbaikan sebelum mengubah versi utama.

### 7. Validasi hasil APK

Sebelum menyatakan workflow selesai, pastikan:
- Task Gradle atau proses EAS berakhir sukses.
- File APK ditemukan pada lokasi hasil build yang benar.
- Ukuran file masuk akal dan lebih besar dari nol.
- APK dapat dibaca sebagai ZIP.
- Application ID sesuai konfigurasi aplikasi.
- Jenis build sesuai permintaan (Development Build atau APK internal biasa).
- ABI di dalam APK sesuai target.
- Path unggah artefak cocok dengan lokasi file yang benar-benar dihasilkan.
- Langkah unggah artefak gagal secara jelas jika APK tidak ditemukan.

Jika tersedia pada runner, gunakan alat Android seperti `apkanalyzer` atau `aapt` untuk memeriksa metadata APK. Jangan menganggap pemeriksaan nama file atau ukuran saja cukup untuk membuktikan APK valid.

## Perbandingan dengan repositori referensi

Ketika pengguna menyebut repositori referensi:
1. Baca workflow, `package.json`, `app.json`, `eas.json`, serta konfigurasi native yang relevan dari kedua repo.
2. Catat perbedaan yang memengaruhi hasil build: runtime, dependency, Expo SDK, Java, Gradle/AGP, SDK, ABI, jenis build, signing, dan artefak.
3. Pisahkan konfigurasi yang memang harus sama dari konfigurasi yang harus mengikuti proyek masing-masing.
4. Terapkan hanya perubahan yang didukung bukti. Jangan menyebut kedua workflow sudah sama jika masih ada perbedaan yang disengaja atau belum diverifikasi.

## Format laporan

Sampaikan secara ringkas:
1. **Temuan:** penyebab atau ketidaksesuaian beserta file dan bagian terkait.
2. **Perubahan:** file yang diubah dan alasan teknisnya.
3. **Validasi:** pemeriksaan yang benar-benar dilakukan dan hasilnya.
4. **Belum terverifikasi:** hal yang hanya dapat dipastikan setelah workflow dijalankan atau APK diuji pada perangkat.

Jangan mengklaim build, instalasi, atau pengujian perangkat berhasil jika belum benar-benar dilakukan.

## Checklist

- [ ] Jalur build (Gradle atau EAS) sudah ditentukan.
- [ ] Jenis APK (Development Build, internal, atau production) sudah jelas.
- [ ] `expo-dev-client` diverifikasi jika Development Build diminta.
- [ ] Versi Java, Node/Bun, Gradle, AGP, dan Android SDK mengikuti proyek.
- [ ] Tidak ada API SDK atau symlink yang dipaksakan tanpa bukti.
- [ ] ABI dikonfigurasi dan diverifikasi sesuai permintaan.
- [ ] Cache memakai key yang relevan dengan lockfile dan konfigurasi.
- [ ] Prebuild tidak menghapus native customization.
- [ ] Lokasi APK ditemukan dan divalidasi sebelum unggah.
- [ ] Artefak gagal dengan jelas jika APK tidak tersedia.
- [ ] Laporan membedakan hasil yang sudah diuji dan yang belum.
