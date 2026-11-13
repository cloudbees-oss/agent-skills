---
name: review-ci-integration
description: Review a CI configuration (GitHub Actions, Jenkins, CircleCI, or Screwdriver) that integrates the Smart Tests CLI, before merging, to catch integration mistakes such as missing always-run reporting, shallow git clones, session IDs not passed between steps/jobs, unstable build names across reruns, matrix jobs sharing one session, unsafe token handling, legacy launchable/LAUNCHABLE_TOKEN usage, and other wiring errors. Use when setting up a new Smart Tests CI integration, adding Smart Tests to a new workspace/team, or reviewing a CI config change/PR that adds or modifies smart-tests CLI commands. For diagnosing a session already flagged "Invalid" in the web app, use diagnose-invalid-session instead.
compatibility: Targets smart-tests CLI v2 (binary `smart-tests`, token `SMART_TESTS_TOKEN`). Flags legacy `launchable`/`LAUNCHABLE_TOKEN` usage for migration.
metadata:
  author: CloudBees
---

# Review CI Integration

## Overview

Review a CI pipeline's Smart Tests CLI integration **before** it ships a broken test session — either while
writing a new integration, or while reviewing a CI config change/PR that touches `smart-tests`/`launchable`
commands. This skill is proactive: none of the checks below require a test session to exist yet.

Related Smart Tests skills, by point in the lifecycle:

| Skill | When | Input |
|-------|------|-------|
| `assess-adoption-fit` | Pre-sales, before adopting Smart Tests at all | Target repo, no CLI usage yet |
| **`review-ci-integration`** | Setup-time / PR review, CLI usage present or being added, no incident yet | CI config file(s) with `smart-tests`/`launchable` commands |
| `diagnose-invalid-session` | Reactive, after the web app already shows "Invalid session" | Test session ID |
| `flaky-tests` | After data is flowing, to investigate why tests became flaky | Workspace, optionally a test path |

For general product or CLI questions that come up during the review, use `expert`.

If the user already has a specific test session showing "Invalid session" or "Invalid observation session" in
the web app, use `diagnose-invalid-session` instead — it owns the full root-cause diagnosis workflow for that
case. This skill restates its two highest-frequency fixes briefly (always-run reporting, shallow clone) with a
link, but does not re-derive that root-cause taxonomy.

### What this skill checks

| Check | Risk if missed |
|-------|-----------------|
| Legacy `launchable`/`LAUNCHABLE_TOKEN` usage | v1 options that no longer exist in v2, confusing errors |
| CLI installed in the build image | Command not found, or Python too old for v2 |
| Token handling | Leaked API key, or no data from fork PRs |
| Session ID passed to every later command | `subset`/`record tests` fail, or results land in the wrong session |
| Reporting runs even on test failure | `NO_TESTS_EXECUTED`, no data from the runs that matter most |
| Full git history available (not shallow) | `NO_COMMIT_HISTORY`, random-looking test selection |
| Stable build name, new session per CI rerun | Inflated durations, broken flaky-score calculation |
| Matrix jobs each record their own session | Parallel jobs collide on one session |
| Split build/test orchestrators share the build name and session | Test results never link back to the right build |
| Multi-repo builds label each `--source` | Commits from some repos never collected |
| `--test-suite` set on `record session` | `record session` fails (required in v2) |
| `--observation` placed on `record session`, not `subset` | `subset` fails in v2; confusion about whether tests are being reduced |

## Prerequisites

The smart-tests CLI (v2) or launchable CLI (v1) is useful for `--help`/`--dry-run` checks but not required —
most of this review is static inspection of CI config files.

