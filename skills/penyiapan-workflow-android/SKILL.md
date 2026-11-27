---
name: penyiapan-workflow-android
description: Gunakan saat pengguna meminta membuat, memperbaiki, membandingkan, atau menganalisis GitHub Actions dan proses build APK Android untuk proyek Flutter maupun React Native dengan Expo, termasuk Development Build, ABI, SDK, cache, dan artefak.
---

# Penyiapan Workflow Android

## Tujuan dan cakupan

Menyiapkan pipeline Android yang menghasilkan APK valid untuk dua framework:
- **Flutter:** aplikasi yang dibangun dengan Flutter dan Dart.
- **React Native dengan Expo:** aplikasi yang dibangun dengan JavaScript/TypeScript, Expo, dan React Native.

Skill ini mengatur GitHub Actions, konfigurasi build Android, cache, ABI, dan artefak. Prosedur Flutter dan Expo wajib dipisahkan. Jangan menerapkan perintah, konfigurasi, atau asumsi satu framework ke framework lainnya.

Jalur build yang mungkin digunakan:
- GitHub Actions menjalankan toolchain framework dan Gradle.
- EAS Build untuk proyek Expo jika memang dipilih.
- Layanan build lain hanya jika diminta atau sudah ditetapkan proyek.

## Pemicu penggunaan

Gunakan saat pengguna meminta:
- Membuat, memperbaiki, atau memeriksa workflow APK Android.
- Menyiapkan APK development, debug, preview/internal, atau production.
- Membatasi arsitektur APK, terutama `arm64-v8a`.
- Menyamakan workflow dengan repositori referensi.
- Memperbaiki kegagalan SDK, Java, Gradle, Flutter, Expo Prebuild, cache, atau upload artefak.

## Aturan wajib

1. Baca aturan repo, memori proyek, dan skill terkait sebelum bekerja.
2. Identifikasi framework berdasarkan file proyek, bukan dugaan. Flutter umumnya memiliki `pubspec.yaml`; Expo memiliki `package.json` dan konfigurasi Expo.
3. Periksa branch yang tersedia. Jika repo tidak memiliki `master`, gunakan `main`. Jangan mengasumsikan nama branch.
4. Jika pengguna menyebut repo referensi, baca workflow dan konfigurasi relevan dari repo referensi serta repo target. Jangan menyalin mentah konfigurasi yang khusus untuk framework atau versi tertentu.
5. Jangan mengarang versi Flutter, Dart, Node.js, Bun, Expo SDK, Java, Android SDK, NDK, Build Tools, Gradle, atau Android Gradle Plugin (AGP). Ambil dari konfigurasi proyek dan dokumentasi resmi yang sesuai.
6. Jangan memaksakan Android API terbaru atau membuat symlink SDK sebagai solusi umum. Gunakan paket SDK yang benar-benar tersedia dan kompatibel dengan toolchain proyek.
7. Jangan menganggap konfigurasi EAS mengendalikan build GitHub Actions yang menjalankan Gradle secara langsung.
8. Jangan mengklaim APK berhasil hanya karena task build sukses. Periksa berkas APK, metadata, ABI, dan hasil upload.
9. Patuhi larangan build lokal yang ditetapkan aturan proyek. Jika build hanya boleh melalui GitHub Actions, jangan menjalankan build pada mesin lokal.
10. Jangan mengubah file atau membuat commit saat pengguna hanya meminta analisis. Lakukan perubahan hanya sesuai permintaan pengguna.

## 1. Identifikasi framework dan konfigurasi

### Flutter

Periksa:
- `pubspec.yaml` dan `pubspec.lock`.
- `.metadata`, `analysis_options.yaml`, dan konfigurasi Flutter lain yang relevan.
- `android/settings.gradle(.kts)`, `android/build.gradle(.kts)`, `android/app/build.gradle(.kts)`.
- `android/gradle/wrapper/gradle-wrapper.properties`.
- `gradle.properties`, manifest, signing config, flavor, dan file konfigurasi Android lain yang dipakai.
- Workflow pada `.github/workflows/`.

Tentukan versi Flutter/Dart dari konfigurasi proyek, FVM jika digunakan, atau kebijakan repo. Jangan mengasumsikan versi stable terbaru otomatis cocok.

### React Native dengan Expo

Periksa:
- `package.json` dan lockfile package manager.
- `app.json` atau `app.config.js/ts`.
- `eas.json`, jika tersedia.
- `android/` beserta konfigurasi Gradle, jika tersedia.
- `expo-dev-client` jika pengguna meminta Development Build.
- Workflow pada `.github/workflows/`.

Tentukan apakah proyek memakai Continuous Native Generation (CNG) atau mengelola direktori native secara langsung. Jangan menjalankan Prebuild dengan `--clean` sebelum memastikan perubahan native khusus tidak akan hilang.

## 2. Tentukan jenis build

Jangan menyamakan jenis build berikut:
- **Debug/development:** build untuk pengembangan dan debugging sesuai kemampuan framework.
- **Expo Development Build:** aplikasi Expo mandiri yang menyertakan `expo-dev-client`; bukan Expo Go.
- **Preview/internal:** APK untuk pengujian atau distribusi internal; belum tentu Development Build.
- **Release/production:** build untuk distribusi pengguna.

