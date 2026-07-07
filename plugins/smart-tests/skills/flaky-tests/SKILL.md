---
name: flaky-tests
description: Investigate cause for flaky tests in a CloudBees Smart Tests workspace. Identifies the spike week from flakiness score trends, reads failure logs to classify the root cause (code regression vs environment/config change), then retrieves suspect commit hashes from builds around the spike.
compatibility: Requires smart-tests CLI authenticated. Git or GitHub CLI needed for diff inspection. jq required for parsing CLI output.
metadata:
  author: CloudBees
---

# Flaky Test Cause Investigation

## Overview

Identify what caused a test to become flaky by combining flakiness score trends, failure logs, and build commit data. The investigation follows four phases:

1. **Identify candidates** — which tests are getting flakier? (skip if test path is already known)
2. **Find the spike** — when did the score jump?
3. **Read the logs** — is this a code regression or an environment/config change?
4. **Get the commits** — which commits were introduced at the spike boundary?

## Prerequisites

The smart-tests CLI must be authenticated. Verify with:

```sh
smart-tests verify
```

`jq` must be available for parsing CLI output. Verify with:

```sh
jq --version
```

## Phase 1: Identify candidate tests

Skip this phase if the user already has a specific test path.

Rank by `weeklyScoreDelta` to find tests getting worse. Run across the whole workspace:

```sh
smart-tests view flaky-tests --weeks 4 --limit 500 \
| jq -r '
  .data.weeks[]
  | select(.calculationStatus == "CALCULATED")
  | select((.flakyTests | length) > 0)
  | "## \(.weekDate)",
    ( .flakyTests
      | sort_by(-(.weeklyScoreDelta // 0))
      | .[]
      | "delta=\(.weeklyScoreDelta // 0)  score=\(.score // 0)  \(.testPath | map("\(.type)=\(.name)") | join("#"))" )'
```

The third column is the `--test-path` value to carry into Phase 2.

## Phase 2: Find the spike week

Start with 4 weeks. Print the weekly trend for the test:

```sh
smart-tests view flaky-tests --test-path "<path>" --weeks 4 \
| jq -r '
  .data.weeks
  | map(select(.calculationStatus == "CALCULATED"))
  | map(select((.flakyTests | length) > 0))
  | sort_by(-(.flakyTests[0].weeklyScoreDelta // 0))
  | .[]
  | "\(.weekDate)  score=\((.flakyTests[0].score) // 0)  delta=\((.flakyTests[0].weeklyScoreDelta) // 0)"'
```

Response shape — `data.weeks[]` contains:

- `weekDate` — ISO week start date
- `calculationStatus` — `CALCULATED`, `NOT_READY`, or `UNEXPECTED`
- `flakyTests[].score` — absolute flakiness score (0.0–1.0)
- `flakyTests[].weeklyScoreDelta` — how much the score changed compared to the prior week (positive = getting flakier, negative = improving)

Skip the current open week (`NOT_READY`). If the 4-week window is entirely flat but the test is still flaky, the spike happened before this window — re-run with `--weeks 8`, then `--weeks 12`.

The output is sorted by `weeklyScoreDelta` descending — the top row is the spike week. If there are multiple spikes, focus on the most recent one that still has a high `score` (not yet resolved). If the score dropped to near-zero between two spikes, the earlier one is a separate resolved issue and can be ignored.

## Phase 3: Read the failure logs

Window the query to the spike week. The spike week `weekDate` is `YYYY-Www` — convert to `YYYY-MM-DD` for `--from`/`--to` (e.g. `2026-W27` → `--from 2026-06-29 --to 2026-07-05`).

Start with `--status FLAKE` — this is the direct flakiness signal (test failed and passed within the same build due to retries). Check count first:

```sh
smart-tests view test-results \
  --test-path "<path>" \
  --from <YYYY-MM-DD> --to <YYYY-MM-DD> \
  --status FLAKE --limit 500 \
| jq '.data.results | length'
```

If the count is 0, the workspace may not have retries configured — re-run with `--status FAILURE`:

```sh
smart-tests view test-results \
  --test-path "<path>" \
  --from <YYYY-MM-DD> --to <YYYY-MM-DD> \
  --status FAILURE --logs --limit 500 \
| jq -r '
  .data.results
  | sort_by(.createdAt)
  | .[]
  | "\(.createdAt)  branch=\(.session.branch)\n\(.logs // "")"'
```

Classify the root cause from the log content. Logs may be sparse — an exit code alone is sufficient:

| Log pattern | Root cause |
|---|---|
| Exit code 142 / `SIGTERM` / timeout | Environment or infrastructure |
| Exit code 1 / assertion failure (`expected X, got Y`) | Code regression |
| Import / dependency / config error | Dependency or config change |

## Phase 4: Get suspect commits

Using `session.branch` from Phase 3, list all commits on or around the spike week:

```sh
git log --all --oneline --since="<YYYY-MM-DD>" --until="<YYYY-MM-DD>"
```

For each candidate commit, inspect which files changed:

```sh
git show <commit-hash> --name-only
```

Match changed files to the root cause classification from Phase 3:

- Environment/infrastructure → look for files matching `.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`, or similar CI config paths
- Code regression → look for files matching the failing test's package path

## Output Format

Report findings in this structure:

```
Spike week:   <YYYY-Www>  score=<N>  delta=<N>
First failure: <timestamp>  branch=<branch>
Classification: <Code regression | Environment/infrastructure | Dependency/config change>
Suspect commit(s):
  <hash>  <date>  <author>  <commit message>
  ...
Reasoning: <one sentence explaining why this commit is suspect>
```

## Examples

**Input:** test path, "it started failing around 2 weeks ago"

**Output:**
```
Spike week:    2026-W27  score=0.99  delta=0.99
First failure: 2026-07-03T17:45:46Z  branch=AIENG-588-handle-multi-repo-github-app-support
Classification: Code regression
Suspect commit(s):
  2d727b9147  2026-07-03  jsekar  feature/AIENG-588: handle multi repo github app sanity validation
Reasoning: Commit modified GithubAppInstallationMapper.java and DBGithubAppInstallation.java,
           both in the intake package covered by the failing test.
```

## Common Edge Cases

- 4-week window is flat but test is still flaky → spike happened before this window, re-run with `--weeks 8`, then `--weeks 12`
- Multiple spikes in the window → focus on the most recent one; earlier spikes with near-zero scores between them are separate resolved issues
- Score fluctuates without a clear step change → compare `weeklyScoreDelta` across weeks instead of absolute `score`
- All weeks show `NOT_READY` → workspace may not have enough run data yet
- No failures found in spike week window → widen `--from`/`--to` by ±1 week

## References

- `smart-tests get docs` downloads the full documentation to `./smart-tests-docs`
- `smart-tests get api-schema` downloads the OpenAPI schema
- `smart-tests <command> --help` lists all available options for any CLI command

## Request

$ARGUMENTS