```sh
uv tool install "smart-tests-cli>=2.17" --upgrade
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

Phase 3 has syntax for GitHub Actions, Jenkins, CircleCI, and Screwdriver.cd only. For AWS CodeBuild, GitLab CI,
or anything else, still run the Phase 2 checklist, but look up the CI system's own syntax for always-run steps,
clone depth, and passing files between jobs rather than guessing.

### Step 1b: Inventory existing Smart Tests CLI usage

Grep the detected CI file(s) for the CLI binary, then read each match in context:

```sh
rg -n '\b(smart-tests|launchable)\b' <ci-files>
```

Commands can have global options before the subcommand (`smart-tests --log-level audit record build ...`) or
span several lines with `\`, so don't rely on a stricter pattern.

For each pipeline/job found, note: which of `record build` / `record session` / `subset` / `record tests`
appear, whether the binary is `smart-tests` or the legacy `launchable`, and whether this is a brand-new
integration (no commands yet) or an existing one being modified.

### Step 1c: Scope to a diff, if reviewing a PR

```sh
git diff <base>...<head> -- .github/workflows Jenkinsfile .jenkins .circleci/config.yml \
  screwdriver.yaml .screwdriver.yaml buildspec.yml .gitlab-ci.yml
```

Review the full surrounding job/stage for context even when only a few lines changed — several checks below
(session handoff, matrix jobs) depend on structure outside the literal diff.

## Phase 2: Run the integration checklist

Work through each item against what Phase 1 found. Mark each **OK**, **NEEDS FIX**, or **N/A**.

### 1. Legacy binary/token naming

Flag `launchable`, `LAUNCHABLE_TOKEN`, or `pip install launchable`.

**Fix:** migrate to `smart-tests`, `SMART_TESTS_TOKEN`, and the install command in item 2. v2 still falls back
to `LAUNCHABLE_TOKEN`, but `LAUNCHABLE_ORGANIZATION`/`LAUNCHABLE_WORKSPACE` have no fallback. Many v1 options
were removed or moved in v2 (see items 4, 10, 11, 12), so a rename alone is not enough — check every option
against `smart-tests <command> --help`.

### 2. CLI installed in the build image

Confirm there's an explicit install step before any `smart-tests` command. CLI v2 requires Python 3.13 or
later.

```sh
uv tool install "smart-tests-cli~=2.17"   # preferred — uv provides its own Python, avoids PATH issues
# or, only if the image has Python >= 3.13:
pip3 install --user --upgrade smart-tests-cli
```

If the pipeline uses `pip`, check the image's Python version (`python3 --version`). Older Python images are
common in Java/Node pipelines.

### 3. Token handling

- `SMART_TESTS_TOKEN` comes from the CI system's secret store (GitHub Actions `secrets.*`, Jenkins credentials,
  CircleCI contexts/project env vars), never a literal value in the CI file.
- Nothing prints it: no `echo $SMART_TESTS_TOKEN`, no `env`/`printenv` dumps, no `set -x` in a shell where it
  is expanded on the command line.
- Fork PRs on GitHub Actions: workflows triggered by `pull_request` from a fork don't receive repository
  secrets, so Smart Tests steps fail or record nothing there. Don't "fix" this by switching to
  `pull_request_target` while checking out the PR head — that runs untrusted code with the token available.
  Accept missing data from fork PRs, or skip the Smart Tests steps when the token is empty.
- Where the CI system can issue OIDC id-tokens (GitHub Actions, Jenkins with the OpenID Connect Provider
  plugin, GitLab), suggest keyless OIDC authentication instead of a long-lived API key (CLI v2.17+). On GitHub
  Actions this needs `permissions: id-token: write`, `SMART_TESTS_GITHUB_OIDC_TOKEN_AUTH=1`, and
  `eval "$(smart-tests verify --oidc)"`; see the "Authenticate a pipeline with OIDC" page in the product docs.
  This is a recommendation, not a NEEDS FIX.

### 4. Session ID passed to every later command

In v2 there is no local session state. `record session` prints the session ID
(`builds/<BUILD>/test_sessions/<ID>`) to stdout, and `subset` and `record tests` both require `--session`:

```sh
smart-tests record session --build "$BUILD" --test-suite unit > session.txt
smart-tests subset maven --session @session.txt --target 30% ... > subset.txt
smart-tests record tests maven --session @session.txt ...
```

Confirm:

- `record session` output is captured (e.g. `> session.txt`), not discarded.
- Every `subset` and `record tests` passes `--session @session.txt` (or the captured value).
- `session.txt` actually exists where those commands run. When they run in a different job, stage, or machine,
  the file must be handed over explicitly: GitHub Actions `actions/upload-artifact`/`download-artifact` or a job
  output, Jenkins `stash`/`unstash`, CircleCI `persist_to_workspace`/`attach_workspace`.

**v1 only:** v1 saved the build and session in local files, so commands could omit `--build`/`--session`. If a
v1 pipeline uses `--no-build`, it must be used consistently on `record session`, `subset`, and `record tests`;
otherwise the later commands fail with "No saved build name found". In v2, `--no-build` exists only on
`record session`.

`subset --from-github-actions` (GitHub App mode) identifies the session from the GitHub Actions context and
can't be combined with `--session`, so don't flag a missing `--session` on that call.

Also confirm the report path/glob passed to `record tests` matches where the test runner actually writes its
reports (e.g. Maven `target/surefire-reports`, Gradle `build/test-results`). The files don't exist at review
time, so check the configuration, not the files. If nothing matches at run time, `record tests` only prints
"Looks like tests didn't run?" and exits successfully, which is easy to miss in CI logs.

### 5. Reporting runs even on test failure

The step that uploads results (`smart-tests record tests ...`) must run even when the preceding test step
failed — otherwise the one run you most need data from never gets reported, which looks like
`NO_TESTS_EXECUTED`. See Phase 3 below for the exact syntax per CI system, and
`diagnose-invalid-session` Root Cause B for the full diagnosis if this has already caused a bad session.

### 6. Full git history available (not shallow)

`record build` collects commit history from the local clone, and a shallow clone prevents that
(`NO_COMMIT_HISTORY`). Mark this N/A when commits reach Smart Tests some other way:

- The workspace uses the GitHub App integration (`smart-tests subset --from-github-actions`).
- Another job or workflow already sends the commits, e.g. a scheduled job that keeps a full clone and runs
  `smart-tests record commit --source .`. This is also the documented alternative when a full clone in the CI
  job is too slow.

See Phase 3 below for exact syntax per CI system, and `diagnose-invalid-session` Root Cause C for the full
diagnosis.

### 7. Stable build name, new session per CI rerun

Smart Tests computes flakiness from the same test running more than once against the **same build**, and
measures duration per **session**. When CI re-runs a failed job:

- The build name must stay the same across attempts. Derive `--build` from the commit SHA or the base run ID
  (`github.run_id`, Jenkins `BUILD_TAG`), not from a fresh UUID, a timestamp, or anything that includes
  `GITHUB_RUN_ATTEMPT`. Otherwise each retry becomes a separate build and the retry never counts toward
  flakiness.
- Each attempt should run `record session` again and get a new session. Don't reuse a `session.txt` cached or
  restored from an earlier attempt: results from several attempts then pile into one session, which inflates
  its recorded duration (2–3x is typical) and skews the flaky-score calculation.

### 8. Matrix jobs each record their own session

If the workflow uses `strategy: matrix` (GitHub Actions), Jenkins parallel stages, or a CircleCI matrix
alongside Smart Tests commands, confirm each parallel job runs its own `record session`, with a `--flavor` per
matrix axis, rather than all jobs sharing one session created upstream. Without this, parallel jobs collide on
a single session.

Pass `--flavor` once per key/value pair; a comma-separated value is not split into several flavors. Pass matrix
values through `env:` rather than expanding `${{ }}` inside `run:`, which is the GitHub Actions script injection
pattern:

```yaml
# GitHub Actions matrix example
- env:
    BUILD_NAME: ${{ github.run_id }}
    MATRIX_OS: ${{ matrix.os }}
    MATRIX_PYTHON: ${{ matrix.python_version }}
  run: |
    smart-tests record session --build "$BUILD_NAME" --test-suite unit \
      --flavor "os=$MATRIX_OS" \
      --flavor "python_version=$MATRIX_PYTHON" \
      > session.txt
