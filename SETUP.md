# Setup and consumer guide

One-time configuration for `yola-org/shared-workflows` and the repositories that
consume it. See [README.md](./README.md) for the available workflows, actions
and caller examples.

## Repository layout

```text
shared-workflows/
  .github/
    actions/
      aws-auth/            # configures AWS credentials from static keys
      bump-infra-tag/      # opens the aws-infra PR
      changed-paths/       # detects changed files for a regex
      version-calculate/   # semantic version + tag name
      version-tag/         # annotated tag push
    workflows/
      reusable-changed-paths.yml
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
| all | `AWS_REGION` | no | repository variable (not a secret); the AWS workflows read `vars.AWS_REGION` automatically and fall back to `ca-central-1` |
| lambda-image-optimizer | `ECR_REGISTRY` | no | defaults to `<account>.dkr.ecr.<region>.amazonaws.com` |

Set `AWS_REGION` under **Settings → Secrets and variables → Actions →
Variables**, not under Secrets. The AWS workflows read it automatically and fall
back to `ca-central-1`. Callers can also pass it explicitly:

```yaml
with:
  aws-region: ${{ vars.AWS_REGION }}
```

## 5. Secrets

| Repository | Secret | Purpose |
| --- | --- | --- |
| all | `AWS_ACCESS_KEY_ID` | static AWS access key ID |
| all | `AWS_SECRET_ACCESS_KEY` | static AWS secret access key |
| all | `INFRA_GITHUB_TOKEN` | PAT with **Contents: write** on the repository and **Contents: write** + **Pull requests: write** on `yola-org/aws-infra`; used for tag pushes and infra bumps. Falls back to `GITHUB_TOKEN` for tags when unset. |

All entries above must be stored under **Settings → Secrets and variables →
Actions → Secrets**. `INFRA_GITHUB_TOKEN` is read as
`secrets.INFRA_GITHUB_TOKEN`, so a repository variable with the same name will
not work.

The AWS workflows pass these keys to the `aws-auth` action, which fails with a
clear message when either one is missing. Create a dedicated IAM user for CI
with only the permissions each pipeline needs (ECR push for
`lambda-image-optimizer`, Terraform apply permissions for `aws-infra`) and store
its programmatic access keys as the two secrets above. Rotate the keys
periodically.

## 6. Environments and branch protection

* `envs/dev` → create a `dev` environment (no reviewers).
* `envs/prod` → create a `prod` environment with **required reviewers** and
  restrict deployment branches to `prod`.
* Protect `dev` and `prod` in `aws-infra` (no direct pushes, require PRs).

## 7. IAM user and access keys

Create a dedicated IAM user for CI and generate an access key pair:

```bash
aws iam create-user --user-name github-actions-ci

aws iam create-access-key --user-name github-actions-ci
# Copy AccessKeyId and SecretAccessKey from the output.
```

Attach only the policies each pipeline needs. The ECR push user needs ECR push
permissions; the Terraform user needs the permissions for the resources in
`envs/dev` and `envs/prod`. Store the key pair as the repository secrets
`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` (see section 5).

## Example pipelines

### lambda-image-optimizer

* Every push/PR: Gradle test + shadow JAR.
* Push to `main`, app files changed: version calculation → Docker build/push to
  ECR (`<version>` and `latest`) → annotated tag → PR bumping
  `image_optimizer_image_tag` in `aws-infra` `dev`.
* App-change filter (`reusable-changed-paths.yml`): `src/`, `docker/`,
  `build.gradle(.kts)`, `settings.gradle(.kts)`, `gradle.properties`,
  `gradlew(.bat)`, `gradle/`, `buildSrc/`, `Dockerfile`, `.dockerignore`.
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
  That is intentional. When `INFRA_GITHUB_TOKEN` is set, tags are pushed with
  that PAT and do trigger workflows.
* Merge commits (`Merge branch 'x' into 'dev'`) count as patch bumps. Use
  squash/fast-forward merges if you want the PR title to drive the bump.
* `terraform apply` only runs from the `dev`/`prod` push pipelines. Pull
  requests and `main` never receive cloud credentials.
