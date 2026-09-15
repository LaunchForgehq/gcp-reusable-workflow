# Cloud Run reusable job workflow

The workflow at `.github/workflows/cloud-run-job.yml` updates a Cloud Run
**Job**'s image and then executes it synchronously, failing the pipeline when
the execution fails. It is the migration counterpart to the Cloud Run service
deployment workflow and exists because a job lifecycle is not a service
lifecycle: a job is finite, has no traffic or ingress, and its exit status is a
release gate rather than a rollout.

## Authority split

This workflow owns the job **revision** and its **execution**. It owns nothing
else about the job.

| Owner | Owns |
| --- | --- |
| Infrastructure (Terraform, or whatever declares the job in the calling project) | service account, environment variables, secret bindings, container command and args, CPU/memory, retry policy, task count, parallelism |
| This workflow | which immutable image the job runs, and when it runs |

Two consequences follow, and both are deliberate:

- **The workflow will not create a job.** A job that first appears during a
  release carries an envelope no review ever saw, so an unknown job name is a
  hard failure that names the missing infrastructure.
- **The workflow sends no envelope fields.** There are no `env_vars`,
  `secret_refs`, `command`, `args`, `job_flags`, or service-account inputs.
  They existed until they were recognized as a second infrastructure
  authority: a migration caller could rewrite a reviewed envelope, and the
  rewritten configuration would silently serve until the next apply reverted
  it. An image update uses `gcloud run jobs update`, a read-modify-write of the
  existing spec, so every field the command does not name is preserved.

Its execution path is:

```text
checkout -> validate -> (optional setup/lint/tests/pre-build)
         -> GitHub OIDC/WIF authentication -> (optional Docker build + Artifact Registry push)
         -> verify job exists -> update job image -> execute -> wait -> assert result
         -> surface execution logs -> execution summary
```

## Inputs, secrets, and outputs

| Input | Required | Default | Purpose |
| --- | --- | --- | --- |
| `environment` | Yes | — | GitHub Environment and deployment label: `dev`, `stage`, or `production` |
| `job` | Yes | — | Cloud Run Job name |
| `image_name` | No | `job` | Image name used to derive the git-SHA image reference |
| `image` | No | empty | Full image reference to run; when set, nothing is built or pushed. Must end in a 40-character git-SHA tag |
| `docker_context` | No | `.` | Docker build context (build path only) |
| `dockerfile` | No | `./Dockerfile` | Dockerfile path (build path only) |
| `build_args` | No | empty | Newline-separated `NAME=VALUE` entries passed to `docker build --build-arg`. Configures the artifact, which the release owns; never the Cloud Run envelope, which infrastructure owns |
| `timeout_minutes` | No | `20` | Workflow job timeout |
| `setup_command` / `lint_command` / `test_command` / `pre_build_command` | No | empty | Validation gates, run only on the build path |

| Secret | Required | Purpose |
| --- | --- | --- |
| `NPMRC` | No | `.npmrc` content for private registries, mounted as BuildKit secret `npmrc` |

| Output | Purpose |
| --- | --- |
| `image` | Image reference the job was executed with |
| `execution_name` | Cloud Run job execution name |

## Failure semantics

The workflow fails when:

- required deployment configuration is missing or malformed;
- an explicitly supplied image is not pinned to a 40-character git-SHA tag;
- an image build or push fails;
- the job does not exist, because the calling infrastructure has not been applied;
- the job image cannot be updated;
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
```

The runtime service account, environment variables, secret bindings, command,
args, CPU, memory, retry policy, and task count are **not** supplied here. They
belong to the infrastructure that declares the job. A caller that needs a new
environment variable adds it to the job definition, not to this workflow.

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
