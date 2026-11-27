---
name: penyiapan-workflow-android
description: Gunakan saat pengguna meminta membuat, memperbaiki, membandingkan, atau menganalisis GitHub Actions dan proses build APK Android untuk proyek Flutter maupun React Native dengan Expo, termasuk Expo Development Build, EAS, ABI, SDK, cache, dan validasi artefak.
---

# Penyiapan Workflow Android

## Tujuan dan cakupan

Menyiapkan pipeline Android yang menghasilkan APK valid untuk dua framework:

- **Flutter:** aplikasi yang dibangun dengan Flutter dan Dart.
- **React Native dengan Expo:** aplikasi yang dibangun dengan JavaScript/TypeScript, Expo, dan React Native.

Skill ini mengatur GitHub Actions, konfigurasi build Android, cache, ABI, dan artefak. Prosedur Flutter dan Expo wajib dipisahkan. Jangan menerapkan perintah, konfigurasi, atau asumsi satu framework ke framework lainnya.

Jalur build yang mungkin digunakan:

- GitHub Actions menjalankan toolchain framework dan Gradle.
- EAS Build cloud untuk proyek Expo jika memang dipilih.
- Layanan build lain hanya jika diminta atau sudah ditetapkan proyek.

## Pemicu penggunaan

Gunakan saat pengguna meminta:

- Membuat, memperbaiki, atau memeriksa workflow APK Android.
- Menyiapkan APK development, debug, preview/internal, atau production.
- Membuat Expo Development Build, bukan sekadar menjalankan Expo Go.
- Membatasi arsitektur APK, terutama `arm64-v8a`.
- Menyamakan workflow dengan repositori referensi.
- Memperbaiki kegagalan SDK, Java, Gradle, Flutter, Expo Prebuild, cache, upload artefak, atau APK yang gagal dipasang.

## Aturan wajib

1. Baca aturan repo, memori proyek, dan skill terkait sebelum bekerja.
2. Identifikasi framework berdasarkan file proyek, bukan dugaan. Flutter umumnya memiliki `pubspec.yaml`; Expo memiliki `package.json` dan konfigurasi Expo.
3. Periksa branch yang tersedia. Jika repo tidak memiliki `master`, gunakan `main`. Jangan mengasumsikan nama branch.
4. Jika pengguna menyebut repo referensi, baca workflow dan konfigurasi relevan dari repo referensi serta repo target. Jangan menyalin mentah konfigurasi yang khusus untuk framework atau versi tertentu.
5. Jangan mengarang versi Flutter, Dart, Node.js, Bun, Expo SDK, Java, Android SDK, NDK, Build Tools, Gradle, Android Gradle Plugin (AGP), atau GitHub Actions. Ambil dari konfigurasi proyek dan dokumentasi resmi yang sesuai.
6. Jangan memaksakan Android API terbaru atau membuat symlink SDK sebagai solusi umum. Gunakan paket SDK yang benar-benar tersedia dan kompatibel dengan toolchain proyek.
7. Jangan menganggap konfigurasi EAS mengendalikan build GitHub Actions yang menjalankan Gradle secara langsung.
8. Jangan mengklaim APK berhasil hanya karena task build sukses. Periksa berkas APK, metadata, ABI, dan hasil upload.
9. **Jangan menjalankan build Flutter, Gradle, Android, atau EAS secara lokal.** Gunakan GitHub Actions atau EAS Build cloud sesuai arsitektur proyek. Jangan memakai `eas build --local`, `flutter build`, `gradlew assemble*`, atau `expo run:android` untuk menguji build lokal.
10. Patuhi larangan mengubah workflow tertentu, melakukan commit, atau mengubah branch yang ditetapkan aturan repo. Jangan melakukan perubahan saat pengguna hanya meminta analisis.
11. Jika repo menggunakan submodule, pastikan SHA submodule sesuai gitlink yang dipatok oleh commit parent/workspace. Jangan otomatis mengambil HEAD terbaru submodule kecuali workflow memang mengharuskannya.
12. Jika ada peringatan deprecation dari GitHub Actions, periksa dokumentasi dan rilis resmi action tersebut. Jangan sekadar menaikkan versi satu tingkat atau menyalin versi dari workflow lama.

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

- **Expo Go:** aplikasi berjalan di runtime Expo Go. Ini bukan APK Development Build milik proyek.
- **Expo Development Build:** aplikasi Android mandiri yang menyertakan `expo-dev-client`; bukan Expo Go.
- **Debug/development:** mode build untuk pengembangan dan debugging sesuai kemampuan framework.
- **Preview/internal:** build untuk pengujian atau distribusi internal; belum tentu Development Build.
- **Release/production:** build untuk distribusi pengguna.

