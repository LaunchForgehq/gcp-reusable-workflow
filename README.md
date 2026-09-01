# GCP reusable workflows

Centralized GitHub Actions workflows for deploying containerized applications to Google Cloud.

The reusable Cloud Run deployment workflow provides application validation gates, GitHub OIDC authentication through Google Workload Identity Federation, immutable Docker image publishing to Artifact Registry, and Cloud Run deployment with separate deploy-time and runtime identities.

See [Cloud Run deployment documentation](docs/cloud-run-deployment.md) for inputs, caller examples, IAM requirements, secret handling, and versioning guidance.
