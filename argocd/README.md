# PipelineIQ ArgoCD App of Apps

Apply only the root Application after ArgoCD and Argo Rollouts are installed:

```bash
kubectl apply -f argocd/application.yaml
```

The root app syncs every child `application.yaml` under this folder and ignores nested Helm chart files. Each child service creates two ArgoCD Applications:

- `<service>-dev` deploys from branch `dev` into namespace `pipelineiq-dev` with `values-dev.yaml`.
- `<service>-prod` deploys from branch `main` into namespace `pipelineiq-prod` with `values-prod.yaml`.

Config and RabbitMQ use sync wave `0`. Application services use sync wave `1`.

## Blue-Green Rollouts

Every application service uses an Argo Rollouts `Rollout` with:

- active service: `<service>`
- preview service: `<service>-preview`
- dev auto promotion: enabled
- prod auto promotion: disabled

To test a rollout, change a service image tag in the relevant values file, commit, and push.

For dev:

```bash
argocd app sync frontend-dev
kubectl argo rollouts get rollout frontend -n pipelineiq-dev
```

For prod after the new ReplicaSet is ready:

```bash
kubectl argo rollouts promote frontend -n pipelineiq-prod
kubectl argo rollouts get rollout frontend -n pipelineiq-prod
```

## Key Vault Environment

The config chart creates only the Azure Key Vault CSI `SecretProviderClass`. It does not create a Kubernetes `ConfigMap` or `Secret`.

Backend pods mount Key Vault at `/mnt/secrets-store`. Their startup command exports each mounted file as an environment variable by converting the file name from kebab-case to upper snake case. For example:

```text
database-url -> DATABASE_URL
entra-client-id -> ENTRA_CLIENT_ID
```

Create these Key Vault secrets:

```text
node-env
frontend-url
public-api-base-url
public-auth-base-url
auth-service-url
dashboard-api-url
github-service-url
pipeline-runner-url
analyzer-service-url
gemini-service-url
notification-service-url
github-callback-url
auth-provider
entra-tenant-id
entra-client-id
entra-callback-url
session-ttl
ai-provider
ai-fallback-enabled
ai-fallback-provider
gemini-model
azure-openai-endpoint
azure-openai-deployment
azure-openai-api-version
queue-provider
rabbitmq-url
servicebus-namespace
smtp-port
github-client-id
github-client-secret
github-webhook-secret
jwt-secret
token-encryption-key
database-url
gemini-api-key
azure-openai-api-key
entra-client-secret
servicebus-connection-string
smtp-host
smtp-user
smtp-password
smtp-from
```

Update these values in `argocd/config/helm/values-dev.yaml` and `argocd/config/helm/values-prod.yaml`:

```text
keyVault.name
keyVault.tenantId
keyVault.userAssignedIdentityID
```

The prod Applications read from `main`, so merge this `argocd` folder to `main` before expecting prod apps to sync successfully.
