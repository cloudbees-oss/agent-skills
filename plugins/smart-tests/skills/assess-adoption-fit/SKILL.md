---
name: assess-adoption-fit
description: Assess whether a target project directory is a good candidate for CloudBees Smart Tests based on CI configuration and recent CI behavior. Use when a user asks whether Smart Tests is likely to help, whether a repository or directory is suitable for Smart Tests adoption, or asks to evaluate CI duration, flaky test reruns, rerun frequency, GitHub Actions, AWS CodeBuild, Jenkins, buildspec, Jenkinsfile, or workflow evidence for Smart Tests value.
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
   - If multiple providers exist, assess each and then combine the evidence.

3. Collect recent CI history when credentials and APIs are available.
   - Do not ask the user for credentials until local detection is complete.
   - Prefer read-only CLI/API calls.
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

Use Jenkins API only when `JENKINS_URL` and credentials are already configured or the user supplies them. Otherwise, inspect local Jenkinsfiles and ask for run history or logs only if needed.

Useful API patterns:

```bash
curl "$JENKINS_URL/job/<job>/api/json?tree=builds[number,result,duration,timestamp,url,actions[parameters[name,value]]]"
curl "$JENKINS_URL/job/<job>/<build>/api/json"
```

Match Jenkins jobs to the target directory by:

- Jenkinsfile path.
- Multibranch pipeline configuration when visible.
- SCM URL and branch.
- Pipeline stages that run tests for the target directory.

Jenkins `duration` is wall-clock build duration. Use stage timing if available to distinguish test time from unrelated deploy or packaging stages.

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