Untuk Flutter, tentukan mode `debug`, `profile`, atau `release` berdasarkan tujuan. Jangan menyebut APK release sebagai development build.

Untuk Expo Development Build:

- Pastikan `expo-dev-client` menjadi dependency dan kompatibel dengan Expo SDK proyek.
- Untuk EAS, periksa profil yang menggunakan `developmentClient: true` dan konfigurasi distribusi internal.
- Untuk GitHub Actions + Gradle, pastikan dependency terpasang sebelum Prebuild dan build.
- Nama profil `development` saja tidak membuktikan bahwa hasilnya Development Build.
- Jangan menyebut Development Build sebagai Expo Go. Development Build memakai binary native proyek sendiri dan digunakan bersama development server.

## 3. Prosedur build Flutter

1. Siapkan Flutter SDK pada versi yang ditentukan proyek di runner CI.
2. Siapkan Java dan Android SDK yang kompatibel dengan AGP/Gradle proyek.
3. Pulihkan dependency menggunakan lockfile. Gunakan `flutter pub get --enforce-lockfile` jika didukung versi Flutter proyek; jika tidak, gunakan perintah yang kompatibel dan jelaskan alasannya.
4. Periksa flavor, entry point, dart-define, signing, dan konfigurasi environment yang diperlukan.
5. Bangun APK sesuai mode dan target ABI yang diminta melalui CI.
6. Temukan dan validasi hasil APK sebelum upload.

Contoh perintah arm64-only untuk Flutter, sesuaikan dengan flavor dan entry point proyek:

```bash
flutter build apk --release --target-platform android-arm64
```

Perintah tersebut adalah referensi konfigurasi untuk runner CI, bukan instruksi untuk menjalankan build lokal.

Untuk APK terpisah per ABI, Flutter menyediakan opsi `--split-per-abi`. Jangan gunakan opsi tersebut jika pengguna meminta satu APK universal atau jika alur distribusi proyek mengharuskan satu berkas tertentu. Periksa nama dan lokasi output aktual, jangan menebaknya.

Jika proyek memakai flavor, gunakan sintaks flavor yang sesuai konfigurasi proyek, misalnya `--flavor` dan `-t` bila diperlukan. Jangan menambahkan flavor atau dart-define yang tidak ditemukan di repo atau tidak diberikan pengguna.

## 4. Prosedur build React Native dengan Expo

### GitHub Actions + Gradle

1. Checkout branch/commit yang benar. Checkout submodule secara rekursif hanya jika proyek memang menggunakannya; pastikan SHA submodule sesuai commit parent.
2. Siapkan Node.js atau Bun sesuai lockfile dan konfigurasi proyek.
3. Pulihkan dependency secara reproducible, misalnya `npm ci` atau perintah package manager yang sesuai.
4. Siapkan Java dan Android SDK sesuai kompatibilitas Expo SDK, AGP, dan Gradle.
5. Jalankan Expo Prebuild hanya jika proyek menggunakan CNG dan direktori native perlu dihasilkan.
6. Jalankan Gradle Wrapper dengan task build yang sesuai jenis APK.
7. Validasi APK dan upload artefak.

Hindari `expo prebuild --clean` jika direktori native memiliki perubahan manual yang harus dipertahankan. Jika proyek mengelola `android/` secara langsung (bare workflow), jangan menjalankan Prebuild tanpa alasan yang terbukti.

### EAS Build cloud

Jika pengguna memilih EAS:

- Pastikan profil yang dipanggil tersedia dan konfigurasi `eas.json` valid.
- Untuk Development Build, periksa `developmentClient: true` dan pastikan `expo-dev-client` terpasang.
- Untuk distribusi internal, periksa `distribution: "internal"`.
- Android EAS secara default dapat menghasilkan AAB. Jika pengguna perlu memasang file langsung ke perangkat, pastikan profil menghasilkan APK melalui konfigurasi yang didukung, misalnya `android.buildType: "apk"` atau `android.gradleCommand` yang memang menghasilkan APK.
- Jangan menganggap `distribution: "internal"` saja selalu membuktikan format output. Periksa hasil build sebenarnya.
- Jika membatasi ABI, pastikan konfigurasi Gradle atau opsi build yang digunakan EAS benar-benar menerapkannya.
- Gunakan secret `EXPO_TOKEN` untuk autentikasi CI bila diperlukan. Jangan mengekspos token pada log atau file.
- Jika APK harus menjadi artefak GitHub Actions, unduh hasil EAS ke runner lalu unggah file APK yang benar. Memulai EAS Build tidak otomatis membuat APK di workspace runner.
- Jangan gunakan `eas build --local`; gunakan layanan EAS Build cloud.

Dokumentasi resmi:

- https://docs.expo.dev/build/
- https://docs.expo.dev/build-reference/apk/
- https://docs.expo.dev/eas/json/

## 5. Konfigurasi GitHub Actions yang berlaku umum

1. Gunakan trigger sesuai permintaan. Untuk build manual gunakan `workflow_dispatch`; jangan menambah trigger otomatis tanpa kebutuhan.
2. Gunakan runner yang sesuai dengan kebutuhan toolchain.
3. Checkout branch/commit yang benar dan submodule hanya jika diperlukan.
4. Siapkan toolchain framework yang tepat: Flutter/Dart untuk Flutter; Node/Bun dan Expo untuk Expo.
5. Siapkan Java dan Android SDK sesuai konfigurasi native proyek.
6. Aktifkan cache dependency framework dan Gradle. Cache key harus memperhitungkan lockfile serta file konfigurasi build yang relevan.
7. Gunakan action versi stabil yang masih didukung. Periksa halaman rilis resmi sebelum mengubah versi. Jangan menyalin versi action lama dari workflow referensi tanpa pemeriksaan.
8. Pulihkan dependency menggunakan lockfile.
9. Jalankan perintah build yang benar untuk framework dan jenis APK.
10. Temukan APK dengan pemeriksaan yang gagal jika file tidak ditemukan.
11. Unggah hanya hasil yang valid. Gunakan `if-no-files-found: error`; jangan memakai `if: always()` untuk menyamarkan kegagalan build sebagai keberhasilan upload.

### Versi GitHub Actions dan deprecation

- Action major version tidak sama dengan versi Java yang diinstal. Contohnya, `actions/setup-java@v6` dapat digunakan untuk memasang JDK 17 jika proyek mewajibkannya.
- **Baseline per 27 November 2026:** dokumentasi resmi `actions/setup-java` menyatakan versi v1 sampai v4 deprecated dan versi stabil terbaru berada pada major v6. Jangan menyarankan migrasi ke v5 tanpa memeriksa rilis terbaru.
- Setiap kali workflow diedit, cek ulang status action pada dokumentasi/rilis resminya. Baseline versi dapat berubah.
- Jika memakai self-hosted runner, periksa persyaratan minimum versi runner dari action yang dipakai, termasuk runtime Node.js yang dibutuhkan action.
- Jangan mengubah versi JDK, Gradle, atau AGP hanya karena versi action berubah. Masing-masing memiliki kompatibilitas terpisah.

Pilih cache secara spesifik:

- Flutter: cache pub dan cache Gradle.
- Expo: cache package manager dan cache Gradle.
  Jangan mengasumsikan cache Android SDK/NDK diperlukan; tambahkan hanya jika manfaat dan key invalidasinya jelas. Hindari menambahkan dua mekanisme cache Gradle yang tumpang tindih tanpa alasan.

## 6. ABI dan kompatibilitas perangkat

Jika pengguna meminta arm64 saja:

- Target ABI harus `arm64-v8a`.
- Flutter: gunakan target platform `android-arm64`; verifikasi output APK.
- React Native/Expo: gunakan konfigurasi arsitektur React Native atau `abiFilters` yang benar-benar berlaku pada versi proyek.
- Jangan menyimpulkan ABI dari nama file artefak.
- Periksa isi APK, terutama direktori native library seperti `lib/arm64-v8a/`, dan pastikan ABI lain tidak ikut terpaket bila targetnya arm64-only.
- Jangan menambah ABI lain tanpa alasan kompatibilitas dan persetujuan pengguna.
- Jika pengguna menyebut tipe perangkat tertentu, verifikasi arsitektur CPU, versi Android/API, dan `minSdkVersion` perangkat tersebut sebelum menyimpulkan APK kompatibel.

Contoh parameter Gradle React Native (bukan perintah universal; verifikasi dukungan versi proyek):

```bash
./gradlew :app:assembleDebug -PreactNativeArchitectures=arm64-v8a
```

Contoh tersebut hanya untuk prosedur CI React Native/Expo, bukan Flutter dan bukan perintah build lokal.

Dokumentasi referensi React Native: https://reactnative.dev/docs/build-speed

## 7. Android SDK dan kegagalan versi

Saat terjadi kegagalan SDK:

1. Ambil `compileSdk`, `targetSdk`, `minSdk`, Build Tools, NDK, AGP, dan Gradle dari konfigurasi native yang benar-benar digunakan.
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
4. Untuk submodule, bandingkan SHA yang dipatok oleh commit parent/workspace; jangan membandingkan dengan HEAD branch submodule yang belum tentu digunakan.
5. Pisahkan pola umum yang dapat disamakan dari konfigurasi yang harus berbeda.
6. Terapkan hanya perubahan yang didukung bukti. Nyatakan dengan jelas jika ada bagian yang belum dapat diverifikasi.

