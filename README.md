# GCP Cloud Run Deployment

Reusable GitHub Actions for building container images and deploying Google Cloud Run services, Cloud Run Jobs, and 2nd-generation Cloud Functions. The repository also provisions optional Cloud Scheduler, Cloud Tasks, and Eventarc resources.

## Authentication

GitHub Actions authenticate to Google Cloud **only with a service account JSON key**.

- Create a dedicated Google Cloud service account for deployments.
- Grant it the minimum roles required by the resources being deployed, including Artifact Registry, Cloud Run, Cloud Functions, Eventarc, Cloud Scheduler, Cloud Tasks, and Secret Manager as applicable.
- Add the complete JSON key content as the GitHub Actions secret `GCP_SA_KEY`.
- `GCP_SA_KEY` is required by every reusable deployment workflow and is passed to `google-github-actions/auth@v2` as `credentials_json`.
- Workload Identity Federation/OIDC is intentionally not supported by the shared authentication action or reusable workflows.

The `service_account` inputs used by Cloud Run, Cloud Functions, Eventarc, and Cloud Scheduler configure identities **inside Google Cloud**. They are not GitHub authentication credentials.

Optional GitHub App secrets are needed only when a workflow checks out a private external repository:

| Secret | Purpose |
| --- | --- |
| `APP_ID` | GitHub App ID |
| `APP_PRIVATE_KEY` | GitHub App private key |

Keep application credentials and API keys in Secret Manager whenever possible. Workflows pass them through the `secrets` input as Secret Manager references; do not commit key material to this repository.

## Reusable Workflows

Reusable workflows are called with `uses: ./.github/workflows/<file>` and receive `GCP_SA_KEY` under `secrets`.

| Workflow | Deploys | Important inputs |
| --- | --- | --- |
| `wf-cloud-run-service.yml` | Cloud Run service | `project_id`, `service_name`, `region`, image/build settings, runtime `service_account`, optional Scheduler |
| `wf-cloud-run-job.yml` | Cloud Run Job | `project_id`, `job_name`, `region`, image/build settings, command/args, runtime `service_account`, optional Tasks/Scheduler |
| `wf-cloud-function.yml` | Cloud Function Gen 2 | `project_id`, `function_name`, `runtime`, `source_dir`, optional Eventarc |

All three workflows can check out an external private repository using `checkout_repository`, `checkout_ref`, `APP_ID`, and `APP_PRIVATE_KEY`. They authenticate to Google Cloud before building or deploying.

## Deployment Workflows

These manually dispatched workflows are ready-to-use examples for the project `phonic-altar-450817-q4`:

| Workflow | Deployment |
| --- | --- |
| `deploy-match-data-ingestion.yml` | Match ingestion and updater services and Cloud Run Jobs, with Cloud Scheduler |
| `deploy-post-match-events.yml` | `post-match-events` Cloud Run Job with Cloud Tasks |
| `deploy-query-injector.yml` | `query-injector` Cloud Run service |
| `deploy-score-computation.yml` | Score computation Cloud Run Jobs and Eventarc-triggered functions |
| `deploy-venues-searchpos.yml` | `venues-searchpos` Cloud Run service |
| `cd-cloud-run-service.yaml` | Direct Cloud Run deployment of an existing image |
| `cicd-cloud-run-service.yml` | Build, push, and deploy a service from an external GitHub repository |
| `wf-cloud-run-service.yml` | Reusable Cloud Run service workflow |
| `wf-cloud-run-job.yml` | Reusable Cloud Run Job workflow |
| `wf-cloud-function.yml` | Reusable Cloud Function workflow |

Run a manual deployment from the GitHub Actions tab. The deployment workflows expose `ref` and `skip_build` where applicable; the ingestion and score-computation workflows also expose a component `target`.

## Composite Actions

| Action | Responsibility |
| --- | --- |
| `gcp-auth` | Authenticates with `GCP_SA_KEY`, sets the project, and configures Docker for Artifact Registry |
| `build-image` | Builds and pushes a Docker image with Buildx and GitHub Actions cache |
| `deploy-cloud-run-service` | Deploys a Cloud Run service with resources, environment variables, and Secret Manager references |
| `deploy-cloud-run-job` | Creates or updates a Cloud Run Job with command, args, resources, and secrets |
| `deploy-cloud-function` | Deploys a 2nd-generation Cloud Function with HTTP or Pub/Sub trigger settings |
| `setup-cloud-scheduler` | Creates or updates an HTTP Scheduler job, optionally using an OAuth service account |
| `setup-cloud-tasks` | Creates a Cloud Tasks queue if it does not exist |
| `setup-eventarc-trigger` | Creates or updates an Eventarc trigger with filters and an invocation service account |

Composite actions expect an authenticated `gcloud` environment unless they are invoked through one of the reusable workflows. `gcp-auth` must run before any action that calls `gcloud`, Docker Artifact Registry, or deployment commands.

## Example Caller

```yaml
jobs:
	deploy:
		uses: ./.github/workflows/wf-cloud-run-service.yml
		with:
			project_id: phonic-altar-450817-q4
			service_name: example-service
			region: europe-southwest1
			checkout_repository: supporterApp/example-service
			service_account: 343004725643-compute@developer.gserviceaccount.com
		secrets:
			GCP_SA_KEY: ${{ secrets.GCP_SA_KEY }}
			APP_ID: ${{ secrets.APP_ID }}
			APP_PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }}
```

The service account in `with.service_account` is the identity assigned to the deployed Cloud Run revision. The `GCP_SA_KEY` secret is the separate deployment credential used by GitHub Actions.

## Repository Layout

```text
.github/
	actions/      Reusable composite actions
	workflows/    Manual deployment and reusable workflows
scripts/        Service-specific deployment assets
sa_gitaction2gcp.json  Local/service-account material; keep sensitive keys out of commits
```

Never commit a real service-account key. Rotate `GCP_SA_KEY` if it is exposed and remove the exposed key from Google Cloud immediately.
