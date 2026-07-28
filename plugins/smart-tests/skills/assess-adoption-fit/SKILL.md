---
name: assess-adoption-fit
description: Assess whether a target project directory is a good candidate for CloudBees Smart Tests based on CI configuration and recent CI behavior. Use when a user asks whether Smart Tests is likely to help, whether a repository or directory is suitable for Smart Tests adoption, or asks to evaluate CI duration, flaky test reruns, rerun frequency, GitHub Actions, AWS CodeBuild, Jenkins, Screwdriver.cd, buildspec, Jenkinsfile, or workflow evidence for Smart Tests value.
---

## Overview

Evaluate Smart Tests fit from evidence, not from repository size or intuition. Focus on whether test selection can reduce real CI waiting time or reduce waste from reruns.

Default to a 30-day lookback. If there are fewer than 10 relevant CI runs, also inspect up to 90 days and clearly label the wider window.

## Assessment Workflow

1. Identify the target directory and repository root.
   - Start from the user-provided directory.
   - Find the git root when available.
   - Treat the user-provided directory as the product under assessment, especially in monorepos.
   - If CI path filters, Jenkinsfile locations, or buildspec paths limit which directories trigger CI, only count CI that applies to the target directory.

2. Detect CI providers from local files before querying remote systems.
   - GitHub Actions: `.github/workflows/*.yml` and `.github/workflows/*.yaml`.
   - AWS CodeBuild: `buildspec.yml`, `buildspec.yaml`, `.aws/codebuild`, CloudFormation/Terraform/CDK references to `AWS::CodeBuild::Project` or `aws_codebuild_project`.
   - Jenkins: `Jenkinsfile`, `jenkinsfile`, `.jenkins`, or pipeline library references.
   - Screwdriver.cd: `screwdriver.yaml`, `screwdriver.yml`, `.screwdriver.yaml`, `.screwdriver.yml`, `.sd.yaml`, `.sd.yml`, or nested `*/screwdriver.yaml` files.
   - If multiple providers exist, assess each and then combine the evidence.

3. Collect recent CI history when credentials and APIs are available.
   - Do not ask the user for credentials until local detection is complete.
   - Prefer read-only CLI/API calls.
   - When Screwdriver.cd configuration is detected, check `SD_API_BASE` and `SD_API_TOKEN` before making any Screwdriver API request. If either variable is unset or empty, pause remote history collection and first prompt the user to set the missing variable(s). Do not make unauthenticated API calls or conclude `Unknown` solely because these variables have not been configured yet.
   - For providers that create per-PR, per-event, or per-matrix job instances, inventory all applicable job identities before querying a single job's history. A static job ID from a pipeline definition is not necessarily the job ID used by historical PR or matrix runs.
   - If remote history is unavailable, use local CI definitions plus any logs or run summaries the user provided, and return `Unknown` rather than guessing.

4. Measure serial CI waiting time.
   - Use wall-clock elapsed time from CI start to final required check completion.
   - For a single workflow/build, use `completed_at - started_at`.
   - For GitHub Actions, use `startedAt` and `updatedAt`; do not use `createdAt` as the CI start time because it can include queue time.
   - For multiple required workflows triggered by the same commit, group by commit SHA or branch/event and use the gate duration from the earliest relevant start to the latest relevant completion.
   - If jobs run in parallel, do not sum all job durations as the primary metric. Use the critical path or gate wall-clock time.
   - If workflows are intentionally sequential, include the sequence in the gate duration.
   - Report job-minutes only as secondary context.

5. Check rerun and flakiness signals.
   - Count explicit reruns, repeated attempts, retry plugins, and failed-then-passed runs for the same commit or pull request.
   - Search CI definitions and logs for retry/flaky markers such as `flaky`, `rerun`, `retry`, `pytest-rerunfailures`, `surefire.rerunFailingTestsCount`, `gradle-test-retry`, `jest.retryTimes`, and test framework retry settings.
   - Treat flakiness as a strong waste signal when reruns are common and test time is meaningful.

