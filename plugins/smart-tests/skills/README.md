# CloudBees Smart Tests Agent Skills

See [top level README](../../README.md) for installation.

## Skills

- `expert`: Answer CloudBees Smart Tests product questions using the CLI and product documentation.
- `assess-adoption-fit`: Assess whether a target project directory is likely to benefit from Smart Tests by checking CI configuration, serial CI duration, reruns/flakiness, and CI frequency.
- `review-ci-integration`: Review a CI pipeline's Smart Tests CLI integration (GitHub Actions, Jenkins, CircleCI, Screwdriver) before merging, to catch mistakes like missing always-run reporting, shallow clones, and duplicate sessions from reruns.

## Usage

Ask your agent to use the relevant skill by name.

For Smart Tests product questions:

```text
Use $expert to explain how CloudBees Smart Tests works with GitHub Actions.
```

For adoption-fit checks:

```text
Use $assess-adoption-fit to assess whether ./services/api is likely to benefit from CloudBees Smart Tests.
```

The adoption-fit check first inspects local CI configuration in the target directory and repository. When credentials are already available, it can also inspect recent CI history from GitHub Actions, AWS CodeBuild, or Jenkins. The result is one of:

- `Effective`: there is enough evidence that Smart Tests is likely worth evaluating.
- `Low`: Smart Tests may still work, but the available evidence suggests limited practical value.
- `Unknown`: the agent could not collect enough evidence to decide.

For reviewing a CI integration before merging:

```text
Use $review-ci-integration to review the Smart Tests setup in .github/workflows/ci.yml before I merge it.
```
