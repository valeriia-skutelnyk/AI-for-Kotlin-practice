# Suite baseline report

Date: 2026-11-18

## Environment

| Item | Value |
|---|---|
| OS and version | macOS 15.7.3 (verified with `sw_vers`; arm64 per `environment_notes.md`) |
| Emulator: AVD name and API level | Pixel_6, API 36 (android-36, google_apis, arm64-v8a). AVD name and API level confirmed right after the run on 2026-11-18 with `adb -s emulator-5554 emu avd name` (Pixel_6) and `adb shell getprop ro.build.version.sdk` (36); image and ABI come from `environment_notes.md` |
| Appium server version | 2.16.2 (`npx --no-install appium --version`; matches `environment_notes.md`) |

## Source state

| Item | Value |
|---|---|
| Repository revision | cadca2f (full: cadca2fb442682e0eda7df9a7199cf5ddfcf1beb), branch `main` |
| Working tree | Not clean. Modified: `agent_docs/building_the_project.md`, `agent_docs/test_architecture.md`. Untracked: `baseline_report.md`, `environment_notes.md`, `tc103/`, `tc103_review.md`. No modified files under `app/`, `fake-api/`, `appium-tests/` or `api-tests/`. This is the state at the time of reporting; the state during the runs is unknown |

## App build variant

stable (no `FLAVOR` set). `suite-run.log` confirms "Building stable debug APK".

## Command used

```
bash ./scripts/run-suite.sh
```

Started 10:26:05, ended 10:28:20 EET (wall clock 2m 15s). Full suite, with no `--tests` filter. The `environment_notes.md` command lines use `--tests` filters and describe the setup check, not this run.

```
./gradlew :fake-api:run
./gradlew :api-tests:test --rerun
node_modules/.bin/allure generate api-tests/build/allure-results -o api-tests/build/reports/allure-report --clean
```

`:fake-api:run` was already running from an earlier session. `:api-tests:test --rerun` ran 10:29:30-10:29:49 (19 s wall clock). The third command generated the API Allure report.

## Results: Appium suite

Source: `appium-tests/build/test-results/test/` (3 JUnit XML files) and `suite-run.log`.

| Total | Passed | Failed | Skipped |
|---|---|---|---|
| 6 | 6 | 0 | 0 |

Check: 6 = 6 + 0 + 0. Failed includes execution errors; the JUnit files report `failures="0" errors="0"`.

Duration: 1m 44s (104,386 ms), from the Allure report summary (`appium-tests/build/reports/allure-report/widgets/summary.json`). The summed test time is 79.2 s. The JUnit suite times total 109.4 s (60.726 + 32.506 + 16.185).

## Results: API suite

Source: `api-tests/build/test-results/test/` (2 JUnit XML files).

| Total | Passed | Failed | Skipped |
|---|---|---|---|
| 4 | 4 | 0 | 0 |

Check: 4 = 4 + 0 + 0. Failed includes execution errors; the JUnit files report `failures="0" errors="0"`.

Duration: 12.2 s (12,187 ms), from the Allure report summary (`api-tests/build/reports/allure-report/widgets/summary.json`). The summed test time is 7.1 s.

## Allure sources

- Appium report: `appium-tests/build/reports/allure-report/index.html`
- API report: `api-tests/build/reports/allure-report/index.html`

## Cross-suite overview

- UI coverage visible in the report: 6 tests in 3 classes. `OnboardingSmokeTest` (3): ride options show prices after search, full onboarding opens the map, wrong OTP shows an error. `PasskeyPromoTest` (2): skip, and create a passkey, each continuing to location onboarding. `RideAndHistoryE2ETest` (1): search a ride then check order history.
- API coverage visible in the report: 4 tests in 2 classes. `ApiSmokeTest` (1): valid phone and OTP issue a token that opens the data endpoints. `RideLifecycleApiTest` (3): a completed ride becomes the newest order, `driver_not_found` rejects an order without changing history, a cancelled ride is not added to order history.
- Failures, skips, or slow-test observations: no failures and no skips in either suite. The slowest Appium test was "Ride options show prices after search" at 22.9 s. The slowest API test was "A completed ride becomes the newest order" at 2.0 s.

## Known issues and flaky observations

None in this run. Unverified items:

- The `fake-api` server was started in an earlier session, so its start time and state before this run are unknown.

## Next action

Record the AVD name and API level automatically in the runner output (or `suite-run.log`) so the next baseline can verify the emulator instead of relying on setup notes.
