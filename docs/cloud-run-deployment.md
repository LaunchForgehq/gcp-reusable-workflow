# Cloud Run reusable deployment workflow

The workflow at `.github/workflows/cloud-run-deploy.yml` builds and deploys a caller repository to Google Cloud Run. One environment-neutral workflow serves dev, stage, and production; callers provide all environment-specific values.

Its deployment path is:

```text
checkout -> setup -> lint -> tests -> pre-build validation
         -> GitHub OIDC/WIF authentication -> Docker build -> Artifact Registry push
         -> Cloud Run deployment -> deployment summary
```

All configured validation commands run before Google Cloud authentication and before an image is built. GitHub Actions stops the job on the first non-zero exit code, so a failing setup, lint, test, or pre-build command prevents image creation, push, and deployment. Commands are intentionally language-neutral and optional.

## Inputs and output

Callers pass application-specific inputs; environment-specific infrastructure
configuration is read from the selected GitHub Environment (see the next
section). This split keeps a project ID, WIF provider, or Artifact Registry
name out of application repositories.

| Input | Required | Default | Purpose |
| --- | --- | --- | --- |
| `environment` | Yes | — | GitHub Environment and deployment label: `dev`, `stage`, or `production` |
| `service` | Yes | — | Cloud Run service name; also the image name unless `image_name` or `image` is supplied |
| `image_name` | No | empty | Image name used to derive `<registry>/<image_name>:<sha>`. Defaults to `service`. Set it when several services run the same artifact with different configuration, so the image is built once instead of duplicated per service |
| `image` | No | empty | Full image reference to deploy. When set, the image is neither built nor pushed, so one built release can be deployed to several services. The reference must already exist in Artifact Registry |
| `docker_context` | No | `.` | Docker build context |
| `dockerfile` | No | `./Dockerfile` | Dockerfile path |
| `build_args` | No | empty | Newline-separated `NAME=VALUE` entries passed to `docker build --build-arg`. Use only for non-secret values that must exist at build time because the framework inlines them into the bundle (for example Next.js `NEXT_PUBLIC_*` variables) |
| `runtime_service_account_variable` | No | empty | Name of the GitHub Environment variable holding the runtime service account |
| `env_vars` | No | empty | Newline-separated non-sensitive `KEY=VALUE` settings |
| `secret_refs` | No | empty | Newline-separated Secret Manager `KEY=SECRET:VERSION` references. `KEY` may be an absolute mount path |
| `cloud_run_flags` | No | empty | Additional `gcloud run deploy` flags |
| `setup_command` | No | empty | Dependency installation or application setup |
| `lint_command` | No | empty | Lint/static validation |
| `test_command` | No | empty | Application tests |
| `pre_build_command` | No | empty | Final validation/application build before Docker build |

### GitHub Environment variables

The workflow reads these from the GitHub Environment named by `environment`.
They are non-sensitive deployment metadata, not application credentials.

| Variable | Purpose |
| --- | --- |
| `GCP_PROJECT_ID` | Target Google Cloud project ID |
| `GCP_REGION` | Artifact Registry and Cloud Run region (for example `us-central1`) |
| `GCP_ARTIFACT_REGISTRY` | Artifact Registry repository name |
| `GCP_WIF_PROVIDER` | Full Workload Identity Federation provider resource name, using the project number |
| `GCP_DEPLOY_SERVICE_ACCOUNT` | Dedicated service account impersonated for push/deploy |
| `GCP_*_RUNTIME_SERVICE_ACCOUNT` | Runtime service accounts, selected through `runtime_service_account_variable` |

### Optional secret

| Secret | Required | Purpose |
| --- | --- | --- |
| `NPMRC` | No | `.npmrc` content for private package registries. It is written to a temporary file and mounted into the image build as BuildKit secret `id: npmrc`; the Dockerfile must consume it with `RUN --mount=type=secret,id=npmrc,dst=/root/.npmrc ...`. Without it, an image whose install step resolves private packages cannot be built |

### Build-time values

Most application configuration belongs at container start: `env_vars` and
`secret_refs` are applied to the Cloud Run revision and can change between
revisions without rebuilding. A smaller class of values must be present while
the image is built, because a framework resolves and inlines them into the
produced bundle. Two inputs cover that class, and they carry different
security properties.

`build_args` is the correct input for **non-secret** build-time values. Build
arguments are recorded in image metadata, so a value passed here is visible to
anyone who can inspect the image. `NAME` must be a valid environment-variable
name; entries that do not match `NAME=VALUE` fail the build before Docker runs.

