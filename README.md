# Shared GitHub workflows

Reusable GitHub Actions workflows and composite actions for the Vestio projects
in the `yola-org` organization. Repositories in the organization call them
directly instead of copying CI code into every project.

See [SETUP.md](./SETUP.md) for the one-time repository setup, required
secrets/variables, environments and the AWS credentials configuration.

## Available building blocks

### Reusable workflows

| Path | Purpose |
|------|---------|
| `.github/workflows/reusable-version.yml` | Semantic version calculation (+ optional tag) |
| `.github/workflows/reusable-changed-paths.yml` | Detect changed files matching a regex |
| `.github/workflows/reusable-release-gate.yml` | Release only for merged pull requests |
| `.github/workflows/reusable-gradle.yml` | Gradle test + package (JDK 25) |
| `.github/workflows/reusable-node-ci.yml` | Node lint/typecheck/build/e2e |
| `.github/workflows/reusable-docker-ecr.yml` | Docker build + push to ECR |
| `.github/workflows/reusable-terraform-plan.yml` | Terraform plan + saved plan artifact |
| `.github/workflows/reusable-terraform-apply.yml` | Terraform apply, gated by a GitHub environment |

Every reusable workflow declares `on: workflow_call`, so it can be invoked from
another repository with a job-level `uses:`.

### Composite actions

| Path | Purpose |
|------|---------|
| `.github/actions/version-calculate` | Version calculation used by the workflows |
| `.github/actions/version-tag` | Annotated tag creation |
| `.github/actions/changed-paths` | Detect changed files matching a regex |
| `.github/actions/aws-auth` | Configure AWS credentials from static keys |
| `.github/actions/bump-infra-tag` | Open a PR that bumps an image tag in `aws-infra` |

## Calling from another repository

A consumer in the same organization references the full path:

```text
yola-org/shared-workflows/.github/workflows/<workflow>.yml@<ref>
yola-org/shared-workflows/.github/actions/<action>@<ref>
```

`<ref>` is a branch, tag or commit SHA in **this** repository. Use `@main` while
no release tag exists; pin a tag (for example `@v1`) once one is created. A
reference to a ref or path that does not exist fails with
`workflow not found`, so double-check the repository name, file path and ref.

For private repositories, the workflow repository must allow access from the
organization. See [SETUP.md](./SETUP.md#2-allow-cross-repository-workflows).

### Reusable workflow example

```yaml
# .github/workflows/ci.yml in the consumer repository
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: write

jobs:
  version:
    if: github.ref == 'refs/heads/main'
    uses: yola-org/shared-workflows/.github/workflows/reusable-version.yml@main
    with:
      tag: false
    secrets: inherit

  test:
    uses: yola-org/shared-workflows/.github/workflows/reusable-gradle.yml@main

  docker:
    needs: [version, test]
    uses: yola-org/shared-workflows/.github/workflows/reusable-docker-ecr.yml@main
    with:
      image-name: vestio/my-service
      version: ${{ needs.version.outputs.version }}
      aws-region: ${{ vars.AWS_REGION }}
      artifact-name: gradle-build-libs
    secrets: inherit
```

### Detect changed paths

`reusable-changed-paths.yml` diffs the caller's event against its base commit
and exposes a `changed` boolean. The `patterns` input is a POSIX extended
regular expression matched against each changed path:

```yaml
jobs:
  changes:
    uses: yola-org/shared-workflows/.github/workflows/reusable-changed-paths.yml@main
    with:
      patterns: '(^|/)(src/|docker/|build\.gradle|gradlew|gradle/|Dockerfile)'

  docker:
    needs: [changes, version, test]
    if: needs.changes.outputs.changed == 'true'
    # ...
```

The same logic is available as the `.github/actions/changed-paths` composite
action when you want it inside an existing job. That action requires the
repository to be checked out with `fetch-depth: 0`.

### Release only after a merged pull request

`reusable-release-gate.yml` returns `release=true` only for a push to the
configured branch whose commit belongs to a merged pull request:

```yaml
jobs:
  release-gate:
    if: ${{ github.event_name == 'push' }}
    permissions:
      contents: read
      pull-requests: read
    uses: yola-org/shared-workflows/.github/workflows/reusable-release-gate.yml@main
    with:
      base-branch: ${{ vars.RELEASE_BRANCH || 'main' }}
      require-merged-pr: ${{ vars.REQUIRE_MERGED_PR != 'false' }}

  docker:
    needs: [release-gate, changes, version, test]
    if: >-
      always() &&
      needs.release-gate.outputs.release == 'true' &&
      needs.changes.outputs.changed == 'true'
    # ...
```

Set `REQUIRE_MERGED_PR=false` to allow direct pushes to release. The gate uses
the GitHub API, so merge commits and squash merges are detected. Rebase merges
are not reliably associated with the pull request; use merge/squash merges or
disable the gate for those repositories.

Rules for callers:

* `uses:` must be a job-level key; it cannot be used as a step.
* Pass secrets with `secrets: inherit` or list them explicitly under `secrets:`.
* The caller must grant the permissions the reusable workflow needs
  (`contents: write` for tagging).
* Reusable workflows are only resolved from the specified ref. If you call
  `@v1`, the `v1` tag must exist in this repository.

### Composite action example

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: yola-org/shared-workflows/.github/actions/aws-auth@main
        with:
          aws-region: ${{ vars.AWS_REGION }}
          access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Required configuration

Details are in [SETUP.md](./SETUP.md):

* `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` secrets for AWS access.
* `AWS_REGION` repository variable for the AWS region (not a secret). The AWS
  workflows read it automatically and fall back to `ca-central-1`.
* `ECR_REGISTRY` repository variable when the registry is not in the caller's
  AWS account.
* `INFRA_GITHUB_TOKEN` secret for release tagging and `bump-infra-tag`.
* A `prod` environment with required reviewers for production applies.

## Pinning a release

Create a tag in this repository, then replace `@main` in consumer repositories
(and in the internal action references in `.github/workflows/`):

```bash
git tag -a v1 -m "shared-workflows v1"
git push origin v1

grep -rl 'yola-org/shared-workflows.*@main' .github |
  xargs sed -i 's#\(yola-org/shared-workflows/[^@]*\)@main#\1@v1#g'
```

Run the replacement in this repository as well. Otherwise a consumer calling
`reusable-*.yml@v1` still resolves the composite actions from `main`.

## Licence

MIT — see [LICENCE](./LICENCE).