6. Check CI frequency.
   - Count relevant CI gate runs per day over the lookback.
   - Treat 3 or more relevant runs per day as a supporting signal, not a standalone reason to recommend Smart Tests.

## Provider Data Collection

### GitHub Actions

Use `gh` when authenticated and the repository remote is GitHub.

Useful commands:

```bash
gh run list --repo <owner>/<repo> --limit 100 --json databaseId,workflowName,event,headSha,headBranch,status,conclusion,createdAt,startedAt,updatedAt
gh run view <run-id> --repo <owner>/<repo> --json databaseId,workflowName,event,headSha,headBranch,createdAt,startedAt,updatedAt,jobs
gh api repos/:owner/:repo/actions/runs --paginate
```

Derive `<owner>/<repo>` from the target repository remote and pass it explicitly. Do not rely on the shell's current working directory.

For path-scoped assessment:

- Read workflow `on.pull_request.paths`, `on.push.paths`, `paths-ignore`, job `if` conditions, and matrix filters.
- Count a workflow only if it can run for changes in the target directory.
- If path filters are absent, assume repository-wide CI applies.

GitHub rerun evidence:

- REST workflow run fields can include `run_attempt`.
- Repeated runs with the same `head_sha`, workflow, and event are rerun evidence.
- A failed run followed by a successful run for the same `head_sha` is likely rerun evidence; label it as likely if explicit attempt counts are unavailable.

### AWS CodeBuild

Use AWS CLI only when credentials are already configured or the user provides the project names. If a profile is needed and known, set it explicitly.

Useful commands:

```bash
aws codebuild list-projects
aws codebuild batch-get-projects --names <project-names>
aws codebuild list-builds-for-project --project-name <project-name>
aws codebuild batch-get-builds --ids <build-ids>
```

Match CodeBuild projects to the target directory by:

- `source.location` repository URL.
- `source.buildspec` path.
- buildspec files in or above the target directory.
- IaC definitions that name the project or buildspec.

CodeBuild duration comes from build `startTime` and `endTime`. Repeated builds for the same `sourceVersion` after failures are rerun evidence.

For projects with many builds, read build IDs newest-first, fetch details with `batch-get-builds` in API-sized batches, and stop once build `endTime` is older than the lookback window.

### Jenkins

Before making any Jenkins API request, complete this authentication gate:

1. Resolve the authentication mode from explicit user instructions or configured environment variables. A Jenkins URL alone, or a token variable name alone, is not authorization to probe the endpoint.
2. If no Jenkins credentials are supplied or configured, pause and ask the user either to provide read-only credentials or to confirm that unauthenticated access is allowed. Do not try an unauthenticated request first. If the user does not confirm either option, inspect local Jenkinsfiles and report remote history as unavailable; do not treat an anonymous 401/403 probe as a completed assessment.
3. If a token is available but the Jenkins username is missing, ask the user for the username before authenticating. Do not silently infer it from `whoami`, a local email, Git author, job owner, or prior context. If proposing a likely username, show only that non-secret value and obtain explicit confirmation before using it with the token.
4. A non-empty, explicitly configured username from either `JENKINS_USER` or `JENKINS_USERNAME`, together with `JENKINS_USER_TOKEN`, may be used without an additional prompt. Never print, echo, log, or place the token in a URL or command-line argument.

After the gate is satisfied, use read-only Jenkins API calls. Keep Basic-auth material out of process arguments where possible, for example:

```bash
jenkins_curl() {
  local jenkins_user="${JENKINS_USER:-${JENKINS_USERNAME:-}}"
  printf 'user = "%s:%s"\n' "$jenkins_user" "$JENKINS_USER_TOKEN" |
    curl -fsS --config - "$@"
}
```

