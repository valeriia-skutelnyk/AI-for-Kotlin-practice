# Test architecture

This document describes the Appium UI suite in `appium-tests/`. Claims were checked by reading the cited sources on 2026-11-17; the suite was not run for this document. Paths are relative to the repository root; `K/` abbreviates `appium-tests/src/test/kotlin/`. Anything not confirmed from the repository is marked UNVERIFIED. Line numbers drift, so re-check a citation before relying on it.

## Layers

The five layers are named in `AGENTS.md:56-63` and `appium-tests/README.md:40-47`.

| Layer | Responsibility | Must not contain |
| --- | --- | --- |
| `rule/` | Session lifecycle, driver configuration, sandbox control and failure evidence (`AGENTS.md:58-59`). `AppiumTestCase` owns only the session lifecycle and the `step()` reporter (`K/rule/AppiumTestCase.kt:34-36`). | UI flows or element catalogs (`K/rule/AppiumTestCase.kt:34-36`). |
| `pages/` | Singleton `Element` catalogs (`AGENTS.md:60`), for example `K/pages/PhoneLoginPage.kt:4-9`. `Element` and `Device` also live here as UI primitives (`K/pages/Element.kt:26`, `K/pages/Device.kt:12`). | Assertions and flows (`AGENTS.md:60`; `appium-tests/README.md:49-50`). Observed: the word "assert" appears in `K/pages/` only in comments (`K/pages/Element.kt:54`, `K/pages/MapPage.kt:31`). |
| `actions/` | Interactions, flows, waits and assertions (`AGENTS.md:61`). Assertions live here (`K/rule/AppiumTestCase.kt:37-38`). Example: `K/actions/RegionActions.kt:9-23`. | Observed, not a stated rule: no `AppiumBy` or `By.` use outside `K/pages/Element.kt`. |
| `tests/` | JUnit scenarios written as named Allure steps (`AGENTS.md:62`). Conventions: camelCase methods, `@DisplayName`, `@AllureId`, `@Feature` and named `step` blocks (`appium-tests/README.md:50-51`). Example: `K/tests/OnboardingSmokeTest.kt:10-14`. | Observed, not a stated rule: tests import only `rule.AppiumTestCase` and `testdata.TestData` from the project, no `pages` or `actions` imports. |
| `testdata/` | Shared values (`AGENTS.md:63`); values live in one place (`K/testdata/TestData.kt:3-4`). | Not stated in the repository. |

Cross-cutting rules:

- Locators use Compose `testTag` resource IDs through `AppiumBy.id`; text XPath locators are forbidden (`AGENTS.md:65-66`). `Element.by()` is the only place that builds a locator (`K/pages/Element.kt:33`). `DriverFactory` sets `appium:disableIdLocatorAutocompletion` so tags stay plain resource IDs (`K/rule/DriverFactory.kt:123-125`).
- `startAuthorized` defaults to `true`; set it to `false` only when onboarding itself is under test (`AGENTS.md:69-70`; default at `K/rule/AppiumTestCase.kt:59`; overrides at `K/tests/OnboardingSmokeTest.kt:14` and `K/tests/PasskeyPromoTest.kt:11`).
- Everything above the driver is stateless: pages and actions resolve the current thread's driver per operation through `DriverFactory.current()` (`K/rule/AppiumTestCase.kt:32-36`; `K/pages/Element.kt:20-21, 30-31`).
- Protected infrastructure: `K/rule/DriverFactory.kt` and `K/pages/VariantLocator.kt` are listed in `scripts/protected-paths.txt:23, 27`. Do not edit `VariantLocator.kt` or add fallback locator attempts, and preserve the session safeguards in `DriverFactory.kt` (`AGENTS.md:19-23`). The safeguards are `enforceAppInstall` and `disableIdLocatorAutocompletion` (`scripts/protected-paths.txt:21-22`), set at `K/rule/DriverFactory.kt:125, 128`.
- `VariantLocator.kt` contains only `variantId(stable, redesign)`: it reads the `ui.variant` system property and fails on any other value (`K/pages/VariantLocator.kt:3-10`). No caller exists in the current sources (grep at time of writing).
- New tests must use unused repository Allure IDs and must not duplicate existing coverage (`AGENTS.md:104-105`).

## Where things live

| What | Directory |
| --- | --- |
| `rule/` | `appium-tests/src/test/kotlin/rule/` |
| `pages/` | `appium-tests/src/test/kotlin/pages/` |
| `actions/` | `appium-tests/src/test/kotlin/actions/` |
| `tests/` | `appium-tests/src/test/kotlin/tests/` |
| `testdata/` | `appium-tests/src/test/kotlin/testdata/` (`TestData.kt`) |

Source: `appium-tests/README.md:41-46`.

Test inventory: this document does not list test classes because the list changes. Find the current tests with `grep -rn "@AllureId" appium-tests/src/test/kotlin/tests`. `AGENTS.md:97-101` and `appium-tests/README.md:3-5` name the starter classes; that is a statement about the starter state, not a maintained registry. A search of `*.md`, `*.csv`, `*.json` and `*.txt` files for `AllureId` found no catalog file; it matched only the convention lines in `appium-tests/README.md:50` and `api-tests/README.md:29`.

