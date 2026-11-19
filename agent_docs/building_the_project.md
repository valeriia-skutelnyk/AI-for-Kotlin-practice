# Building the project

Environment: macOS 15.7.3, arm64. Observations are dated 2026-11-14 (bootstrap, Appium run) and 2026-11-17 (Gradle builds, ktlint, API run). UNVERIFIED means: read from repository files but not run on this machine. All Windows commands (`.ps1`, `gradlew.bat`, `npm.cmd`) are UNVERIFIED.

## Prerequisites

| Requirement | Value | Source |
| --- | --- | --- |
| JDK | 17 | `appium-tests/build.gradle.kts:6`, `api-tests/build.gradle.kts:7` (`jvmToolchain(17)`) |
| Gradle | 9.5.1, supplied by the wrapper (`./gradlew`), no separate install | `gradle/wrapper/gradle-wrapper.properties` |
| Node.js | 20 or newer | `scripts/bootstrap.sh` (check `Node.js 20+`) |
| Appium | 2.16.2 exact; the runner rejects any other server version | `package.json:7`, `scripts/run-suite.sh:42` |
| UiAutomator2 driver | 3.9.8 exact | `package.json:8` |
| Allure CLI | 2.43.0 exact | `package.json:6` |
| Android SDK | platform 36, build-tools 36.0.0 | `scripts/bootstrap.sh` (checks `Android API 36 platform`, `Android build-tools 36.0.0`) |
| Emulator (AVD) | name `Pixel_6`; image `system-images;android-36;google_apis;<ABI>`; ABI `arm64-v8a` on arm64/aarch64, otherwise `x86_64`; RAM 1536-4096 MB, default 2048 | `scripts/setup-emulator.sh:7, 14, 19-24` |
| Free disk | at least 8 GB to install SDK packages or create the AVD | `scripts/setup-emulator.sh:148, 163` |

Kotlin 2.3.20 and Android Gradle Plugin 9.2.0 are pinned and resolved by Gradle. [`build.gradle.kts:2-6`]

Setup commands, in order:

1. `npm ci` installs the pinned Node toolchain. Never run `npm install` or `npm audit fix`: they can move the pinned versions. [`package.json`] Observed: `added 872 packages`.
2. `./scripts/setup-emulator.sh` creates and boots the AVD. `CHECK_ONLY=true` only validates an existing AVD. [`scripts/setup-emulator.sh`] Observed: `CHECK_ONLY=true` printed `[OK] Pixel_6 uses API 36, arm64-v8a and 2048 RAM.`
3. `bash ./scripts/bootstrap.sh` checks the environment and prints `Passed: N, failed: M`; ready means `failed: 0`. [`scripts/bootstrap.sh`] Observed: `Passed: 14, failed: 0`.

Caveats:

- The AVD must be named exactly `Pixel_6`: the scripts default to it. [`scripts/bootstrap.sh:60`, `scripts/setup-emulator.sh:23`] Observed with an AVD whose registration file was named `Pixel_6_API_36`: bootstrap passed `Practice AVD Pixel_6` but failed `Booted practice emulator`; with `AVD_NAME=Pixel_6_API_36` the two results swapped.
- The AVD must be registered: `emulator -list-avds` must list `Pixel_6`, otherwise `emulator -avd Pixel_6` fails with `Unknown AVD name`. Observed.
- `setup-emulator.sh` fails with `.../Pixel_6.avd already exists!` when the AVD folder exists. Observed.
- `APPIUM_HOME` is unset by the Appium start script, so the repository-pinned driver is used. [`scripts/start-appium.sh:11`]
- Start Appium only through `scripts/start-appium.sh`. [`AGENTS.md:46-47`]

## Build commands

| Goal | Command | Source | Status |
| --- | --- | --- | --- |
| Stable debug APK | `./gradlew :app:assembleStableDebug` | `AGENTS.md:44` | run, `BUILD SUCCESSFUL` |
| Redesign debug APK | `./gradlew :app:assembleRedesignDebug` | `AGENTS.md:44-45` | run, `BUILD SUCCESSFUL` |
| Both flavors | `./gradlew :app:assembleDebug` | `.github/workflows/ci.yml:34` | run, `BUILD SUCCESSFUL` (2026-11-17, together with the two rows below) |
| Compile Appium tests only | `./gradlew :appium-tests:compileTestKotlin` | `.github/workflows/ci.yml:54` | run, `BUILD SUCCESSFUL` (2026-11-17) |
| Kotlin format check, before committing Kotlin changes | `./gradlew ktlintCheck` | `AGENTS.md:52` | run, `BUILD SUCCESSFUL` (2026-11-17) |

Flavors: `stable` (`isDefault = true`) and `redesign`. [`app/build.gradle.kts:23-27`] Use `gradlew.bat` on Windows (UNVERIFIED).

Artifact: `app/build/outputs/apk/<flavor>/debug/app-<flavor>-debug.apk`. [`scripts/run-suite.sh:92`] Observed: `app/build/outputs/apk/stable/debug/app-stable-debug.apk` and `app/build/outputs/apk/redesign/debug/app-redesign-debug.apk` exist (12229658 bytes each).

