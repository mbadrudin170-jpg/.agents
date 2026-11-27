---
name: setup-android-workflow
description: Setup GitHub Actions workflow for Expo/React Native Android development APK (arm64 only) without SDK mismatch or artifact path issues.
---

# Setup Android Workflow

## Trigger

- Use `workflow_dispatch` only to avoid unwanted automatic builds.

## Steps

1. Checkout with submodules recursive if needed.
2. Setup Node 20 and Java 17.
3. Install Android SDK components using `platforms;android-37.0` (not `android-37`).
4. Create symlink:
   `ln -sfn "$ANDROID_HOME/platforms/android-37.0" "$ANDROID_HOME/platforms/android-37"`
5. Prebuild Android with `npx expo prebuild --platform android --clean`.
6. Build arm64 only:
   `./gradlew assembleDebug -PreactNativeArchitectures=arm64-v8a -Pandroid.injected.build.abi=arm64-v8a`
7. List APK outputs before upload:
   `find android/app/build/outputs -name '*.apk' -print`
8. Upload APK artifact with path `android/app/build/outputs/**/*.apk`.

## Notes

- GitHub runners expose API 37 as `android-37.0`, causing `android-37` lookup failures.
- ABI-specific output may be nested, so use a broad artifact path.
- Keep NDK and build-tools versions compatible with Expo SDK.