Record the authentication mode and whether the user confirmed it in the evidence, but never record the token or other secret values.

Useful API patterns for authenticated mode:

```bash
jenkins_curl "$JENKINS_URL/job/<job>/api/json?tree=builds[number,result,duration,timestamp,url,actions[parameters[name,value]]]"
jenkins_curl "$JENKINS_URL/job/<job>/<build>/api/json"
```

Match Jenkins jobs to the target directory by:

- Jenkinsfile path.
- Multibranch pipeline configuration when visible.
- SCM URL and branch.
- Pipeline stages that run tests for the target directory.

Jenkins `duration` is wall-clock build duration. Use stage timing if available to distinguish test time from unrelated deploy or packaging stages.

### Screwdriver.cd

Detect Screwdriver from local pipeline files before asking for API access. Read root and nested pipeline files because monorepos can have separate pipelines for subdirectories.

Before using the Screwdriver API, verify that both required environment variables are non-empty:

```bash
# Configure these through a secret-safe mechanism; do not paste tokens into shell history.
: "${SD_API_BASE:?Set SD_API_BASE to your Screwdriver API base URL}"
: "${SD_API_TOKEN:?Set SD_API_TOKEN through a secret-safe mechanism}"
```

If `SD_API_BASE` or `SD_API_TOKEN` is unset or empty, stop and prompt the user to set it before continuing. Show the setup example, identify which variable is missing, and do not print or echo the token. Do not substitute unrelated credentials such as `LAUNCHABLE_API_KEY` for `SD_API_TOKEN`. Re-check the environment after the user sets the value(s), then continue with the read-only API calls below. Do not put a real token in an `export` command, command-line argument, or log.

Common files:

```bash
rg --files --hidden -g 'screwdriver*.yml' -g 'screwdriver*.yaml' -g '.screwdriver*.yml' -g '.screwdriver*.yaml' -g '.sd*.yml' -g '.sd*.yaml'
```

Identify relevant jobs by:

- `requires` triggers such as `~pr`, `~pr:<branch>`, `~commit`, branch regexes, and scheduled annotations such as `screwdriver.cd/buildPeriodically` or `build_periodically`.
- `sourcePaths`, nested pipeline file location, or job commands that restrict the job to the target directory.
- Test templates such as `java-gradle/test`, `nodejs/test`, or explicit test commands in `steps`.
- Environment variables such as `GRADLE_TASK`, `NPM_SCRIPT`, `SD_TEMPLATE_FULLNAME`, or other test task selectors.

Use the Screwdriver API only when a token is already configured or the user provides one. Do not print token values. A typical setup is:

```bash
: "${SD_API_BASE:?Set SD_API_BASE to your Screwdriver API base URL}"
: "${SD_API_TOKEN:?Set SD_API_TOKEN through a secret-safe mechanism}"
: "${PIPELINE_ID:?Set PIPELINE_ID for the target pipeline}"
```

Authenticate by exchanging the user or pipeline token for a JWT:

```bash
SD_JWT="$(
  printf 'header = "Authorization: Bearer %s"\n' "$SD_API_TOKEN" |
    curl -fsS --config - "$SD_API_BASE/auth/token" |
    jq -er '.token'
)"
: "${SD_JWT:?Screwdriver authentication did not return a token}"
```

The command above captures the returned `.token` without printing it. Keep the token in the current shell rather than exporting it when possible. Use this helper so bearer tokens are supplied through curl's config input instead of appearing in process arguments:

```bash
sd_curl() {
  printf 'header = "Authorization: Bearer %s"\n' "$SD_JWT" |
    curl -fsS --config - "$@"
}

sd_curl "$SD_API_BASE/openapi.json"
sd_curl "$SD_API_BASE/pipelines/$PIPELINE_ID/events?type=pr"
sd_curl "$SD_API_BASE/events/<event-id>/builds"
sd_curl "$SD_API_BASE/pipelines/$PIPELINE_ID/builds?count=50&page=1&sortBy=createTime&sort=descending&fetchSteps=true"
sd_curl "$SD_API_BASE/builds/<build-id>/steps/<step-name>/logs"
```