```

### 9. Split build/test orchestrators share the build name and session

When build and test run on physically separate CI instances/pipelines with no shared filesystem (e.g. two
separate Jenkins controllers, or a build repo and a test repo), confirm:

- Both sides use the same build name, passed explicitly (an artifact version or build number), so the test
  side's `record session --build` finds the build recorded by the other side.
- The session ID is created on the side that runs the tests, or handed over explicitly (artifact, job
  parameter). The ID is assigned by the server, so it can't be rebuilt from the build name alone.

### 10. Multi-repo builds label each `--source`

```sh
# Before (wrong): only one repo's commits collected
smart-tests record build --build $BUILD --source ./main_repo

# After (correct): every repo labeled
smart-tests record build --build $BUILD --source main=./main_repo --source lib=./lib_repo
```

For microservice/independent-build setups, `record commit` + `--no-commit-collection` + `--commit REPO=HASH`
is the alternative pattern.

### 11. `--test-suite` set on `record session`

`--test-suite` is required on `record session` in v2 (it was optional in v1), and `--session-name` was removed.
Flag `--session-name` and any `record session` call without `--test-suite`. Use one test suite name per kind of
test run (e.g. `unit`, `e2e`) and keep it stable over time.

### 12. `--observation` placement

`--observation` belongs on `record session`. In v1 it was also accepted on `subset`; v2 removed it there, so a
migrated `subset --observation` fails. This is also a common point of confusion — customers often think
observation mode reduces the tests that run. It does not: the full suite still runs, and the subset is
predicted internally but not enforced.

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

Clone depth (`shallow: false` gives a full clone; use `shallow: true, depth: 50` to limit it instead):

```groovy
checkout([
    $class: 'GitSCM',
    extensions: [[$class: 'CloneOption', shallow: false]]
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
  - session: smart-tests record session --build "$BUILD_NAME" --test-suite unit > session.txt
  - subset: smart-tests subset ... --session @session.txt
  - test: ...
  - teardown-record-tests: smart-tests record tests ... --session @session.txt
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
why. Flag any fix that could affect historical trend continuity (e.g. renaming a build name scheme or
`--test-suite` breaks continuity with prior data) as a soft risk rather than applying it silently — confirm
with the user first.

### Step 3: Verification

Tell the user to run a new CI build after merging and check the session in the web app. If it still shows
"Invalid session" or "Invalid observation session", use `diagnose-invalid-session` next.

## Examples

**Input:** a GitHub Actions workflow where a `setup` job runs
`smart-tests record session --build "$BUILD_NAME" --test-suite unit > session.txt` and uploads `session.txt`,
then a `test` job with `strategy: matrix: os: [ubuntu-latest, windows-latest]` downloads it and runs
`smart-tests subset` and `smart-tests record tests` with `--session @session.txt`.

**Output:**
```
CI system(s) detected: GitHub Actions
Smart Tests commands found: record session, subset, record tests

Checklist:
NEEDS FIX  Matrix jobs each record their own session — both matrix jobs use the one session created in
           the setup job, so the ubuntu and windows runs collide on one session.
...

Suggested fixes:
- Move `record session` into the matrix job and add a flavor per matrix axis:
    + env:
    +   BUILD_NAME: ${{ github.run_id }}
    +   MATRIX_OS: ${{ matrix.os }}
    + run: |
    +   smart-tests record session --build "$BUILD_NAME" --test-suite unit \
    +     --flavor "os=$MATRIX_OS" > session.txt
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
- "Migration from CLI v1 to v2" page in the product docs — renamed and removed options per command
- `expert` — product and CLI questions beyond this checklist; it sets up the CLI and product docs for lookup
- `diagnose-invalid-session` — reactive diagnosis once a session already shows invalid in the web app
- `assess-adoption-fit` — pre-adoption fit assessment
- [CircleCI configuration reference](https://circleci.com/docs/configuration-reference/) — `when` attribute, `checkout` step
- [Screwdriver.cd job configuration](https://docs.screwdriver.cd/user-guide/configuration/) — teardown steps

## Request

$ARGUMENTS
