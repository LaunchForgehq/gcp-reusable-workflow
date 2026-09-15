# GCP reusable workflows

Centralized GitHub Actions workflows for deploying containerized applications to Google Cloud.

Two reusable workflows are published here:

- `cloud-run-deploy.yml` deploys a long-running **Cloud Run service**. It provides
  application validation gates, GitHub OIDC authentication through Google Workload
  Identity Federation, immutable Docker image publishing to Artifact Registry, and
  Cloud Run deployment with separate deploy-time and runtime identities.
- `cloud-run-job.yml` creates, executes, and verifies a **Cloud Run job** - the
  finite, one-shot counterpart used for schema migrations. It fails the pipeline
  on a non-zero execution result and surfaces the execution identity and logs.

See [Cloud Run deployment documentation](docs/cloud-run-deployment.md) for inputs, caller examples, IAM requirements, secret handling, and versioning guidance.
See [Cloud Run job documentation](docs/cloud-run-jobs.md) for migration-job
inputs, failure semantics, image pinning, and caller examples.

Both workflows accept an optional `NPMRC` secret so repositories that resolve
private packages can install inside `docker build` without baking credentials
into an image layer. Callers pass it with
`secrets: NPMRC: ${{ secrets.<NAME> }}`; the Dockerfile must consume it with
`RUN --mount=type=secret,id=npmrc,dst=/root/.npmrc ...`.