Other locations:

- Shared base class: `K/rule/AppiumTestCase.kt`; it registers `FailureDigest` and `ArtifactsOnFailure` (`:47`).
- Gradle test configuration: APK path (`appium-tests/build.gradle.kts:26-31`), Allure results directory (`:37-43`), device pool and device evidence directory (`:44-57`).
- Runners: `scripts/run-suite.ps1` and `scripts/run-suite.sh` (`AGENTS.md:48-49`).

## Waits and retries

### Policy

- Do not use `Thread.sleep`; synchronize through `Element.waitFor`, `waitForGone` and narrowly justified `retryClick` calls (`AGENTS.md:66-67`; `appium-tests/README.md:51`). No `sleep` occurs under `K/` (grep at time of writing).
- Do not weaken assertions, increase timeouts or add retries without evidence that the test expectation is correct and the synchronization is the defect (`AGENTS.md:11-12`).
- Waits and retries live on `Element`; device-level primitives live on `Device` (`K/rule/AppiumTestCase.kt:39-40`).

### Helpers on `Element` (`K/pages/Element.kt`)

| Helper | What its body does | Intended use | Misuse to avoid |
| --- | --- | --- | --- |
| `waitFor(timeoutSec)` | `WebDriverWait` with `visibilityOfElementLocated(AppiumBy.id(testTag))`; returns the `WebElement` (`:35-39`). Default timeout is set by the `defaultTimeoutSec` constructor parameter (`:28`). | Wait for an element to be visible before reading or asserting on it, for example screen readiness at `K/actions/MapActions.kt:11-15` and `K/actions/SupportActions.kt:9-12`. | Raising the timeout to hide a defect (`AGENTS.md:11-12`). Using it to prove absence: it waits for visibility (`:35-37`); use `waitForGone` or `isPresent`. |
| `waitForGone(timeoutSec)` | `WebDriverWait` with `invisibilityOfElementLocated(by())`; returns the `Boolean` result of `until` (`:42-44`). It does not use `visibilityOfElementLocated`. | Wait for a transient element to disappear, for example `K/actions/RegionActions.kt:17-23`, `K/actions/NotificationActions.kt:21-27`. | Raising the timeout without evidence (`AGENTS.md:11-12`). UNVERIFIED: that it throws on timeout; that is Selenium behavior, not visible in this repository. |
| `click`, `sendKeys`, `clear`, `text` | Each locates the element with the same visibility wait as `waitFor` before acting (`:46-48`, `:74-84`, `:86-87`). | Normal interactions; no extra wait is needed before them. | Adding a sleep or an extra wait in front of them. |
| `retryClick(timeoutSec, message)` | `FluentWait` polling every 300 ms, ignoring only `NoSuchElementException`, `StaleElementReferenceException` and `ElementClickInterceptedException`; each poll runs `driver.findElement(by()).click()` (`:57-72`). It does not call `locate`, so it has no visibility wait, although the class KDoc says every operation except `isPresent` waits for visibility (`:22-24`). | A click that can hit a transient state mid-transition. Current callers are drawer taps: `K/actions/DrawerActions.kt:16, 22, 32-34` and `K/actions/DriverActions.kt:23`. Other failures are deliberately not retried (`:50-56`). | Using it as a general retry or as a fix for a flaky test (`AGENTS.md:11-12`). It is not a substitute for disabled animations (`K/actions/DrawerActions.kt:10-12`). Pass a specific `message`: it becomes the failure text (`:64`). |
| `isPresent()` | `driver.findElements(by()).isNotEmpty()`; no wait (`:89-90`). | Absence checks and polling loops, for example `K/actions/SupportActions.kt:12`, `K/actions/OrderActions.kt:40`, `K/actions/MapActions.kt:70, 74`. | Using it where a visible element is required. |

Per-element default timeouts are set where the element is declared (for example `K/pages/MapPage.kt:20`); call sites can override them (for example `K/actions/MapActions.kt:12-13`).

UNVERIFIED: the polling interval of `WebDriverWait` (the repository does not set it). UNVERIFIED: whether an Appium implicit wait applies to the `findElement` inside `retryClick`; `DriverFactory` sets none (`K/rule/DriverFactory.kt:115-128`), but the server default was not checked.

### Helpers on `Device` (`K/pages/Device.kt`)

None of these wait.

- `hideKeyboard()` calls `driver.hideKeyboard()` inside `runCatching`, so a failure is swallowed; a missing driver is still an error (`:18-23`).
- `pressBack()` presses the system Back key (`:29-34`).
- `pullDownToRefresh()` runs a `mobile: swipeGesture` (`:37-53`). Callers follow it with a wait or an assertion (`K/actions/MapActions.kt:96-99, 103-104`).

### Sandbox states are not a wait