## 9. Validasi APK

Sebelum menyatakan workflow berhasil, periksa:

- Proses build benar-benar berakhir sukses.
- APK ditemukan pada lokasi output aktual dan ukurannya lebih dari nol.
- APK dapat dibaca sebagai arsip ZIP dan memiliki struktur APK yang benar.
- Application ID/package name sesuai konfigurasi aplikasi.
- Mode/jenis build sesuai permintaan.
- ABI di dalam APK sesuai target.
- `minSdkVersion` kompatibel dengan perangkat target.
- Signing sesuai kebutuhan distribusi.
- Path upload cocok dengan lokasi output.
- Upload gagal secara jelas jika APK tidak ditemukan.
- Untuk Expo/EAS, pastikan file yang diunduh benar-benar APK, bukan AAB yang salah diberi ekstensi.

Gunakan `apkanalyzer`, `aapt`, atau alat Android lain yang tersedia pada runner untuk memeriksa metadata. Pemeriksaan ukuran dan nama file saja tidak membuktikan APK valid. Jangan menganggap APK kecil otomatis rusak atau APK besar otomatis benar.

### Diagnosis pesan "There was a problem parsing the package"

Jangan langsung menyimpulkan penyebabnya. Periksa secara berurutan:

1. Pastikan file yang diunduh adalah APK, bukan AAB, HTML error page, atau arsip lain.
2. Periksa integritas ZIP/APK dan pastikan proses build serta upload tidak terpotong.
3. Periksa `minSdkVersion` dibandingkan versi Android perangkat.
4. Periksa ABI native library di dalam APK dibandingkan arsitektur perangkat.
5. Periksa AndroidManifest, package name, dan signature APK.
6. Periksa apakah APK hasil build sesuai varian/flavor yang dimaksud dan bukan artefak dari job lama.

Ikon aplikasi yang tidak tampil merupakan masalah yang perlu diperiksa terpisah pada manifest, launcher activity, dan resource icon. Ikon yang hilang sendiri tidak membuktikan APK rusak. Demikian pula, ukuran file saja tidak cukup untuk menentukan validitas APK.

Instalasi dan pengujian pada perangkat fisik hanya boleh dinyatakan berhasil jika benar-benar dilakukan. Karena build tidak boleh dilakukan lokal, gunakan hasil workflow cloud dan bukti log/artefak untuk analisis; jangan mengklaim instalasi perangkat telah diuji.

## 10. Format laporan

Sampaikan:

1. **Temuan:** masalah dan lokasi file.
2. **Perubahan:** file yang diubah dan alasan teknis.
3. **Validasi:** pemeriksaan yang benar-benar dijalankan beserta hasilnya.
4. **Belum terverifikasi:** hal yang memerlukan workflow berjalan atau pengujian perangkat.

Jangan mengklaim build, instalasi, atau pengujian perangkat berhasil jika belum dilakukan.

## Rujukan resmi

Gunakan dokumentasi sesuai framework dan versi proyek. Utamakan dokumentasi resmi dan halaman rilis upstream:

- GitHub Actions setup-java: https://github.com/actions/setup-java
- Flutter Android deployment: https://docs.flutter.dev/deployment/android
- Expo APK build: https://docs.expo.dev/build-reference/apk/
- Expo EAS configuration: https://docs.expo.dev/eas/json/
- Expo Continuous Native Generation: https://docs.expo.dev/workflow/continuous-native-generation/
- React Native Android build architecture: https://reactnative.dev/docs/build-speed

## Checklist

- [ ] Aturan repo, memori proyek, dan skill terkait telah dibaca.
- [ ] Framework dikenali dengan benar: Flutter atau Expo.
- [ ] Prosedur khusus framework tidak tercampur.
- [ ] Jenis build dan target ABI sudah jelas.
- [ ] Versi toolchain mengikuti proyek dan dokumentasi resmi.
- [ ] Versi GitHub Actions masih didukung dan tidak deprecated.
- [ ] Branch, submodule, dan lockfile ditangani dengan benar.
- [ ] Cache sesuai framework dan memiliki invalidasi yang tepat.
- [ ] Tidak ada build lokal, SDK, atau symlink yang dipaksakan.
- [ ] Prebuild Expo tidak menghapus perubahan native.
- [ ] Flavor, signing, environment, dan entry point Flutter diperiksa jika relevan.
- [ ] Expo Development Build memakai `expo-dev-client` dan bukan disamakan dengan Expo Go.
- [ ] Format APK dipastikan; AAB tidak salah dianggap APK.
- [ ] APK ditemukan dan divalidasi sebelum upload.
- [ ] Laporan membedakan hasil teruji dan belum terverifikasi.
      kopisatu