Values that must stay out of image metadata need a BuildKit secret instead, and
the Dockerfile must declare the corresponding `RUN --mount=type=secret,...`.
The existing `NPMRC` secret covers registry authentication. Any other
build-time secret must be added to this workflow deliberately, as a named
secret with a fixed mount id, so that the set of values reaching an image build
stays reviewable rather than being driven by an open-ended caller input.

The workflow exposes `service_url`. A caller can read it as `needs.deploy.outputs.service_url` when a downstream job declares `needs: deploy`.

Commands run through Bash with `-e` and `pipefail`. Do not place secrets directly in command inputs: command text and application output are visible in Actions logs.

## Stage caller

Create a small workflow in the application repository, for example `.github/workflows/deploy-stage.yml`:

```yaml
name: Deploy Stage

on:
  push:
    branches:
      - main

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    # Pin an immutable commit SHA (or a published major tag) so a caller cannot
    # be changed underneath a release.
    uses: <OWNER>/<WORKFLOW_REPO>/.github/workflows/cloud-run-deploy.yml@<SHA>
    secrets:
      # Only needed when the image install step resolves private packages.
      NPMRC: ${{ secrets.APP_NPMRC }}
    with:
      environment: stage
      service: product-os-api
      docker_context: .
      dockerfile: ./Dockerfile
      runtime_service_account_variable: GCP_API_RUNTIME_SERVICE_ACCOUNT
      setup_command: corepack enable && pnpm install --frozen-lockfile
      lint_command: pnpm lint
      test_command: pnpm test
      pre_build_command: pnpm build
      env_vars: |
        APP_ENV=stage
        LOG_LEVEL=info
      secret_refs: |
        DATABASE_URL=product-os-stage-database-url:latest
        JWT_SECRET=product-os-stage-jwt-secret:latest
      cloud_run_flags: >-
        --cpu=1
        --memory=512Mi
        --min-instances=0
        --max-instances=5
        --concurrency=80
        --timeout=60
```

The same test gate works without workflow changes for other ecosystems:

```yaml
# Node/npm
setup_command: npm ci
lint_command: npm run lint
test_command: npm test
pre_build_command: npm run build

# Java/Maven
test_command: ./mvnw clean verify

# Java/Gradle
test_command: ./gradlew clean test
pre_build_command: ./gradlew build

# Python
setup_command: pip install -r requirements.txt
lint_command: ruff check .
test_command: pytest
```

Toolchains not already available on the GitHub-hosted runner can be installed in `setup_command`, or the central workflow can later add generic, typed toolchain setup inputs without changing the deployment contract.

## Production caller

Use a separate caller with production values, while invoking the same reusable workflow:

```yaml
name: Deploy Production

on:
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    uses: <OWNER>/<WORKFLOW_REPO>/.github/workflows/cloud-run-deploy.yml@v1
    with:
      environment: production
      project_id: my-production-project
      region: us-central1
      artifact_registry: apps
      service: product-os-api
      docker_context: .
      dockerfile: ./Dockerfile
      workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}
      deploy_service_account: ${{ vars.GCP_DEPLOY_SERVICE_ACCOUNT }}
      runtime_service_account: product-os-api-runtime@my-production-project.iam.gserviceaccount.com
      setup_command: corepack enable && pnpm install --frozen-lockfile
      lint_command: pnpm lint
      test_command: pnpm test
      pre_build_command: pnpm build
      env_vars: |
        APP_ENV=production
        LOG_LEVEL=info
      secret_refs: |
        DATABASE_URL=product-os-production-database-url:latest
        JWT_SECRET=product-os-production-jwt-secret:latest
      cloud_run_flags: >-
        --cpu=2
        --memory=1Gi
        --min-instances=1
        --max-instances=20
        --concurrency=80
        --timeout=60
```

Create a GitHub Environment named `production` in each application repository and configure its protection rules and required reviewers. Because the reusable job sets `environment` from the caller input, those protections gate the deployment job.

## Authentication and secrets

The workflow requests only `contents: read` and `id-token: write`. `google-github-actions/auth` exchanges GitHub's short-lived OIDC token through Workload Identity Federation and impersonates the dedicated deployment service account. No service-account JSON key or GitHub credential secret is accepted.

Keep configuration according to its sensitivity and purpose:

