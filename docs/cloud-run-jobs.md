# Cloud Run reusable job workflow

The workflow at `.github/workflows/cloud-run-job.yml` creates or updates a
Cloud Run **Job** and then executes it synchronously, failing the pipeline when
the execution fails. It is the migration counterpart to the Cloud Run service
deployment workflow and exists because a job lifecycle is not a service
lifecycle: a job is finite, has no traffic or ingress, and its exit status is a
release gate rather than a rollout.

Its execution path is:

```text
checkout -> validate -> (optional setup/lint/tests/pre-build)
         -> GitHub OIDC/WIF authentication -> (optional Docker build + Artifact Registry push)
         -> create/update job -> execute -> wait -> assert result
         -> surface execution logs -> execution summary
```

## Inputs, secrets, and outputs

| Input | Required | Default | Purpose |
| --- | --- | --- | --- |
| `environment` | Yes | — | GitHub Environment and deployment label: `dev`, `stage`, or `production` |
| `job` | Yes | — | Cloud Run Job name |
| `image_name` | No | `job` | Image name used to derive the git-SHA image reference |
| `image` | No | empty | Full image reference to run; when set, nothing is built or pushed |
| `docker_context` | No | `.` | Docker build context (build path only) |
| `dockerfile` | No | `./Dockerfile` | Dockerfile path (build path only) |
| `runtime_service_account_variable` | No | empty | GitHub Environment variable holding the job runtime service account |
| `env_vars` | No | empty | Newline-separated non-sensitive `KEY=VALUE` settings |
| `secret_refs` | No | empty | Newline-separated `KEY=SECRET:VERSION` references; `KEY` may be an absolute mount path |
| `command` | No | empty | Container entrypoint override |
| `args` | No | empty | Newline-separated argument override |
| `job_flags` | No | empty | Additional `gcloud run jobs deploy` flags |
| `timeout_minutes` | No | `20` | Workflow job timeout |
| `setup_command` / `lint_command` / `test_command` / `pre_build_command` | No | empty | Validation gates, run only on the build path |

| Secret | Required | Purpose |
| --- | --- | --- |
| `NPMRC` | No | `.npmrc` content for private registries, mounted as BuildKit secret `npmrc` |

| Output | Purpose |
| --- | --- |
| `image` | Image reference the job was executed with |
| `execution_name` | Cloud Run job execution name |

`env_vars` and `secret_refs` are sent to Cloud Run with a custom `@` delimiter,
so values containing commas survive intact. They are applied with set
semantics for the named keys; the job definition keeps any other existing
configuration.

## Failure semantics

The workflow fails when:

- required deployment configuration is missing or malformed;
- an image build or push fails;
- the job cannot be created or updated;
- `gcloud run jobs execute --wait` returns a non-zero status;
- the execution reports `failedCount != 0` or `succeededCount == 0`.

Log surfacing is best-effort and never masks the exit status: if Cloud Logging
returns nothing for the execution, the workflow emits a warning and the run
identity stays in the step summary.

## Pinning the image

By default the image is derived exactly as in the service workflow:

```text
<region>-docker.pkg.dev/<project>/<registry>/<image_name>:<github.sha>
```

That keeps one immutable release identity across a service and the job that
migrates its database. Pass `image` instead when the pipeline has already built
and pushed the digest and the job must run that exact artifact. Nothing in this
workflow ever deploys or executes `latest`.

## Caller example

```yaml
name: Migrate production

on:
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  migrate:
    uses: <OWNER>/<WORKFLOW_REPO>/.github/workflows/cloud-run-job.yml@<SHA>
    secrets:
      NPMRC: ${{ secrets.LAUNCHFORGE_NPMRC }}
    with:
      environment: production
      job: launchforge-migrate-cp
      image_name: launchforge-control-plane
      dockerfile: ./deploy/production/control-plane.Dockerfile
      runtime_service_account_variable: GCP_MIGRATION_RUNTIME_SERVICE_ACCOUNT
      env_vars: |
        NODE_ENV=production
        MIGRATION_TARGET=control-plane
      secret_refs: |
        DATABASE_URL=launchforge-cp-database-url:latest
      job_flags: >-
        --cpu=1
        --memory=1Gi
        --max-retries=0
        --task-timeout=900
```

Callers that already build the image elsewhere pass `image` and omit the build
inputs:

```yaml
    with:
      environment: production
      job: barobyte-migrate
      image: us-central1-docker.pkg.dev/<project>/<registry>/barobyte-identity:${{ github.sha }}
```

## IAM requirements

The deployment service account needs the same Artifact Registry write access as
a service deployment, plus permission to manage and execute the job
(`roles/run.developer` on the job or project is sufficient) and
`roles/iam.serviceAccountUser` on the job runtime service account.

The job runtime service account is separate from the deployer and receives only
application permissions, for example `roles/secretmanager.secretAccessor` on
the specific secrets the migration reads.

## Migrations are forward-only

This workflow executes a command; it does not decide rollback semantics. Use it
for forward-only, expand/contract migrations, and never run a down migration as
a rollback. Application rollback is a redeploy of a previous immutable image;
database rollback is a forward fix.
