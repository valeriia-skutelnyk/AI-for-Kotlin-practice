# AI Policy

Version: 0.1, week-1 draft. Owner: Valeriia.
Scope: this repository, including the Appium and API test suites. Applies to any AI agent, such as Claude Code, Cursor, or Codex, and to any human applying an AI-generated diff.

Ground rule: every line below must be backed by something observed in this repo, and every rule must name how its violation is detected. A rule whose violation cannot be detected is not a rule, it is a wish.

Week 1 observations this policy rests on:
1. A skill claimed more than the script does. `.agents/skills/run-appium-suite/SKILL.md:7, 77` says the runner checks the real result in `suite-run.log` on every OS and to read the `Appium result` line. `scripts/run-suite.sh` never writes that line and has no zero-test guard; only `scripts/run-suite.ps1:120, 133` does. An agent in a fresh session repeated the claim (consumer test of `agent_docs/building_the_project.md`).
2. CI is green without a single Appium test. `.github/workflows/ci.yml:52-54` only compiles `appium-tests`; it runs the API suite (`ci.yml:38, 50`) but never the UI suite (`AGENTS.md` section CI).
3. Success without proof of execution. `bash ./scripts/run-suite.sh` printed `BUILD SUCCESSFUL` for the APK build and then stopped with `No booted emulator/device found. Start one and retry.`; `bash ./scripts/bootstrap.sh` reported `[FAIL] Booted practice emulator` while an emulator was running under a different AVD name.

## Allowed changes

What the agent may generate or modify, and the proof required before the change is accepted. "Allowed" means the agent may draft; a human merges.

| Allowed change | Condition: proof required | How a violation is detected |
|---|---|---|
| New tests, page objects, and actions under `appium-tests/src` | Output of a fresh local runner (`bash ./scripts/run-suite.sh` or `.\scripts\run-suite.ps1`, which pass `--rerun`) against the target APK, attached to the PR. The `TEST-*.xml` files under `appium-tests/build/test-results/test` must show `failures="0" errors="0" skipped="0"`, come from an unfiltered run (no `--tests`) whose total is at least the baseline total plus the new tests, and be newer than the APK build (`ls -l`). The PR description states the commit SHA and the `stable` or `redesign` variant. Exit code 0 alone is not proof, because `run-suite.sh` has no zero-test guard | Reviewer rejects a PR with no run output, a cached or red run, or XML counts of 0 |
| New clients, models, test data, and tests under `api-tests/src` | `fake-api` is running and a fresh `./gradlew :api-tests:test --rerun` shows `BUILD SUCCESSFUL` plus a non-zero count in `api-tests/build/test-results/test/TEST-*.xml` with `failures="0" errors="0" skipped="0"`, an unfiltered run, a total of at least 4 plus the new tests, and files newer than the run start | Reviewer rejects missing, red, filtered, connection-failed, or 0-test output |
| Refactors of helpers, waiters and scaffolding (`appium-tests/src/test/kotlin/pages/Element.kt`, `rule/`, `actions/`; `api-tests/src/test/kotlin/client/`) | Fresh runs of both affected suites whose totals match `baseline_report.md` (Appium 6/6/0/0, API 4/4/0/0 at revision `cadca2f`); a lower total or any removed or skipped existing test is a human gate, not an explanation the agent writes itself; the human updates `baseline_report.md` in a separate PR | Reviewer compares the attached totals with `baseline_report.md`; a lower total or any mismatch with it goes to a human gate |
| Triage drafts, bug reports, and docs under `agent_docs/` | Every claim cites `file:line` or a quoted log line that was verified with `grep -nF`; anything else is marked `UNVERIFIED` | Reviewer spot-checks three citations against the source; a claim sourced only from `README.md` or prose is rejected |

## Forbidden changes

What the agent must never touch, even when it would make the suite green. Each row has a detector.

