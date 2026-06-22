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

## Key Vault Secrets

The config chart creates only the Azure Key Vault CSI `SecretProviderClass`. It syncs Key Vault secrets into the Kubernetes Secret named `pipelineiq-kv-secrets`.

Create these secrets in Azure Key Vault:

```text
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

The running services still reference `pipelineiq-config` for non-secret environment values and `pipelineiq-kv-secrets` for secret values. Keep managing `pipelineiq-config` separately unless you want the service Rollout templates changed to read all environment from Key Vault.

The prod Applications read from `main`, so merge this `argocd` folder to `main` before expecting prod apps to sync successfully.
