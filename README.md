# Shared GitLab CI templates

Reusable CI/CD building blocks for the Vestio projects. The templates are
plain YAML files (not CI components), so they can be included from any project
in the `tech1212145` group.

| File | Purpose |
|------|---------|
| `templates/version.yml` | Semantic version calculation and Git tag creation |
| `templates/gradle.yml` | Gradle test + package jobs (JDK 25) |
| `templates/docker.yml` | Docker build + push to ECR |
| `templates/infra-bump.yml` | Open an MR that bumps an image tag in `aws-infra` |
| `templates/security.yml` | GitLab SAST + Secret Detection |

## Quick start

Include only the files a project needs. Always include `version.yml` when
using `docker.yml`, `gradle.yml`, or `infra-bump.yml`, because those jobs read
the `VERSION` variable produced by `version:calculate`.

```yaml
include:
  - project: "tech1212145/ci-templates"
    ref: master # prefer a tag in the long term
    file:
      - templates/version.yml
      - templates/gradle.yml
      - templates/docker.yml
      - templates/infra-bump.yml
```

A minimal Lambda-style pipeline:

```yaml
stages: [test, build, release]

workflow:
  rules:
    - if: $CI_COMMIT_TAG
      when: never
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH

test:
  extends: .gradle-test

build:
  extends: .gradle-build

docker-build-push:
  extends: .docker-build-push-ecr
  variables:
    DOCKER_IMAGE_NAME: my-service

git-tag:
  extends: .version:tag
  needs:
    - job: version:calculate
      artifacts: true
    - job: docker-build-push
```

`.version:tag` defaults to the default branch (and `VERSION_SKIP=true` disables
it), so the example above does not need extra rules.

## `version.yml`

### `version:calculate` (ready to run)

Runs in `.pre` on the default branch and exports `VERSION` and `GIT_TAG`
through a dotenv artifact.

| Variable | Default | Description |
|----------|---------|-------------|
| `VERSION_PREFIX` | `""` | Prefix for independent streams, e.g. `dev-`, `prod-` |
| `VERSION_INITIAL` | `1.0.0` | Version used when no matching tag exists |
| `VERSION_BUMP` | from commit title | `major`, `minor`, or `patch` |
| `VERSION_SKIP` | unset | `true` disables the job |

Version bump detection:

- `!feat:` / `feat!:` → major
- `feat:` → minor
- anything else → patch

Override the rules by redefining the job in the consuming repo:

```yaml
version:calculate:
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
```

> **Rules gotcha:** GitLab evaluates `rules` in order and uses the first match.
> If no rule matches, the job is **not added** to the pipeline. A rule list
> with only a conditional `when: never` entry therefore excludes the job from
> every pipeline. Always end with a positive rule, or use
> `- when: on_success`.

### `.version:tag` (template)

Creates and pushes an annotated tag. It defaults to the default branch and is
disabled by `VERSION_SKIP=true`; override `rules` to tag another branch. It has
no default `needs`, so in the `.post` stage it automatically inherits dotenv
variables from earlier jobs. Add `needs` when the tag must wait for a specific
job, and list `version:calculate` there explicitly — dotenv variables are
**not** inherited transitively through another job.

## `gradle.yml`

| Job | Description |
|-----|-------------|
| `.gradle-test` | Runs `./gradlew test` |
| `.gradle-build` | Runs `./gradlew build -x test` and stores `build/libs/` |

`GRADLE_IMAGE` defaults to `gradle:9-jdk25`. Gradle caches are stored in
`.gradle/` and reused across pipelines. `.gradle-build` has no default
`needs`, so it runs after the `test` stage and still receives `VERSION` from
`version:calculate` in `.pre`. If you add `needs` to the build job, list
`version:calculate` explicitly; on MR pipelines (where it does not exist) use
`optional: true` and the build falls back to `0.0.0-SNAPSHOT`.

## `docker.yml`

`.docker-build-push-ecr` builds and pushes `<registry>/<image>:$VERSION` and
`:latest`. It runs on the default branch only and is skipped when
`VERSION_SKIP=true`.

Required variables:

- `DOCKER_IMAGE_NAME` (job variable)
- `DOCKER_REGISTRY`, `AWS_DEFAULT_REGION` (CI/CD variables)
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (CI/CD variables)

Optional: `DOCKERFILE_PATH` (default `docker/Dockerfile`),
`DOCKER_BUILD_CONTEXT` (default `.`).

## `infra-bump.yml`

`.update-infra-image-tag` clones `INFRA_PROJECT_PATH`, updates
`INFRA_TF_VAR_KEY` in `INFRA_TARGET_FILE`, pushes a branch, and opens a merge
request. Re-running is safe: the branch is reset and an existing open MR is
reused.

Required: `INFRA_COMPONENT`, `INFRA_TF_VAR_KEY`, `ACCESS_TOKEN`.

Optional: `INFRA_PROJECT_PATH` (default `aws-infra`),
`INFRA_TARGET_BRANCH` (default `dev`),
`INFRA_TARGET_FILE` (default `envs/dev/terraform.tfvars`),
`INFRA_ENV_LABEL` (default `dev`).

## `security.yml`

Includes GitLab's SAST and Secret Detection templates. The consumer must have
`test` and `secret-detection` in its `stages`, or rely on the stages defined
in this file. Set `AST_ENABLE_MR_PIPELINES=true` to also scan merge requests.

## Licence

MIT — see [LICENCE](./LICENCE).