A direct `:appium-tests:test` run without `-Dapp.apk` uses `app/build/outputs/apk/stable/debug/app-stable-debug.apk`, even for the redesign flavor. [`appium-tests/build.gradle.kts:27-31`]

## Run the suite

### API tests

1. Start the backend and leave it running: `./gradlew :fake-api:run`. It listens on port 8080; override with `FAKE_API_PORT`. [`fake-api/README.md:13`] Observed: the Gradle progress bar stays at 83% while the server runs; this is normal.
2. In another terminal: `./gradlew :api-tests:test --rerun`. One class: add `--tests tests.ApiSmokeTest`. Observed: `ApiSmokeTest ... PASSED`, `BUILD SUCCESSFUL`.

The tests target `http://localhost:8080` unless `-Dapi.url=<url>` is passed. [`api-tests/build.gradle.kts:29-30`] Swagger UI at `http://localhost:8080/swagger` is the readiness check. [`scripts/run-suite.sh:28`]

### Appium UI suite

Canonical command: `./scripts/run-suite.sh` (macOS/Linux) or `.\scripts\run-suite.ps1` (Windows, UNVERIFIED). [`AGENTS.md:48`, `README.md:77`]

Must already be running (the runner exits with a message if not):

- fake-api on `127.0.0.1:8080`. [`scripts/run-suite.sh:28-29`]
- the pinned Appium on `127.0.0.1:4723`, started with `./scripts/start-appium.sh`. [`scripts/run-suite.sh:35-37`] Observed: `Appium v2.16.2`, driver `uiautomator2@3.9.8`.
- exactly one booted device; with several, set `DEVICE=<serial>`. [`scripts/run-suite.sh:51-68`] Observed with no emulator: `No booted emulator/device found. Start one and retry.`

One class: `bash ./scripts/run-suite.sh --tests tests.RideAndHistoryE2ETest`. Extra arguments are forwarded to Gradle. [`scripts/run-suite.sh:94`] Observed: that class `PASSED`.

Redesign flavor: `FLAVOR=redesign ./scripts/run-suite.sh` (default `stable`). [`scripts/run-suite.sh:11-15`] UNVERIFIED (not run).

Why the runner and not a bare Gradle task: it passes `--rerun` (`:91`), builds the selected flavor and passes `-Dapp.apk`, `-Dui.variant`, `-Dappium.devices` (`:92-94`), disables animations (`:76`) and resets sandbox state through `ConditionReceiver` (`:80`).

CI does not run the Appium suite: a search for `appium|run-suite` in the workflow finds only the compile step and a comment saying Appium tests need a local emulator and Appium server. [`.github/workflows/ci.yml:52-54`] CI also starts fake-api and runs the API tests. [`.github/workflows/ci.yml:38, 50`] A green CI run is therefore not UI evidence.

## Verify it worked

Starter inventory: six Appium cases (`OnboardingSmokeTest` 3, `PasskeyPromoTest` 2, `RideAndHistoryE2ETest` 1, counted via `@Test`) [`AGENTS.md:97-101`] and four API cases (`ApiSmokeTest` 1, `RideLifecycleApiTest` 3, counted via `@Test`). [`AGENTS.md:103-104`]

A green exit code is not proof. Check that result files exist and the counts are non-zero.

Appium, observed on macOS:

| Evidence | Path |
| --- | --- |
| JUnit XML | `appium-tests/build/test-results/test/TEST-tests.<Class>.xml` |
| Allure report | `appium-tests/build/reports/allure-report/index.html` (open through `python3 -m http.server`; opening it as `file://` shows `500 Failed to fetch`) |
| Run log | `suite-run.log` at the repository root |
| Failure artifacts, only after a failure | `appium-tests/build/reports/failures` [`ArtifactsOnFailure.kt:44`] |
| Failure digests, only after a failure | `appium-tests/build/reports/digests` [`FailureDigest.kt:16`] |

- `run-suite.ps1` appends `Appium result: N passed, F failed, E errors, S skipped (T total) on <device>` to `suite-run.log` and fails with `No tests executed` when the total is 0. [`scripts/run-suite.ps1:120, 133`] (Windows, UNVERIFIED)
- `run-suite.sh` contains neither the `Appium result` line nor a zero-test check. [`grep "Appium result\|No tests executed"` over both scripts matched only `run-suite.ps1`] On macOS/Linux check that `TEST-*.xml` exists and that its `tests` counts match the inventory above; the console also prints one `PASSED` line per test. Whether `suite-run.log` after a complete bash run lacks the `Appium result` line: UNVERIFIED (the log from the run that stopped at the device check has none).
- `.agents/skills/run-appium-suite/SKILL.md:7, 77` says the runner checks the real result in `suite-run.log` on every supported OS and tells the reader to read the final `Appium result` line. For the bash runner no such line exists in the script, so do not rely on that line on macOS/Linux.

API: observed `api-tests/build/test-results/test/TEST-tests.ApiSmokeTest.xml` after a run, and a `PASSED` line in the console. No repository script totals API counts.

### Sandbox states

The adb return code does not prove that a state applied: verify the visible or API-level effect. Enable states after the Appium session starts, because session creation relaunches the app process. [`AGENTS.md:91-93`]