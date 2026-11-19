# environment_notes.md

- OS + version:            macOS 15.7.3
- Host CPU architecture:   arm64
- JDK:                     openjdk version "17.0.21" 2026-10-20 (Temurin-17.0.21+9)
- Android platform:        android-36
- Build-tools:             36.0.0
- System image:            android-36 / google_apis / arm64-v8a
- AVD name:                Pixel_6
- Node.js:                 v26.3.0
- Appium:                  2.16.2
- uiautomator2 driver:     uiautomator2@3.9.8 [installed (npm)]
- APK build variant:       stableDebug
- Test run command:        bash ./scripts/run-suite.sh --tests tests.RideAndHistoryE2ETest
                           ./gradlew :api-tests:test --rerun --tests tests.ApiSmokeTest
- Allure CLI:              2.43.0
- Date of setup:           2026-11-14