#### Mandatory Screwdriver PR build discovery

Apply this procedure before reporting that a PR test job has zero runs. Treat a zero result from a fixed `/jobs/{id}/builds` query as incomplete until every step below is complete.

1. Build the candidate job inventory. Start with static job IDs from the pipeline workflow graph, then add every dynamic job ID discovered from PR/event workflow graphs or build metadata. For a test job named `pr_test`, the candidate set must include job names such as `pr_test` and `PR-1234:pr_test`, not only one numeric ID.
2. Use `/pipelines/{id}/builds` as the historical PR-build inventory. Request at most 50 records per page, sort newest-first, and continue pagination until the oldest `createTime`/`endTime` is older than the lookback boundary or the API returns no more records. Ordinary `/pipelines/{id}/events` results are supplemental because PR events may be absent there.
3. Filter the inventory by `meta.build.jobName` when that field is present and retain `eventId`, `jobId`, `sha`, `createTime`, `startTime`, `endTime`, and `status`. For example, identify unit-test builds with a suffix match such as `endswith(":pr_test")`; do not filter only by the static numeric job ID. If `meta.build.jobName` is absent, resolve the `jobId` through `/jobs/{jobId}` and the workflow graph or job definitions, and do not discard the build solely because the nested field is missing. If no reliable mapping is possible, report collection as incomplete.
4. For each discovered PR `eventId`, query `/events/{event-id}/builds` and merge the results by build ID. Use the event-level response to recover parallel jobs and the event-specific dynamic job IDs. A UI URL such as `/v2/pipelines/{id}/pulls/{event-id}` is not itself an API contract; confirm the ID with `/v4/events/{event-id}` and use the documented v4 endpoints.
5. Use `/jobs/{id}/builds` only as a detail or cross-check after the candidate IDs are known. An empty response for the static pipeline job does not prove that no PR build ran.

Do not report zero PR runs for `pr_test` unless all of the following are true:

- pipeline-build pagination was exhausted: the oldest record passed the lookback boundary or the API returned no more records; record an empty response and page count when no records were returned;
- the `meta.build.jobName` PR suffix filter was applied where the field exists, and fallback `jobId`/workflow mapping was attempted for records without it;
- dynamic PR/event job IDs and event build lists were checked;
- each candidate build had a usable timestamp comparison against the lookback window; and
- no eligible build was found.

If any condition is incomplete, report `Unknown` or `collection incomplete`, identify the missing source, and do not convert an empty fixed-job response into a zero count.

Record collection coverage in the evidence: endpoints queried, page count, oldest record examined, job-name filter, static and dynamic job IDs, and raw versus eligible build counts.

For Screwdriver duration:

- For a single build, use `endTime - startTime`.
- For a PR or commit gate, group builds by `eventId` when available. Use the earliest relevant build `startTime` and latest relevant build `endTime`.
- If `eventId` is missing or not sufficient, use `sha` plus the PR number parsed from `meta.build.jobName` only to find related candidates. Preserve each build/event boundary and do not aggregate records solely on that pair; report collection as incomplete if no reliable gate grouping exists.
- For Smart Tests impact, separately compute the critical path of test builds or test steps. Identify test builds through local pipeline template names such as `java-gradle/test` or `nodejs/test`, or map numeric API `templateId` values through the workflow/job definitions before classifying them. Then compute earliest test start to latest test completion per event.
- Do not sum parallel Screwdriver job durations as the primary metric.
- If the long wall-clock time is in deploy, package, artifact upload, cache, Snyk, or teardown steps rather than test steps, report that Smart Tests may not reduce the bottleneck.

Screwdriver rerun and flakiness evidence:

