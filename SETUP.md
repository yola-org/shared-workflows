# Setup and consumer guide

One-time configuration for `yola-org/shared-workflows` and the repositories that
consume it. See [README.md](./README.md) for the available workflows, actions
and caller examples.

## Repository layout

```text
shared-workflows/
  .github/
    actions/
      aws-auth/            # OIDC with static-key fallback
      bump-infra-tag/      # opens the aws-infra PR
      version-calculate/   # semantic version + tag name
      version-tag/         # annotated tag push
    workflows/
      reusable-docker-ecr.yml
      reusable-gradle.yml
      reusable-node-ci.yml
      reusable-terraform-apply.yml
      reusable-terraform-plan.yml
      reusable-version.yml
      validate.yml         # actionlint + YAML validation for this repo
  README.md
  SETUP.md
```

## 1. Create the repositories

Create these private repositories under `yola-org`:

```text
yola-org/shared-workflows
yola-org/aws-infra
yola-org/vestio-web
yola-org/vestio-backend
yola-org/lambda-image-optimizer
```

Push tags as well (`git push --tags`) if you want version streams to continue
from the current release instead of starting at `1.0.0`.

## 2. Allow cross-repository workflows

In `yola-org/shared-workflows` → **Settings → Actions → General → Access**,
choose **Accessible from repositories in the yola-org organization**.

Without this, a private shared repository cannot be resolved by consumers and
the calling job fails with `workflow not found`.

This is the supported path for all `yola-org` repositories. The caller uses:

```yaml
uses: yola-org/shared-workflows/.github/workflows/reusable-gradle.yml@main
```

and can use `secrets: inherit`, because the caller and called workflow are in
the same organization.

## 3. Pin the shared workflows

The examples use `@main` because no release tag is created automatically. After
the first push, create a tag and replace the references:

```bash
git tag -a v1 -m "shared-workflows v1"
git push origin v1

# Run this in shared-workflows AND in every consumer repository.
grep -rl 'yola-org/shared-workflows.*@main' .github |
  xargs sed -i 's#\(yola-org/shared-workflows/[^@]*\)@main#\1@v1#g'
```

Run the replacement in this repository as well. Otherwise `reusable-*.yml@v1`
would keep calling the composite actions from `main` and the pin would not mean
anything.

## 4. Repository variables

| Repository | Variable | Required | Purpose |
| --- | --- | --- | --- |
| all | `AWS_REGION` | no | defaults to `ca-central-1` |
| lambda-image-optimizer | `ECR_REGISTRY` | no | defaults to `<account>.dkr.ecr.<region>.amazonaws.com` |
| aws-infra | `IMAGE_OPTIMIZER_IMAGE_TAG` | prod only | `TF_VAR_image_optimizer_image_tag` when `envs/prod/terraform.tfvars` is absent. Dev always reads `envs/dev/terraform.tfvars`. |

## 5. Secrets

Preferred (OIDC):

| Repository | Secret | Purpose |
| --- | --- | --- |
| all | `AWS_ROLE_ARN` | role assumed through GitHub OIDC |
| lambda-image-optimizer | `INFRA_TOKEN` | fine-grained PAT with **Contents: write** and **Pull requests: write** on `yola-org/aws-infra` |
| all | `RELEASE_TOKEN` | optional PAT with **Contents: write**, only if branch protection blocks `GITHUB_TOKEN` tag pushes |

Fallback (static keys): set `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
instead of `AWS_ROLE_ARN`. The `aws-auth` action picks whichever is configured
and fails with a clear message when neither is present.

## 6. Environments and branch protection

* `envs/dev` → create a `dev` environment (no reviewers).
* `envs/prod` → create a `prod` environment with **required reviewers** and
  restrict deployment branches to `prod`.
* Protect `dev` and `prod` in `aws-infra` (no direct pushes, require PRs).

## 7. IAM OIDC provider and role

Create the GitHub OIDC provider once per AWS account:

```text
Provider URL: https://token.actions.githubusercontent.com
Audience:     sts.amazonaws.com
```

Trust policy (narrow the `sub` condition per repository/environment):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:yola-org/*:*"
        }
      }
    }
  ]
}
```

Tighter subjects (one per job, since plan jobs use a branch ref and apply jobs
use an environment):

* `repo:yola-org/lambda-image-optimizer:ref:refs/heads/main` — ECR push.
* `repo:yola-org/aws-infra:ref:refs/heads/dev` — dev plan.
* `repo:yola-org/aws-infra:environment:dev` — dev apply.
* `repo:yola-org/aws-infra:ref:refs/heads/prod` — prod plan.
* `repo:yola-org/aws-infra:environment:prod` — prod apply.

The role needs the permissions the CI user had: ECR push for the lambda
repository, and the full Terraform apply permissions for `aws-infra`. The OIDC
role must at least cover the resources in `envs/dev` and `envs/prod`.

## Example pipelines

### lambda-image-optimizer

* Every push/PR: Gradle test + shadow JAR.
* Push to `main`, app files changed: version calculation → Docker build/push to
  ECR (`<version>` and `latest`) → annotated tag → PR bumping
  `image_optimizer_image_tag` in `aws-infra` `dev`.
* App-change filter: `src/`, `docker/`, `build.gradle(.kts)`, `settings.gradle(.kts)`,
  `gradle.properties`, `gradlew(.bat)`, `gradle/`, `buildSrc/`.
* A docs/CI-only push does not push an image, tag, or open a PR.

### vestio-backend

* Every push/PR: Gradle test + `bootJar`.
* Testcontainers tests run on the GitHub-hosted runner's Docker daemon.

### vestio-web

* Every push/PR: `npm ci`, `oxfmt --check`, `tsc --noEmit`, `next build`, then
  Playwright against `next start` with the mock backend.
* Lint, typecheck and e2e are non-blocking.

### aws-infra

* Every push/PR: `terraform fmt -check -recursive`, `init -backend=false`,
  `validate` for `envs/*`.
* Push to `dev`: plan → apply automatically → `dev-X.Y.Z` tag when `.tf` or
  `.tfvars` files changed.
* Push to `prod`: plan → wait for `prod` environment approval → apply →
  `prod-X.Y.Z` tag when `.tf` or `.tfvars` files changed.
* Push to `main`: validation only.

## Version calculation

| Commit title | Bump |
| --- | --- |
| `!feat:`, `!feat(scope):`, `feat!:`, `feat(scope)!:` | major |
| `feat:`, `feat(scope):` | minor |
| anything else | patch |

With a prefix, only tags for that stream are considered; if none exist the
stream starts at `1.0.0` (`dev-1.0.0`, `prod-1.0.0`). Pass the `bump` input to
override the commit-based detection.

## Notes and caveats

* Tags pushed with the default `GITHUB_TOKEN` do not trigger new workflow runs.
  That is intentional.
* Merge commits (`Merge branch 'x' into 'dev'`) count as patch bumps. Use
  squash/fast-forward merges if you want the PR title to drive the bump.
* `envs/prod` has no committed `terraform.tfvars`, and
  `image_optimizer_image_tag` has no default. Set the
  `IMAGE_OPTIMIZER_IMAGE_TAG` repository variable (or commit the tfvars file)
  before the first prod plan.
* `terraform apply` only runs from the `dev`/`prod` push pipelines. Pull
  requests and `main` never receive cloud credentials.