| Forbidden change | Why it is forbidden | How a violation is detected |
|---|---|---|
| Editing `app/` or `fake-api/` for any reason in a test task (including adding a `testTag` or fixing a product bug) | The test goes green because the observed product changed, not because the test was fixed: it masks a regression | `git diff --name-only origin/main` and `git status --short` list any path under `app/` or `fake-api/`; both are in `scripts/protected-paths.txt`. Any hit is rejected unless a human explicitly requested a product change in the task |
| Editing `.github/workflows/`, `package.json`, `package-lock.json`, `appium-tests/src/test/kotlin/rule/DriverFactory.kt` or `appium-tests/src/test/kotlin/pages/VariantLocator.kt` | Greening a build by editing the pipeline is not a fix; these files pin the toolchain and decide which APK is installed, so a change can hide a wrong build (`scripts/protected-paths.txt`, `AGENTS.md`) | The same `git diff --name-only` check against every path in `scripts/protected-paths.txt`; these are protected, not free to edit: any hit needs a run traceable to the target APK and explicit human approval, which is a named reviewer's comment on the PR |
| `Thread.sleep` or any fixed sleep in `appium-tests/src` | Hides the real timing problem; the framework has `Element.waitFor`, `waitForGone` and `retryClick` (`AGENTS.md`) | `grep -rn 'Thread.sleep' appium-tests/src api-tests/src` must return nothing (on the current revision it returns nothing) |
| Text-based or XPath locators | Break on a copy change or localization; the convention is `AppiumBy.id` with `testTag` ids (`AGENTS.md`) | `grep -rniE 'xpath\|By\.xpath' appium-tests/src` must return nothing, and `grep -rn 'AppiumBy\|By\.' appium-tests/src` must match only `pages/Element.kt` |
| Weakening an assertion, raising a timeout, or adding `retryClick` without evidence | The check stops seeing the bug it exists for (`AGENTS.md`) | Review every changed `assert*` line against what the screen renders and every new `retryClick` or timeout against a cited failure artifact |
| Stating a count or fact from prose docs | Docs go stale and skills can overstate (observation 1) | The claim must cite code or a run artifact; a reviewer rejects claims whose only source is `README.md` or a `SKILL.md` |
| Sending real secrets, tokens, credentials or personal data to a model | Protected data may leave its approved boundary | Reviewers check committed prompts and fixtures for provenance; only synthetic fixtures such as `TestData.kt` values are used; if safety cannot be verified, stop and do not send it |

## Human gates

Actions that always require an explicit human decision, no matter how confident the agent is.

- Merging any PR: a human reviews the full diff and the attached local run output before merge.
- Any change to a path in `scripts/protected-paths.txt`, or to shared infrastructure (`appium-tests/src/test/kotlin/rule/`, `pages/Element.kt`, `api-tests/src/test/kotlin/client/`): the human approves the scope before the change and the diff after.
- Removing, skipping or merging an existing test (the total drops below `baseline_report.md`): the human decides.
- Accepting a "flaky test" verdict, weakening an existing assertion, increasing a timeout, or adding `retryClick`: the human reviews the original failing artifact (JUnit XML, `build/reports/failures`, digest) first; a later green run is not evidence that the change was correct.
- Running `git push`: needs explicit approval for that specific push in the current session (`AGENTS.md`).
- Changing the pinned toolchain (`package.json`, `package-lock.json`): never run `npm install` or `npm audit fix`.

## Stop conditions

Situations where the agent must stop and report instead of continuing to push for green.

- The fix requires editing `app/`, `fake-api/` or any other path in `scripts/protected-paths.txt`: stop and report the suspected product bug instead of patching around it.
- A key claim cannot be tied to code, a JUnit XML file or a quoted log line: stop and mark it `UNVERIFIED` instead of stating it as fact.
- A check turns green only after a retry, a timeout increase, a weakened assertion, a narrowed `--tests` filter or reduced coverage: stop and report the original failure.
- The runner output shows 0 tests, a cached `UP-TO-DATE` test task, or a missing `TEST-*.xml`: stop, the run is not valid proof.

## Safe failure modes

What failing well looks like. When in doubt, prefer a loud red over a quiet green.

- If the emulator, Appium server (`127.0.0.1:4723`) or `fake-api` (port 8080) is unavailable, report `blocked` and name the missing prerequisite (for example `No booted emulator/device found`), never `done`; a claim without the required fresh run is unverified.
- Report failed checks as they are: keep the failing test names, the assertion line and the artifact paths (`appium-tests/build/reports/failures`, `appium-tests/build/reports/digests`); never rerun until green and report only the last run.
- Report UI and API results separately and with their own counts; mark conclusions without evidence `unverified`; never present `BUILD SUCCESSFUL` or exit code 0 as proof that tests ran.
- Mark AI-authored commits with the trailer `Assisted-by: Claude Code` so later review can find machine-drafted code.
