# PipelineIQ Main Helm and CI/CD Repository

This repository is the shared GitOps repository for PipelineIQ. It owns the Helm charts, ArgoCD application definitions, and the reusable GitHub Actions workflow used by every service repository except Terraform.

## Reusable Service Pipeline

Service repositories call:

```yaml
uses: PipelineIQproject/pipeline_main/.github/workflows/reusable-node-service-ci.yml@dev
```

The workflow performs the capstone CI/CD gates:

- install dependencies, lint, test, and app build when scripts exist
- SonarQube Cloud code scan
- Snyk dependency scan
- Docker multistage image build
- Trivy image scan with HIGH/CRITICAL failure gate
- Snyk container scan
- smoke test
- push image to Azure Container Registry
- update this repository's `values-dev.yaml` on `dev`
- wait for GitHub Environment approval, then update `values-prod.yaml` on `master`
- send Slack success/failure notification

ArgoCD inside AKS should watch this repository. After CI updates a values file, ArgoCD reconciles the cluster deployment.

## Required GitHub Secrets in Each Service Repository

Add these under `Settings -> Secrets and variables -> Actions -> Repository secrets`.

| Secret | Purpose |
| --- | --- |
| `ACR_LOGIN_SERVER` | Azure Container Registry login server, for example `pipelineiqacr.azurecr.io`. |
| `ACR_USERNAME` | ACR username or service principal client id with push permission. |
| `ACR_PASSWORD` | ACR password or service principal secret. |
| `SONAR_TOKEN` | SonarQube Cloud token for the service project. |
| `SNYK_TOKEN` | Snyk API token used for dependency and container security scans. |
| `MAIN_REPO_PAT` | GitHub PAT with `contents:write` on `PipelineIQproject/pipeline_main`. |
| `SLACK_WEBHOOK_URL` | Slack incoming webhook URL for pipeline notifications. |
| `SMOKE_TEST_ENV_FILE` | Optional dotenv content used only when an HTTP service needs runtime env vars for smoke testing. |

Recommended repository variables:

| Variable | Purpose |
| --- | --- |
| `SONAR_ORGANIZATION` | SonarQube Cloud organization key. |
| `SONAR_PROJECT_KEY` | SonarQube Cloud project key for that service. |

The caller workflows currently pass explicit Sonar values as inputs. Keep the values aligned with the SonarQube Cloud project names.

## Manual Production Approval

Create a GitHub Environment named `production` in every service repository:

1. Open `Settings -> Environments`.
2. Create `production`.
3. Add required reviewers.
4. Keep deployment branches restricted to `dev` if you only want dev builds to request promotion.

When the service workflow reaches `Update Prod Helm Values`, GitHub pauses the job until approval is granted. After approval, it commits the same tested image tag to `master` in `values-prod.yaml`.

## Running a Service Pipeline

Push to a service repository `dev` branch:

```bash
git checkout dev
git add .
git commit -m "your change"
git push origin dev
```

Or trigger manually from GitHub:

```text
Actions -> Service CI/CD -> Run workflow -> Branch: dev
```

The expected flow is:

```text
service dev push
-> reusable workflow in this repo
-> ACR image push
-> values-dev.yaml commit to this repo dev
-> ArgoCD deploys dev
-> production environment approval
-> values-prod.yaml commit to this repo master
-> ArgoCD deploys prod
```

## Helm Validation

Render a chart before ArgoCD syncs it:

```powershell
helm lint ./auth-service -f ./auth-service/values-dev.yaml
helm template auth-service ./auth-service -n pipelineiq -f ./auth-service/values-dev.yaml
```

Install shared resources first:

```powershell
helm upgrade --install pipelineiq-config ./config -f ./config/values-prod.yaml
helm upgrade --install pipelineiq-keyvault ./keyvault -n pipelineiq -f ./keyvault/values-prod.yaml
helm upgrade --install rabbitmq ./rabbitmq -n pipelineiq -f ./rabbitmq/values-prod.yaml
```

Install application services:

```powershell
helm upgrade --install frontend ./frontend -n pipelineiq -f ./frontend/values-prod.yaml
helm upgrade --install auth-service ./auth-service -n pipelineiq -f ./auth-service/values-prod.yaml
helm upgrade --install dashboard-api ./dashboard-api -n pipelineiq -f ./dashboard-api/values-prod.yaml
helm upgrade --install github-integration-service ./github-integration-service -n pipelineiq -f ./github-integration-service/values-prod.yaml
helm upgrade --install pipeline-runner-service ./pipeline-runner-service -n pipelineiq -f ./pipeline-runner-service/values-prod.yaml
helm upgrade --install workflow-analyzer-service ./workflow-analyzer-service -n pipelineiq -f ./workflow-analyzer-service/values-prod.yaml
helm upgrade --install gemini-ai-service ./gemini-ai-service -n pipelineiq -f ./gemini-ai-service/values-prod.yaml
helm upgrade --install webhook-service ./webhook-service -n pipelineiq -f ./webhook-service/values-prod.yaml
helm upgrade --install notification-service ./notification-service -n pipelineiq -f ./notification-service/values-prod.yaml
helm upgrade --install api-gateway ./api-gateway -n pipelineiq -f ./api-gateway/values-prod.yaml
```

## ArgoCD Verification

```bash
argocd app list
argocd app sync pipelineiq
argocd app get pipelineiq
kubectl get pods -n pipelineiq
kubectl get ingress -n pipelineiq
```

Successful deployment should show running pods, healthy ArgoCD sync status, and a service or ingress endpoint that returns HTTP 200 for health endpoints.