Untuk Flutter, tentukan mode `debug`, `profile`, atau `release` berdasarkan tujuan. Jangan menyebut APK release sebagai development build.

Untuk Expo Development Build:
- Pastikan `expo-dev-client` menjadi dependency dan kompatibel dengan Expo SDK.
- Untuk EAS, periksa profil yang menggunakan `developmentClient: true` dan distribusi internal.
- Untuk GitHub Actions + Gradle, pastikan dependency terpasang sebelum Prebuild dan build.
- Nama profil `development` saja tidak membuktikan bahwa hasilnya Development Build.

## 3. Prosedur build Flutter

1. Siapkan Flutter SDK pada versi yang ditentukan proyek.
2. Siapkan Java dan Android SDK yang kompatibel dengan AGP/Gradle proyek.
3. Pulihkan dependency menggunakan lockfile. Gunakan `flutter pub get --enforce-lockfile` jika didukung versi Flutter proyek; jika tidak, gunakan perintah yang kompatibel dan jelaskan alasannya.
4. Periksa flavor, entry point, dart-define, signing, dan konfigurasi environment yang diperlukan.
5. Bangun APK sesuai mode dan target ABI yang diminta.
6. Temukan dan validasi hasil APK sebelum upload.

Contoh perintah arm64-only untuk Flutter, sesuaikan dengan flavor dan entry point proyek:

```bash
flutter build apk --release --target-platform android-arm64
```

Untuk APK terpisah per ABI, Flutter menyediakan opsi `--split-per-abi`. Jangan gunakan opsi tersebut jika pengguna meminta satu APK universal atau jika alur distribusi proyek mengharuskan satu berkas tertentu. Periksa nama dan lokasi output aktual, jangan menebaknya.

Jika proyek memakai flavor, gunakan sintaks flavor yang sesuai konfigurasi proyek, misalnya `--flavor` dan `-t` bila diperlukan. Jangan menambahkan flavor atau dart-define yang tidak ditemukan di repo atau tidak diberikan pengguna.

## 4. Prosedur build React Native dengan Expo

### GitHub Actions + Gradle

1. Checkout branch/commit yang benar. Checkout submodule secara rekursif hanya jika proyek memang menggunakannya.
2. Siapkan Node.js atau Bun sesuai lockfile dan konfigurasi proyek.
3. Pulihkan dependency secara reproducible, misalnya `npm ci` atau perintah package manager yang sesuai.
4. Siapkan Java dan Android SDK sesuai kompatibilitas Expo SDK, AGP, dan Gradle.
5. Jalankan Expo Prebuild hanya jika proyek menggunakan CNG dan direktori native perlu dihasilkan.
6. Jalankan Gradle Wrapper dengan task build yang sesuai jenis APK.
7. Validasi APK dan upload artefak.

Hindari `expo prebuild --clean` jika direktori native memiliki perubahan manual yang harus dipertahankan.

### EAS Build

Jika pengguna memilih EAS:
- Pastikan profil yang dipanggil tersedia dan konfigurasi `eas.json` valid.
- Untuk Development Build, periksa `developmentClient: true` dan `distribution: internal`.
- Pastikan output Android berupa APK jika harus dipasang langsung. Periksa `android.buildType` dan konfigurasi profil.
- Jika membatasi ABI, pastikan konfigurasi Gradle atau opsi build yang digunakan EAS benar-benar menerapkannya.
- Gunakan secret `EXPO_TOKEN` untuk autentikasi CI bila diperlukan. Jangan mengekspos token pada log atau file.
- Jika APK harus menjadi artefak GitHub Actions, unduh hasil EAS ke runner lalu unggah file yang benar. Memulai EAS Build tidak otomatis membuat APK di workspace runner.

Dokumentasi resmi: https://docs.expo.dev/build/

## 5. Konfigurasi GitHub Actions yang berlaku umum

1. Gunakan trigger sesuai permintaan. Untuk build manual gunakan `workflow_dispatch`; jangan menambah trigger otomatis tanpa kebutuhan.
2. Gunakan runner yang sesuai dengan kebutuhan toolchain.
3. Checkout branch/commit yang benar dan submodule hanya jika diperlukan.
4. Siapkan toolchain framework yang tepat: Flutter/Dart untuk Flutter; Node/Bun dan Expo untuk Expo.
5. Siapkan Java dan Android SDK sesuai konfigurasi native proyek.
6. Aktifkan cache dependency framework dan Gradle. Cache key harus memperhitungkan lockfile serta file konfigurasi build yang relevan.
7. Gunakan action versi yang masih didukung. Jangan menyalin versi action lama dari workflow referensi tanpa pemeriksaan.
8. Pulihkan dependency menggunakan lockfile.
9. Jalankan perintah build yang benar untuk framework dan jenis APK.
10. Temukan APK dengan pemeriksaan yang gagal jika file tidak ditemukan.
11. Unggah hanya hasil yang valid. Gunakan `if-no-files-found: error`; jangan memakai `if: always()` untuk menyamarkan kegagalan build sebagai keberhasilan upload.

