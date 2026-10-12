---
name: review-ci-integration
description: Review a CI configuration (GitHub Actions, Jenkins, CircleCI, or Screwdriver) that integrates the Smart Tests CLI, before merging, to catch integration mistakes such as missing always-run reporting, shallow git clones, duplicate sessions from reruns, matrix jobs sharing one session, legacy launchable/LAUNCHABLE_TOKEN usage, and other wiring errors. Use when setting up a new Smart Tests CI integration, adding Smart Tests to a new workspace/team, or reviewing a CI config change/PR that adds or modifies smart-tests CLI commands. For diagnosing a session already flagged "Invalid" in the web app, use diagnose-invalid-session instead.
compatibility: Targets smart-tests CLI v2 (binary `smart-tests`, token `SMART_TESTS_TOKEN`). Flags legacy `launchable`/`LAUNCHABLE_TOKEN` usage for migration.
metadata:
  author: CloudBees
---

# Review CI Integration

## Overview

Review a CI pipeline's Smart Tests CLI integration **before** it ships a broken test session — either while
writing a new integration, or while reviewing a CI config change/PR that touches `smart-tests`/`launchable`
commands. This skill is proactive: none of the checks below require a test session to exist yet.

This is one of three Smart Tests skills, each covering a different point in the lifecycle:

| Skill | When | Input |
|-------|------|-------|
| `assess-adoption-fit` | Pre-sales, before adopting Smart Tests at all | Target repo, no CLI usage yet |
| **`review-ci-integration`** | Setup-time / PR review, CLI usage present or being added, no incident yet | CI config file(s) with `smart-tests`/`launchable` commands |
| `diagnose-invalid-session` | Reactive, after the web app already shows "Invalid session" | Test session ID + subset ID |

If the user already has a specific test session showing "Invalid session" or "Invalid observation session" in
the web app, use `diagnose-invalid-session` instead — it owns the full root-cause diagnosis workflow for that
case. This skill restates its two highest-frequency fixes briefly (always-run reporting, shallow clone) with a
link, but does not re-derive that root-cause taxonomy.

### What this skill checks

| Check | Risk if missed |
|-------|-----------------|
| Legacy `launchable`/`LAUNCHABLE_TOKEN` usage | Wrong package installed, confusing errors |
| CLI installed in the build image | Command not found in Python-less containers |
| Reporting runs even on test failure | `NO_TESTS_EXECUTED`, no data from the runs that matter most |
| Full git history available (not shallow) | `NO_COMMIT_HISTORY`, random-looking test selection |
| Deterministic session/build naming across reruns | Inflated/split durations, broken flaky-score calculation |
| Matrix jobs use distinct session names | Parallel jobs collide on one session |
| Split build/test orchestrators share a stable key | Test results never link back to the right build |
| Multi-repo builds label each `--source` | Commits from some repos never collected |
| `--no-build` used consistently + report file exists | Silent session-0 creation, 404 on close |
| `--observation` placed on `record session`, not `subset` | Confusion about whether tests are being reduced |

## Prerequisites

The smart-tests CLI (v2) or launchable CLI (v1) is useful for `--help`/`--dry-run` checks but not required —
most of this review is static inspection of CI config files.

```sh
uv tool install "smart-tests-cli>=2.16" --upgrade
smart-tests verify
```

`rg` (ripgrep) or `grep` for scanning CI files:

```sh
rg --version || grep --version
```

## Phase 1: Detect CI system and locate Smart Tests usage

### Step 1a: Detect the CI system(s) present

- `.github/workflows/*.yml` / `.github/workflows/*.yaml` → GitHub Actions
- `Jenkinsfile` / `.jenkins` → Jenkins
- `.circleci/config.yml` → CircleCI
- `screwdriver.yaml` / `.screwdriver.yaml` → Screwdriver.cd
- `buildspec.yml` → AWS CodeBuild
- `.gitlab-ci.yml` → GitLab CI

A repo can have more than one (e.g. migrating from Jenkins to GitHub Actions) — review each independently.

### Step 1b: Inventory existing Smart Tests CLI usage

Grep the detected CI file(s) for the command sequence:

```sh
rg -n 'smart-tests (verify|record build|record session|subset|record tests|record commit|record attachment)|launchable (verify|record|subset)' <ci-files>
```

For each pipeline/job found, note: which of `record build` / `record session` / `subset` / `record tests`
appear, whether the binary is `smart-tests` or the legacy `launchable`, and whether this is a brand-new
integration (no commands yet) or an existing one being modified.

### Step 1c: Scope to a diff, if reviewing a PR

```sh
git diff <base>...<head> -- .github/workflows Jenkinsfile .circleci/config.yml screwdriver.yaml
```

