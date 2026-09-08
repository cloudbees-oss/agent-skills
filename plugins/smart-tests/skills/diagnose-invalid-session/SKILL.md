---
name: diagnose-invalid-session
description: Diagnose "Invalid session" or "Invalid observation session" errors shown in the CloudBees Smart Tests web app. Identifies root causes such as test path mismatches, missing test recordings, or configuration issues, then suggests CI file fixes. Use when a user reports invalid session, invalid observation session, integration issue, unrecorded tests, or test path mismatch.
compatibility: Requires smart-tests CLI (v2.15.0+) or launchable CLI (v1) authenticated. jq required for parsing CLI output.
metadata:
  author: CloudBees
---

# Diagnose Invalid Session

## Overview

Investigate why a test session is marked "Invalid" or "Invalid observation session" in the Smart Tests web app.
The web app flags a session as invalid when it detects integration issues that prevent accurate analysis. The most
common root cause is a **test path format mismatch** between the subset request and the recorded test results.

This skill uses only the `smart-tests` (v2) or `launchable` (v1) CLI — no database or internal tools required.

### What Causes Invalid Sessions

| Condition | Description |
|-----------|-------------|
| **Test path mismatch** | The test paths sent during `subset` differ in format from those sent during `record tests` |
| **NO_TESTS_EXECUTED** | No test results were recorded for the session |
| **TOO_MANY_SUBSETTINGS** | 100+ subset requests were made for a single test session |
| **NO_COMMIT_HISTORY** | No commit history was available when recording the build (PTS v1 only) |
| **Unrecorded subset/remainder tests** | More than 5% of subset or remainder test paths were not found in recorded results |
| **Extra tests recorded** | More than 5% of recorded tests were not in any subset request (observation mode) |

## Prerequisites

The smart-tests CLI (v2) or launchable CLI (v1) must be installed and authenticated.

Install (v2):

```sh
uv tool install "smart-tests-cli>=2.15" --upgrade
```

Verify connectivity:

```sh
smart-tests verify
# or for v1:
launchable verify
```

If `verify` fails with an authentication error, the user needs to set up a token:

1. Go to the Smart Tests web app → **Settings** → **API Keys**
2. Generate a new token
3. Set it in the CLI:
   ```sh
   smart-tests config set --token <TOKEN>
   # or for v1:
   launchable config set --token <TOKEN>
   ```

`jq` must be available for parsing CLI output:

```sh
jq --version
```

## Phase 1: Gather Context

Collect the following from the user before starting diagnosis:

### Test Session ID

The numeric ID shown in the web app:
- In the URL: `https://app.smart-tests.com/.../test-sessions/6909578`
- In the session detail page header: `Test session #6909578`

### Subset ID

Found in one of these places:
- The output of the `smart-tests subset` command: `Run smart-tests inspect subset --subset-id 26876 to view full subset details`
- The web app Analyze page for the test session

### Additional Context

- CLI version: v2 (`smart-tests`) or v1 (`launchable`)?
- Test runner: Maven, Gradle, pytest, Cypress, raw/JSON, etc.
- CI system: Jenkins, GitHub Actions, CircleCI, etc.
- Screenshot of the error (if available) — look for:
  - "Invalid observation session" badge
  - "Integration issue found" warning icon
  - Analyze page showing specific issue descriptions

## Phase 2: Inspect Subset Request

Examine the subset request to check for unrecognized tests:

```sh
smart-tests inspect subset --subset-id <SUBSET_ID> --json
# or for v1:
launchable inspect subset --subset-id <SUBSET_ID> --json
```

Parse the output to check the `is_new` field:

```sh
smart-tests inspect subset --subset-id <SUBSET_ID> --json \
| jq -r '
  (.subset + .remainder)
  | group_by(.is_new)
  | map({is_new: .[0].is_new, count: length})
  | .[]
  | "is_new=\(.is_new)  count=\(.count)"'
```

### Interpreting `is_new`

- `is_new: true` means the model has never seen matching test results for this path — it is either genuinely new or the path format differs between request and report.
- **All tests `is_new: true`** → Strong signal of **test path format mismatch**. The subset request paths do not match the recorded test result paths. Proceed to Phase 3.
- **Some tests `is_new: true`** → These are likely genuinely new tests. This is normal and not the cause of the invalid session.
- **No tests `is_new: true`** → The mismatch is not in test paths. Check for other root causes (Phase 4).

Also note the test path format used in the subset request:

```sh
smart-tests inspect subset --subset-id <SUBSET_ID> --json \
| jq -r '.subset[:5] | .[].test_path'
```

## Phase 3: Compare with Recorded Paths

Retrieve the test paths that were actually recorded for this session:

```sh
smart-tests inspect tests --test-session-id <TEST_SESSION_ID>
# or for v1:
launchable inspect tests --test-session-id <TEST_SESSION_ID>
```

Compare the paths from Phase 2 (subset request) with the paths from this step (recorded results).

### Common Mismatch Patterns

| Pattern | Subset request path | Recorded path |
|---------|-------------------|---------------|
| Directory vs package | `file=src/test/java/com/example/MyTest.java` | `class=com.example.MyTest#testcase=testMethod` |
| Short vs fully-qualified class | `class=MyTest` | `class=com.example.MyTest` |
| Missing hierarchy level | `class=MyTest` | `class=MyTest#testcase=testMethod` |
| Path separator | `file=src\test\MyTest.java` | `file=src/test/MyTest.java` |
| Different test runner profile | `file=tests/test_login.py` | `class=test_login#testcase=test_auth` |