- Repeated PR events for the same `sha`, or repeated builds for the same `PR-<number>:<job>` and `sha`, are candidate repeated executions; inspect the event creator, trigger, PR number, and retry/attempt metadata before labeling them as reruns.
- A repeated build for the same event/job is explicit rerun evidence only when the API exposes distinct attempts or a retry marker. Otherwise, preserve the event/job/build IDs and classify it as ordinary repeated execution or likely rerun based on the available trigger evidence.
- Keep explicit rerun, likely rerun, and ordinary repeated execution as separate labels; do not call different event IDs with the same SHA explicit reruns without supporting trigger evidence.
- A failed build followed by a successful build for the same PR/job or same `sha` is likely rerun/flaky evidence; label it as likely unless logs explicitly identify a flaky test or retry.
- Search logs and local config for retry signals, but distinguish real test retry output from unrelated release-note text containing words such as `rerun`.
- Timeout failures, for example a fixed CI timeout repeated across runs, are waste signals, but do not label them flaky without supporting evidence.

## Decision Rules

Return one of these decisions:

- `Effective`: Smart Tests is likely worth evaluating because there is evidence of meaningful CI time, rerun/flaky waste, or both.
- `Low`: Smart Tests may still work, but the available evidence suggests limited practical value.
- `Unknown`: Evidence is insufficient; list exactly what data is missing.

Use these thresholds:

- Effective: median or p75 serial CI gate time is 30 minutes or more.
- Effective: serial CI gate time is 20-30 minutes and CI runs at least 3 times per day or reruns/flakiness are common.
- Effective: serial CI gate time is 10-20 minutes and both supporting signals are present: at least 3 relevant CI runs per day and repeated rerun/flaky evidence.
- Supporting: relevant CI runs at least 3 times per day.
- Supporting: reruns or flaky tests occur repeatedly.
- Low: serial CI gate time is about 10 minutes or less.
- Low: serial CI gate time is 10-20 minutes without repeated rerun/flaky evidence or high CI frequency.
- Low: relevant CI runs less than once every 3 days, unless reruns/flakiness are severe.

Frequency alone should not produce `Effective`. A 30-minute serial CI gate can produce `Effective` even when frequency is moderate, because each affected run has meaningful savings potential.

## Output Format

Keep the final assessment concise and evidence-based:

```markdown
Decision: Effective | Low | Unknown

Why:
- Serial CI gate time: p50 X min, p75 Y min, max Z min over N runs.
- Frequency: A relevant runs/day over the last B days.
- Rerun/flaky signal: ...
- Target-directory relevance: ...

Evidence:
- Provider/workflow/build/job: ...
- Lookback window: ...
- Commands or data sources used: ...
- For Screwdriver.cd: collection coverage: endpoints, pages, oldest record, filters, candidate job IDs, and raw/eligible counts. If a count is zero, state which zero-count conditions were verified.

Reasons value may be limited:
- ...

Next checks:
- ...
```

For `Low`, always include concrete reasons. Examples:

- CI gate time is about 10 minutes, so the maximum practical time savings is small.
- CI gate time is about 40 minutes, but relevant CI runs only once every 3 days, so total savings may be low.
- The long-running CI time is deploy/package time rather than test execution, so Smart Tests may not reduce the bottleneck.
- The target directory is excluded by CI path filters, so the observed CI does not apply to it.

For `Unknown`, be explicit about the smallest useful next data request, such as GitHub Actions run history, CodeBuild project names, Jenkins job URLs, or 30 days of CI run summaries.

## Guardrails

- Do not claim savings without identifying test-related CI time.
- Do not treat total parallel job-minutes as the primary serial time metric.
- Do not count CI that cannot be triggered by the target directory.
- Do not require all three signals. CI duration is the strongest signal; frequency and flakiness refine confidence.
- Do not hide uncertainty. If provider APIs are inaccessible, say what was checked locally and what remains unknown.