`ConditionControl` sends an adb broadcast and falls back to `mobile: shell` through the Appium session (`K/rule/ConditionControl.kt:38-47`). A successful delivery does not prove the state took effect; the test must assert the observable effect (`K/rule/ConditionControl.kt:23-27`; `AGENTS.md:91-93`). Pattern: enable the state, `waitFor` the effect, then assert (`K/actions/RegionActions.kt:9-15`). The adb transport times out after 15 s (`K/rule/ConditionControl.kt:57`).

### Session-level retry (not UI synchronization)

`DriverFactory.createSessionWithHealthRetry` makes up to two session attempts and retries only when the failure message mentions `device offline`, `not in the list of connected devices` or `adbexec` (`K/rule/DriverFactory.kt:105-146, 177-182`). It waits for the device with `adb wait-for-device` (`:148-166`). The file is protected (`scripts/protected-paths.txt:23`); do not change it to fix a test.

## Failure artifacts

Two JUnit extensions are registered on the base class (`K/rule/AppiumTestCase.kt:47`). The exception handler runs before the digest callback, so the digest reuses the captured logcat (`K/rule/AppiumTestCase.kt:41-45`).

### `ArtifactsOnFailure` (`K/rule/ArtifactsOnFailure.kt`)

- Trigger: `TestExecutionExceptionHandler` on a failed test; it captures best-effort evidence and always rethrows the original failure (`:26-34`).
- Driver: `DriverFactory.current()`. With no live session it prints a message and captures nothing (`:37-42`).
- Directory: `build/reports/failures`, relative to the JVM working directory, created with `mkdirs` (`:44`). `appium-tests/README.md:59` gives `appium-tests/build/reports/failures`. `appium-tests/build.gradle.kts` sets no `workingDir`. UNVERIFIED: that the test JVM's working directory is `appium-tests/` (it would follow from the Gradle default, which is not stated in the repository).
- File stem: the test display name with characters outside `A-Za-z0-9._-` replaced by `_`, then `-<epoch millis>` (`:45-46`).
- Artifacts, in capture order (logcat first, because Appium logs the screenshot payload, `:49-50`):

| Artifact | File | Allure attachment | Source |
| --- | --- | --- | --- |
| Device logcat | `<stem>.logcat` | `Device logcat`, `text/plain` | `:51-66` |
| Screenshot | `<stem>.png` | `Failure screenshot`, `image/png` | `:68-77` |
| UI page source | `<stem>.xml` | `UI page source`, `application/xml` | `:79-88` |

- Each artifact is written to a file and attached to Allure; the two destinations are isolated, so one failing does not block the other or the remaining artifacts (`:97-131`).

### `FailureDigest` (`K/rule/FailureDigest.kt`)

- Trigger: `AfterTestExecutionCallback`; it does nothing when the test passed (`:13-14`).
- File: `build/reports/digests/test_log_{method}_{try}.txt` (`:9, 16, 48`). The method name comes from `context.testMethod`, falling back to the display name (`:18-21`). `try` is the highest existing suffix plus one, so earlier digests are kept (`:60-71`).
- Contents (`:26-46`): test name and class; failure type and first non-empty message line (`:151-159`); project stack frames only, with a fallback to raw frames (`:79-87`); paths of the three raw artifacts; distinct `resource-id` values from the page source (`:111-122`); a logcat slice.
- Raw artifacts are the newest files in `build/reports/failures` whose name starts with the display-name stem (`:96-107`); a missing file is reported as `(missing)` (`:109`).
- Logcat slice: reads the captured `.logcat` file instead of querying the driver again, because Appium drains the log buffer on read (`:124-130`). It starts at the last `FATAL EXCEPTION` whose next line names the app process `com.sandbox.qa`, otherwise it takes the tail of the log (`:161-176`). It drops screenshot payloads and collapses repeated Appium lines (`:197-244`).

### Other evidence written by the suite

- Allure step screenshot: after every successful `step`, `attachScreenState` attaches `Screen after step` to Allure; it is not written to `build/reports/failures`. A step that throws skips it, and `ArtifactsOnFailure` owns evidence then (`K/rule/AppiumTestCase.kt:102-116, 137-150`).
- Allure results: `appium-tests/build/allure-results` (`appium-tests/build.gradle.kts:37-43`; `appium-tests/README.md:57`).
- Device assignment log: `DeviceExecutionEvidence.record` appends one line per test to `<udid>.log` under `appium.device.evidence.dir` and does nothing when the property is unset (`K/rule/DeviceExecutionEvidence.kt:15-30`). Gradle sets the directory to `build/reports/device-execution` (`appium-tests/build.gradle.kts:51-57`).
- Report locations: JUnit XML, Gradle HTML, static Allure report, failure artifacts and digests (`appium-tests/README.md:55-60`). UNVERIFIED: which runner steps generate the static Allure report.
- Evidence limits: CI compiles this module but does not execute Appium tests (`AGENTS.md:107-112`; `appium-tests/README.md:66-67`). A bare cached `:appium-tests:test` result is not evidence (`appium-tests/README.md:34-36`).