Pilih cache secara spesifik:
- Flutter: cache pub dan cache Gradle.
- Expo: cache package manager dan cache Gradle.
Jangan mengasumsikan cache Android SDK/NDK diperlukan; tambahkan hanya jika manfaat dan key invalidasinya jelas.

## 6. ABI dan kompatibilitas perangkat

Jika pengguna meminta arm64 saja:
- Target ABI harus `arm64-v8a`.
- Flutter: gunakan target platform `android-arm64`; verifikasi output APK.
- React Native/Expo: gunakan konfigurasi arsitektur React Native atau `abiFilters` yang benar-benar berlaku pada versi proyek.
- Jangan menyimpulkan ABI dari nama file artefak.
- Periksa isi APK, terutama direktori native library seperti `lib/arm64-v8a/`, dan pastikan ABI lain tidak ikut terpaket bila targetnya arm64-only.
- Jangan menambah ABI lain tanpa alasan kompatibilitas dan persetujuan pengguna.

Contoh parameter Gradle React Native (bukan perintah universal; verifikasi dukungan versi proyek):

```bash
./gradlew :app:assembleDebug -PreactNativeArchitectures=arm64-v8a
```

Jangan menerapkan parameter React Native tersebut pada proyek Flutter.

## 7. Android SDK dan kegagalan versi

Saat terjadi kegagalan SDK:
1. Ambil `compileSdk`, `targetSdk`, Build Tools, NDK, AGP, dan Gradle dari konfigurasi native yang benar-benar digunakan.
2. Periksa versi Flutter/Expo dan persyaratan Android yang sesuai.
3. Periksa paket yang tersedia pada runner. Bedakan API level dengan nama paket SDK; jangan menebak suffix versi.
4. Instal paket dengan nama persis yang tersedia.
5. Jika versi yang diperlukan tidak tersedia, cari versi kompatibel berdasarkan dokumentasi resmi framework dan toolchain.
6. Jangan menurunkan atau menaikkan versi utama secara sepihak. Jelaskan bukti, dampak, dan pilihan perbaikannya.

## 8. Perbandingan repositori referensi

Ketika pengguna menyebut repo referensi:
1. Baca workflow, konfigurasi framework, konfigurasi Android, lockfile, dan file build yang relevan pada kedua repo.
2. Pastikan framework kedua repo sama sebelum menyalin langkah khusus framework.
3. Bandingkan runtime, dependency, versi SDK, Java, Gradle/AGP, ABI, jenis build, signing, cache, dan lokasi artefak.
4. Pisahkan pola umum yang dapat disamakan dari konfigurasi yang harus berbeda.
5. Terapkan hanya perubahan yang didukung bukti. Nyatakan dengan jelas jika ada bagian yang belum dapat diverifikasi.

## 9. Validasi APK

Sebelum menyatakan workflow berhasil, periksa:
- Proses build benar-benar berakhir sukses.
- APK ditemukan pada lokasi output aktual dan ukurannya lebih dari nol.
- APK dapat dibaca sebagai arsip ZIP.
- Application ID/package name sesuai konfigurasi aplikasi.
- Mode/jenis build sesuai permintaan.
- ABI di dalam APK sesuai target.
- Signing sesuai kebutuhan distribusi.
- Path upload cocok dengan lokasi output.
- Upload gagal secara jelas jika APK tidak ditemukan.

Gunakan `apkanalyzer`, `aapt`, atau alat Android lain jika tersedia untuk memeriksa metadata. Pemeriksaan ukuran dan nama file saja tidak membuktikan APK valid. Instalasi dan pengujian pada perangkat fisik hanya boleh dinyatakan berhasil jika benar-benar dilakukan.

## 10. Format laporan

Sampaikan:
1. **Temuan:** masalah dan lokasi file.
2. **Perubahan:** file yang diubah dan alasan teknis.
3. **Validasi:** pemeriksaan yang benar-benar dijalankan beserta hasilnya.
4. **Belum terverifikasi:** hal yang memerlukan workflow berjalan atau pengujian perangkat.

Jangan mengklaim build, instalasi, atau pengujian perangkat berhasil jika belum dilakukan.

## Checklist

- [ ] Framework dikenali dengan benar: Flutter atau Expo.
- [ ] Prosedur khusus framework tidak tercampur.
- [ ] Jenis build dan target ABI sudah jelas.
- [ ] Versi toolchain mengikuti proyek dan dokumentasi resmi.
- [ ] Branch, submodule, dan lockfile ditangani dengan benar.
- [ ] Cache sesuai framework dan memiliki invalidasi yang tepat.
- [ ] Tidak ada SDK atau symlink yang dipaksakan tanpa bukti.
- [ ] Prebuild Expo tidak menghapus perubahan native.
- [ ] Flavor, signing, environment, dan entry point Flutter diperiksa jika relevan.
- [ ] APK ditemukan dan divalidasi sebelum upload.
- [ ] Laporan membedakan hasil teruji dan belum terverifikasi.