Review the full surrounding job/stage for context even when only a few lines changed — several checks below
(session naming, matrix jobs) depend on structure outside the literal diff.

## Phase 2: Run the integration checklist

Work through each item against what Phase 1 found. Mark each **OK**, **NEEDS FIX**, or **N/A**.

### 1. Legacy binary/token naming

Flag `launchable`, `LAUNCHABLE_TOKEN`, or `pip install launchable`.

**Fix:** migrate to `smart-tests`, `SMART_TESTS_TOKEN`, `uv tool install smart-tests-cli~=2.0`.

### 2. CLI installed in the build image

If the base image is Java/Node/TypeScript-only (no Python step or Python base image), confirm there's an
explicit install step before any `smart-tests` command:

```sh
uv tool install smart-tests-cli~=2.0   # preferred — no Python pre-req, avoids PATH issues
# or: pip3 install --user --upgrade smart-tests-cli
```

### 3. Reporting runs even on test failure

The step that uploads results (`smart-tests record tests ...`) must run even when the preceding test step
failed — otherwise the one run you most need data from never gets reported, which looks like
`NO_TESTS_EXECUTED`. See Phase 3 below for the exact syntax per CI system, and
`diagnose-invalid-session` Root Cause B for the full diagnosis if this has already caused a bad session.

### 4. Full git history available (not shallow)

A shallow clone can prevent commit-history collection, which `record build` needs (PTS v1; PTS v2 does not
always require it). See Phase 3 below for exact syntax per CI system, and `diagnose-invalid-session` Root
Cause C for the full diagnosis.

### 5. Deterministic session/build naming across CI reruns

If `--session`/`--build` is derived from a value that changes on retry (a fresh UUID, a timestamp, or a
per-attempt ID), a CI auto-rerun of a flaky job creates a **new** session instead of continuing the original
one. This has been observed in production: a customer's GitHub Actions pipeline re-ran failed jobs but kept
reusing the same session ID across retries, so recorded duration came out 2–3x the real value and the
flaky-score calculation broke because the session was effectively split.

**Fix:** derive the session/build name from something stable across retries — the commit SHA or the base
run ID — not from a per-attempt value such as a value that increments on `GITHUB_RUN_ATTEMPT`.

### 6. Matrix jobs use distinct, deterministic session names

If the workflow uses `strategy: matrix` (GitHub Actions), Jenkins parallel stages, or a CircleCI matrix
executor alongside Smart Tests commands, confirm each parallel job gets its own `--session-name` (and
optionally `--flavor`) derived from the matrix context — not one shared literal string. Without this,
parallel jobs collide on a single session.

```yaml
# GitHub Actions matrix example
- run: |
    smart-tests record session --build "${{ github.run_id }}" \
      --session-name "${{ github.run_id }}-${{ matrix.os }}-${{ matrix.python_version }}" \
      --flavor "os=${{ matrix.os }},python_version=${{ matrix.python_version }}" \
      > session.txt
```

### 7. Split build/test orchestrators share a stable key

When build and test run on physically separate CI instances/pipelines with no shared filesystem (e.g. two
separate Jenkins controllers, or a build repo and a test repo), the session file created by
`record build`/`record session` on one machine isn't visible to the other. Confirm a stable shared key (an
artifact version or build number, passed explicitly to both sides) is used instead of relying on session-file
handoff.

### 8. Multi-repo builds label each `--source`

```sh
# Before (wrong): only one repo's commits collected
smart-tests record build --build $BUILD --source ./main_repo

# After (correct): every repo labeled
smart-tests record build --build $BUILD --source main=./main_repo --source lib=./lib_repo
```

For microservice/independent-build setups, `record commit` + `--no-commit-collection` + `--commit REPO=HASH`
is the alternative pattern.

### 9. `--no-build` consistency and report-file existence

If `--no-build` appears on `record session`/`subset`, confirm it also appears on `record tests` — using it
on only one side produces "No saved build name found" errors. Also confirm a step verifies the test report
file exists before `record tests` runs:

```sh
ls path/to/results.xml   # fail fast if the glob matches nothing
```

If the report-file glob matches nothing, the CLI never creates the session and `record tests` later 404s on
`.../test_sessions/0/close` — a confusing failure far from its real cause.

### 10. `--observation` placement

`--observation` belongs on `record session`, not on `subset`. This is a common point of confusion —
customers often think observation mode reduces the tests that run. It does not: the full suite still runs,
and the subset is predicted internally but not enforced.

## Phase 3: CI-specific syntax reference

Only use the subsection(s) for the CI system(s) actually detected in Phase 1.

### GitHub Actions

Always-run reporting:

```yaml
- name: Record test results
  if: always()
  run: smart-tests record tests ...
```

