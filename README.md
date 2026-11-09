# Shared GitHub workflows

Reusable GitHub Actions workflows and composite actions for the Vestio projects
in the `yola-org` organization. Repositories in the organization call them
directly instead of copying CI code into every project.

See [SETUP.md](./SETUP.md) for the one-time repository setup, required
secrets/variables, environments and the AWS OIDC configuration.

## Available building blocks

### Reusable workflows

| Path | Purpose |
|------|---------|
| `.github/workflows/reusable-version.yml` | Semantic version calculation (+ optional tag) |
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
| `.github/actions/aws-auth` | OIDC, with static-key fallback |
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
  id-token: write

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
      artifact-name: gradle-build-libs
    secrets: inherit
```

Rules for callers:

* `uses:` must be a job-level key; it cannot be used as a step.
* Pass secrets with `secrets: inherit` or list them explicitly under `secrets:`.
* The caller must grant the permissions the reusable workflow needs
  (`contents: write` for tagging, `id-token: write` for OIDC).
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
          role-arn: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ca-central-1
```

## Required configuration

Details are in [SETUP.md](./SETUP.md):

* `AWS_ROLE_ARN` secret for OIDC, or `AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY` as a fallback.
* `ECR_REGISTRY` repository variable when the registry is not in the caller's
  AWS account.
* `INFRA_TOKEN` secret in the application repository for `bump-infra-tag`.
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
