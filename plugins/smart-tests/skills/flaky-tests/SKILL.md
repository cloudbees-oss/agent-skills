---
name: flaky-tests
description: Investigate flaky and frequently-failing tests in a CloudBees Smart Tests workspace. Two-stage workflow. First, scope a time range to find which tests fail most or whose flaky score rose the most. Second, take a test name and pin down when it started failing (regression onset).
---

You help the user investigate failing and flaky tests in a CloudBees Smart Tests
workspace. This is a task skill: run the CLI, compute the answer, and report it.
It is built around the two questions users actually ask, in order:

1. Over a time range, which tests are failing or getting flakier, and by how much?
2. For one of those tests, when did it start failing?

For conceptual questions about the product, use the `expert` skill instead.

## One-time setup

The CLI ships in the `smart-tests-cli` Python package:

```sh
uv tool install "smart-tests-cli>=2.12" --upgrade
```

Three environment variables select the workspace and authenticate. All three
must be exported in the shell that runs the CLI:

```sh
export SMART_TESTS_ORGANIZATION=<org>
export SMART_TESTS_WORKSPACE=<workspace>
export SMART_TESTS_TOKEN=<token>   # or LAUNCHABLE_TOKEN for backward compatibility
```

Common pitfall: exporting the organization and workspace but forgetting the
token. The CLI then sends requests with no `Authorization` header and the server
replies `403 Forbidden` rather than a clear "token missing" error. Confirm the
connection first:

```sh
smart-tests verify
```

## Stage 1: scope a time range

`smart-tests view flaky-tests` returns, per ISO week, the tests scored by how
flaky they were. Each test carries both an absolute `score` (0.0-1.0, higher is
flakier) and a `weeklyScoreDelta` (the change versus the prior week). Use
`score` to answer "what fails the most" and `weeklyScoreDelta` to answer "what
is getting worse".

```sh
smart-tests view flaky-tests --weeks 4 --limit 500
```

Options:

- `--weeks N` last N ISO weeks (default 1, max 12). Easiest way to set a window.
- `--from DATE --to DATE` explicit ISO-8601 range (e.g. `2026-04-08`).
- `--year-week YYYY-Www` a single ISO week, e.g. `2026-W15`.
- `--limit N` max tests per week (default 50, max 500).
- `--test-path PATH` restrict to one test (used in Stage 2).

Response shape (`data.weeks[]`): each week has `weekDate`, `calculationStatus`
(`CALCULATED`, `NOT_READY`, or `UNEXPECTED`), `flakyTestCount`, and
`flakyTests[]`. Each entry in `flakyTests[]` has `testPath` (an array of
`{type, name}` components), `score`, and `weeklyScoreDelta`. The current,
still-open week is `NOT_READY`; read the most recent `CALCULATED` week for a
stable ranking.

Rank the biggest risers in each completed week, and emit a ready-to-use test
path for Stage 2 (the `testPath` components joined as `type=value` with `#`):

```sh
smart-tests view flaky-tests --weeks 4 --limit 500 \
| jq -r '
  .data.weeks[]
  | select(.calculationStatus == "CALCULATED")
  | "## \(.weekDate)  (flakyTestCount=\(.flakyTestCount))",
    ( .flakyTests
      | sort_by(-.weeklyScoreDelta)
      | .[]
      | "score=\(.score)\tdelta=\(.weeklyScoreDelta)\t"
        + (.testPath | map("\(.type)=\(.name)") | join("#")) )'
```

Swap `sort_by(-.weeklyScoreDelta)` for `sort_by(-.score)` to rank by absolute
flakiness instead of by rise. The third column is the exact `--test-path` string
to paste into Stage 2.

## Stage 2: find when a test started failing

Two granularities, coarse then precise. Run the coarse one first to narrow the
window, then the precise one to pin the exact build.

### 2a. Weekly trend (coarse, up to 12 weeks)

Filter `flaky-tests` to one test path over the maximum window. The week where
the score jumps from roughly zero to positive is the onset week.

```sh
smart-tests view flaky-tests \
  --test-path 'file=foo/bar_test.rb#class=BarTest#testcase=test_baz' \
  --weeks 12 \
| jq -r '.data.weeks
    | sort_by(.weekDate)
    | .[]
    | "\(.weekDate)\t\(.calculationStatus)\tscore=\((.flakyTests[0].score) // 0)\tdelta=\((.flakyTests[0].weeklyScoreDelta) // 0)"'
```

### 2b. First failing execution (precise)

`view test-results` returns individual executions, each with a `createdAt`
timestamp, a `status`, `passed`/`failed`/`skipped` counts, and a `session`
(`buildId`, `branch`). Sort ascending by `createdAt` and take the first failure
to get the exact onset, including the build and branch where it first broke.

Bracket the window around the onset week from 2a so the result fits in one page
(max 500 rows). Filter for failures locally rather than with `--status`, which
can time out server-side on large workspaces:

```sh
smart-tests view test-results \
  --test-path 'file=foo/bar_test.rb#class=BarTest#testcase=test_baz' \
  --from 2026-05-01 --to 2026-05-31 --limit 500 \
| jq -r '.data.results
    | sort_by(.createdAt)
    | map(select(.status == "FAILURE" or (.failed // 0) > 0))
    | if length == 0 then "no failures in range"
      else (.[0] | "first failure: \(.createdAt)  build=\(.session.buildId)  branch=\(.session.branch)  status=\(.status)")
      end'
```

Add `--logs` to include captured stdout/stderr from failed executions when you
need the actual error. If `metadata.hasMore` is true the window held more than
one page. Narrow `--from`/`--to` (the onset week from 2a is the natural bracket)
rather than paging with `--offset`, so the earliest failure is guaranteed to be
on the first page.

## Test path format

A test path is `key=value` components joined by `#`, e.g.
`file=<path>#class=<class>#testcase=<name>`. This is the same format the record
and subset commands use. Notes:

- A file-level prefix alone does not match; you usually need `class` and
  `testcase` to identify a single test.
- `/` and `:` inside a value are literal and must not be encoded. Only the
  characters structural to the format (`#`, `=`, `&`, `%`) are percent-encoded
  when they appear inside a value.
- The reliable way to get a correct path is to copy it from Stage 1: the
  `testPath` components joined as `type=value` with `#` (the helper in the Stage
  1 recipe does exactly this).

## Reference

- `smart-tests get docs` downloads the full documentation to `./smart-tests-docs`.
- `smart-tests get api-schema` downloads the OpenAPI schema.

## Request

$ARGUMENTS