| Store | Put here | Do not put here |
| --- | --- | --- |
| GitHub variables | Project ID, WIF provider resource name, deployment service-account email, other non-sensitive deployment configuration | Application credentials |
| GitHub secrets | Caller-only credentials for separate workflows when unavoidable; this deployment workflow does not request any | GCP service-account JSON keys or normal application runtime secrets |
| GCP Secret Manager | Database credentials, API keys, JWT secrets, provider credentials | Non-sensitive deployment metadata |

`env_vars` and `secret_refs` both use merge semantics. The supplied keys are updated while unrelated existing Cloud Run environment variables and secret mappings remain intact. Secret references—not secret values—are sent to Cloud Run and are not included in the deployment summary.

## GCP identities and IAM

Use separate identities for deployment and runtime.

The WIF principal needs `roles/iam.workloadIdentityUser` on a dedicated account such as `github-cloud-run-deployer@my-project.iam.gserviceaccount.com`, restricted by the provider's attribute condition and IAM binding to the intended GitHub organization/repository/ref.

Grant the deployment service account only what the workflow needs:

- `roles/artifactregistry.writer` on the target Artifact Registry repository.
- Cloud Run deployment permissions on the target service. `roles/run.developer` is sufficient for ordinary container deployment; Google documents `roles/run.admin` when configuring Cloud Run secret mappings. Scope the role to the target service where supported, or use a custom role containing only the required `run.services` and `run.operations` permissions.
- `roles/iam.serviceAccountUser` on the selected runtime service account. This is required to attach that identity to the Cloud Run revision.

Depending on organizational policy and how Cloud Run service agents are initialized, an administrator may need to perform one-time service-agent setup. Do not compensate by granting the deployer Owner or Editor.

The runtime account, for example `product-os-api-runtime@my-project.iam.gserviceaccount.com`, receives only application permissions. Grant `roles/secretmanager.secretAccessor` on the specific referenced secrets, not project-wide when resource-level grants are practical. Add narrowly scoped Pub/Sub, Cloud Storage, database, or other roles only when the application requires them. The deployer does not need those runtime permissions. If `runtime_service_account` is omitted, the deployer still needs `iam.serviceAccounts.actAs` on whichever existing/default service identity Cloud Run uses.

## Image and release model

Every image is named:

```text
<region>-docker.pkg.dev/<project>/<registry>/<image_name>:<github.sha>
```

`image_name` defaults to `service`, so the single-service case is unchanged. Supply it explicitly when one artifact serves more than one service — for example Data Plane services split by lane and by public/internal trust, which run identical images under different configuration. Supply `image` instead to deploy an already-built reference without rebuilding, which is the pattern that keeps one release SHA pinned across several services.

The Git SHA tag provides a traceable, immutable deployment identifier; the workflow never deploys `latest`. Configure Artifact Registry tag immutability to prevent an existing SHA tag from being overwritten.

Callers should pin this reusable workflow to a stable major tag such as `@v1`, not `@main`. Publish semantic release tags such as `v1.1.0`, and move the compatible `v1` tag deliberately. Breaking changes belong in `v2`, allowing production repositories to choose when to migrate. For stricter supply-chain controls, callers may pin an immutable full commit SHA and use dependency automation to propose updates.

The current workflow keeps test, image build, push, and deploy as distinct steps. A future CI/CD split should build, test, scan, and push once, then promote the exact image digest through stage and production. Production promotion must deploy the already-tested digest and must not rebuild the source.

## Setup outside this repository

This workflow does not provision infrastructure. Before using it:

1. Create the Google Cloud projects, Artifact Registry repositories, Cloud Run prerequisites, runtime service accounts, and secrets.
2. Create and restrict a GitHub OIDC Workload Identity Provider and bind the permitted caller repository identities.
3. Create the deployment service account and apply the narrow IAM grants above.
4. Grant each runtime service account access to only its required secrets and services.
5. Add non-sensitive WIF/provider and service-account values as GitHub repository or environment variables.
6. Create GitHub Environments and configure production approvals/protection rules.
7. Publish stable semantic/major tags for this workflow repository and reference one from each caller.
8. Ensure each application repository has a Dockerfile and `.dockerignore`; exclude `gha-creds-*.json` as defense in depth.

The workflow accepts standard Google Cloud project IDs. Legacy domain-scoped project IDs contain a colon and require a different Artifact Registry image-path transformation, so they are rejected early rather than producing a malformed image reference.