If a pattern is identified, proceed to Phase 4 for the fix.

If the user does not have a subset ID but does have the test session ID, use audit logging on a fresh subset run to capture the request paths:

```sh
smart-tests --log-level audit subset <TESTRUNNER> --build <BUILD> --session <SESSION> --dry-run <TEST_PATHS>
```

The `--dry-run` flag prevents any data from being sent to the server.

## Phase 4: Diagnose and Fix

Based on the findings from Phases 2–3, apply the appropriate fix:

### Root Cause A: Test Path Format Mismatch (Most Common)

The test paths sent during `smart-tests subset` use a different format than those recorded by `smart-tests record tests`.

**Fix:** Ensure both commands use the same test runner profile and the same path structure.

Common CI fixes:

**Wrong test directory passed to subset:**

```yaml
# Before (wrong): passing source directory
- smart-tests subset maven --build $BUILD --session $SESSION src/

# After (correct): passing test directory
- smart-tests subset maven --build $BUILD --session $SESSION src/test/
```

**Different test runner profiles:**

Both `subset` and `record tests` must use the same profile (e.g., both `maven`, both `gradle`, both `raw`).

```yaml
# Before (wrong): mismatched profiles
- smart-tests subset raw --build $BUILD --session $SESSION ...
- smart-tests record tests gradle --build $BUILD --session $SESSION ...

# After (correct): matching profiles
- smart-tests subset gradle --build $BUILD --session $SESSION ...
- smart-tests record tests gradle --build $BUILD --session $SESSION ...
```

**Custom/raw profile path structure:**

When using the `raw` profile, the `testPath` format in the JSON input must match what is recorded. Use
`smart-tests inspect subset --json` to see the expected format (`file=...#class=...#testcase=...`).

### Root Cause B: No Tests Executed (`NO_TESTS_EXECUTED`)

No test results were recorded for the session. This typically happens when `record tests` fails silently
or is skipped when tests fail.

**Diagnosis:**

```sh
smart-tests stats test-sessions --days 7
# Expected: count > 0 and averageDurationSeconds > 0
```

**Fix:** Ensure `record tests` always runs, even when tests fail:

GitHub Actions:

```yaml
- name: Record test results
  if: always()
  run: smart-tests record tests ...
```

Jenkins:

```groovy
post {
    always {
        sh 'smart-tests record tests ...'
    }
}
```

Also verify:
- The `--session` name matches between `subset` and `record tests`
- Test report files (JUnit XML, etc.) exist and are not empty when `record tests` runs

### Root Cause C: Too Many Subsettings (`TOO_MANY_SUBSETTINGS`)

100 or more subset requests were made for a single test session. This happens when `smart-tests subset`
is called in a loop (once per test file) instead of once for the entire suite.

**Fix:** Call `smart-tests subset` once per test session. For parallel test execution, use the `--bin` option:

```sh
# Split tests across 4 parallel runners
smart-tests subset maven --build $BUILD --session $SESSION --bin 1/4 src/test/
smart-tests subset maven --build $BUILD --session $SESSION --bin 2/4 src/test/
smart-tests subset maven --build $BUILD --session $SESSION --bin 3/4 src/test/
smart-tests subset maven --build $BUILD --session $SESSION --bin 4/4 src/test/
```

### Root Cause D: No Commit History (`NO_COMMIT_HISTORY`)

No commit history was available when recording the build.

**Important:** PTS v2 does not always require commit history. Check the PTS version first. If the user
is on PTS v2 and sees this error, the root cause may be something else (e.g., test path mismatch) —
investigate other causes first.

For PTS v1, this is typically caused by a shallow Git clone.

**Diagnosis:**

```sh
smart-tests --log-level audit record build --build $BUILD --source . --dry-run
```

Look for warnings about missing Git objects or shallow clones.

**Fix (PTS v1 only):**

GitHub Actions:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 50  # or 0 for full history
```

Jenkins:

```groovy
checkout([
    $class: 'GitSCM',
    extensions: [[$class: 'CloneOption', depth: 50, shallow: false]]
])
```

### Root Cause E: Extra Tests Recorded (Observation Mode)

In observation mode, tests were recorded that were not part of any subset request. This happens when
additional test suites share the same test session name.

**Fix:** Use separate `--session` names for different test suites:

```sh
# Before (wrong): same session for different suites
smart-tests subset maven --session unit-tests ...
# ... also runs integration tests in the same session

# After (correct): separate sessions
smart-tests subset maven --session unit-tests --build $BUILD ...
smart-tests record tests maven --session unit-tests --build $BUILD ...
smart-tests record tests maven --session integration-tests --build $BUILD ...
```

## Output Format

Report findings in this structure:

```
Root cause:     <Test path mismatch | NO_TESTS_EXECUTED | TOO_MANY_SUBSETTINGS | NO_COMMIT_HISTORY | Extra tests recorded>

Evidence:
- is_new ratio: <X of Y tests are is_new=true>
- Subset request path example: <path from inspect subset>
- Recorded path example:       <path from inspect tests>
- Mismatch pattern:            <description of the difference>

Suggested fix:
- <specific CI file change with before/after example>

Verification:
- After applying the fix, run a new CI build and check the test session in the web app
- The "Invalid observation session" badge should no longer appear
```

## References

- `smart-tests get docs` downloads the full product documentation to `./smart-tests-docs`
- `smart-tests <command> --help` lists all available options for any CLI command
- Web app Analyze page: click the "Analyze" link on any test session to see detailed integration issue breakdowns

## Request

$ARGUMENTS
