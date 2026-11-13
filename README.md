# reusable-workflows

Reusable GitHub Actions workflows for `brightsheets` repos. Call them from a short workflow in each repo; `/snyk-onboard` writes that caller for you.

| Workflow | What it does |
|---|---|
| [`go-ci.yaml`](.github/workflows/go-ci.yaml) | Full Go pipeline: lint → test matrix → Snyk Open Source + Code → Docker build → Snyk Container. Calls the workflows below. |
| [`go-snyk.yaml`](.github/workflows/go-snyk.yaml) | Snyk Open Source and Code scans for Go, SARIF to the Security tab, sticky PR comment, optional quality gate and Snyk monitor. |
| [`docker-build.yaml`](.github/workflows/docker-build.yaml) | Builds an image tagged `<branch>-<sha7>`, optionally pushes it (GHCR by default, never overwrites an existing tag), and uploads it as a tar artifact. |
| [`docker-snyk-scan.yaml`](.github/workflows/docker-snyk-scan.yaml) | Snyk Container scan of that tar artifact, SARIF + PR comment, optional gate and monitor. |
| [`go-lint.yaml`](.github/workflows/go-lint.yaml) | golangci-lint, report-only by default. |
| [`go-test.yaml`](.github/workflows/go-test.yaml) | `go test` for one configuration, with optional coverage artifact. |
| [`js-ci.yaml`](.github/workflows/js-ci.yaml) | JavaScript pipeline (npm or pnpm): build → Snyk Open Source + Code → Docker build → Snyk Container. |
| [`js-build.yaml`](.github/workflows/js-build.yaml) | Installs dependencies and runs a package.json script (default `build`), with an optional build artifact. |
| [`js-snyk.yaml`](.github/workflows/js-snyk.yaml) | Snyk Open Source and Code scans for a JavaScript project, SARIF + PR comment, optional gate and monitor. |

Snyk scans are **report-only** by default: findings show in the Security tab and a PR comment, and the build passes. Set `snyk-fail-on-issues: true` to make them a gate. Results are published to Snyk (`snyk monitor`) only from the default branch and `hotfix/*`.

## Use it

`.github/workflows/ci.yml` in a Go repo:

```yaml
name: CI
on:
  push:
    branches: [main, "hotfix/**"]
  pull_request:

permissions:
  contents: read
  actions: read
  security-events: write
  pull-requests: write
  packages: write

jobs:
  ci:
    uses: brightsheets/reusable-workflows/.github/workflows/go-ci.yaml@v1
    with:
      go-private-owners: brightsheets   # only if it imports private brightsheets modules
      docker-build: true                # only if the repo has a Dockerfile
    secrets:
      snyk-token: ${{ secrets.SNYK_TOKEN }}
      go-modules-token: ${{ secrets.GO_MODULES_TOKEN }}
```

A JavaScript repo with a Dockerfile:

```yaml
jobs:
  ci:
    uses: brightsheets/reusable-workflows/.github/workflows/js-ci.yaml@v1
    with:
      node-version: "18"
      docker-build: true
    secrets:
      snyk-token: ${{ secrets.SNYK_TOKEN }}
```

Just the Go Snyk scan:

```yaml
jobs:
  snyk:
    uses: brightsheets/reusable-workflows/.github/workflows/go-snyk.yaml@v1
    secrets:
      snyk-token: ${{ secrets.SNYK_TOKEN }}
```

## Secrets

| Secret | Needed for |
|---|---|
| `SNYK_TOKEN` | Every Snyk scan. Without it the scans are skipped with a warning and the rest of the pipeline still runs. A Snyk service-account token for the `brightsheets` org. With monitor on, its role also needs "View Project Ignores". |
| `GO_MODULES_TOKEN` | Repos that import private Go modules from other `brightsheets` repos. A fine-grained PAT with read access to those repos' contents. Alternatively pass `gh-app-id` and `gh-app-private-key` for a GitHub App installed on `brightsheets`. |

Pushing to GHCR uses the built-in `GITHUB_TOKEN`, so it needs no secret.

## Private Go modules in Docker builds

The build gets a git config with the token as the BuildKit secret `gitconfig`:

```dockerfile
RUN --mount=type=secret,id=gitconfig,target=/root/.gitconfig \
    GOPRIVATE=github.com/brightsheets/* go mod download
```

## Versions

Callers pin `@v1`. The `v1` tag moves to the latest compatible change; breaking changes get `v2`.

Keep `v1` a **lightweight** tag. When it pointed at an annotated tag, every caller of `go-ci.yaml@v1` failed at startup with "workflow file issue", while the same commit worked as `@main`. To move it:

```
gh api -X DELETE repos/brightsheets/reusable-workflows/git/refs/tags/v1
gh api repos/brightsheets/reusable-workflows/git/refs -f ref=refs/tags/v1 -f sha=$(git rev-parse main)
```

This repo is private. It's shared with the other `brightsheets` repos through Settings → Actions → General → Access ("Accessible from repositories owned by the user").