Full clone:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # or 50
```

### Jenkins

Always-run reporting:

```groovy
post {
    always {
        sh 'smart-tests record tests ...'
    }
}
```

Full clone:

```groovy
checkout([
    $class: 'GitSCM',
    extensions: [[$class: 'CloneOption', depth: 50, shallow: false]]
])
```

### CircleCI

Always-run reporting — the `when` attribute on a step (default is `on_success`; `always` and `on_fail` are
the other options):

```yaml
- run:
    name: Record test results
    command: smart-tests record tests ...
    when: always
```

Clone depth — CircleCI's built-in `checkout` step has no `depth`/`fetch-depth` key. Its default
`method: blobless` is a partial clone that filters blob content not attached to the current commit, but it
**does** retain the full commit graph — this is not the same failure mode as GitHub Actions' default. The
two real risks on CircleCI are:

- A custom `run: git clone --depth N ...` step replacing the built-in `checkout` step — this genuinely
  shallow-clones. Remove `--depth` or use the built-in `checkout` step instead.
- `record build`/commit collection needing blob content that blobless mode filtered out. Fix: set
  `checkout: method: full`.

### Screwdriver.cd

Always-run reporting — a step named with the `teardown-` prefix always runs at the end of a build, even if a
prior step failed or the build was aborted:

```yaml
steps:
  - subset: smart-tests subset ...
  - test: ...
  - teardown-record-tests: smart-tests record tests ...
```

Clone depth — **lower confidence than the sections above**: these environment variables are confirmed from
the `screwdriver-cd/scm-github` plugin source, not from a published docs.screwdriver.cd page, so verify
against the cluster's actual SCM plugin/version before relying on them:

```yaml
jobs:
  main:
    environment:
      GIT_SHALLOW_CLONE: false   # or GIT_SHALLOW_CLONE_DEPTH / GIT_SHALLOW_CLONE_SINCE
    steps: ...
```

## Phase 4: Report and optionally fix

### Step 1: Report findings

```
CI system(s) detected: <GitHub Actions | Jenkins | CircleCI | Screwdriver | ...>
Smart Tests commands found: <record build | record session | subset | record tests | none>

Checklist:
[OK|NEEDS FIX|N/A]  <item name> — <one-line evidence/reason>
...

Suggested fixes:
- <description, with diff if the file is accessible>
```

### Step 2: Fix CI configuration

If the CI file(s) are accessible, apply fixes directly for each "NEEDS FIX" item, show the diff, and explain
why. Flag any fix that could affect historical trend continuity (e.g. renaming a session/build key breaks
continuity with prior data) as a soft risk rather than applying it silently — confirm with the user first.

### Step 3: Verification

Tell the user to run a new CI build after merging and check the session in the web app. If it still shows
"Invalid session" or "Invalid observation session", use `diagnose-invalid-session` next.

## Examples

**Input:** a GitHub Actions workflow with `strategy: matrix: os: [ubuntu-latest, windows-latest]` and a single
shared `smart-tests record session --build "$BUILD" --session-name fixed-name`.

**Output:**
```
CI system(s) detected: GitHub Actions
Smart Tests commands found: record build, record session, subset, record tests

Checklist:
NEEDS FIX  Matrix jobs use distinct session names — both matrix jobs pass the same literal
           --session-name "fixed-name", so the ubuntu and windows runs will collide on one session.

Suggested fixes:
- Derive --session-name from matrix context:
    - --session-name fixed-name
    + --session-name "${{ github.run_id }}-${{ matrix.os }}"
```

## Common Edge Cases

- **Hidden test-order dependencies**: some test suites have undocumented dependencies between tests. Once
  subsetting is enabled in production, reordering/reducing tests can surface these as new failures even
  though observation-mode data looked fine. This is a go-live risk, not a wiring mistake — mention it as a
  heads-up, not a checklist failure, and suggest mandatory/always-run test lists for critical tests or
  starting production rollout with a single stable suite.
- **CI system not auto-detected**: ask the user directly which CI system(s) are in use rather than guessing.
- **Multiple CI systems in one repo**: review each independently; don't assume findings from one apply to
  the other.

## References

- `smart-tests get docs` downloads the full product documentation to `./smart-tests-docs`
- `smart-tests <command> --help` lists all available options for any CLI command
- `diagnose-invalid-session` — reactive diagnosis once a session already shows invalid in the web app
- `assess-adoption-fit` — pre-adoption fit assessment
- [CircleCI configuration reference](https://circleci.com/docs/configuration-reference/) — `when` attribute, `checkout` step
- [Screwdriver.cd job configuration](https://docs.screwdriver.cd/user-guide/configuration/) — teardown steps

## Request

$ARGUMENTS
