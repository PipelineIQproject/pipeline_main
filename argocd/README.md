# PipelineIQ ArgoCD App of Apps

Apply only the root Application after ArgoCD and Argo Rollouts are installed:

```bash
kubectl apply -f argocd/application.yaml
```

The root app syncs every child `application.yaml` under this folder and ignores nested Helm chart files. Each child service creates two ArgoCD Applications:

- `<service>-dev` deploys from branch `dev` into namespace `pipelineiq-dev` with `values-dev.yaml`.
- `<service>-prod` deploys from branch `master` into namespace `pipelineiq-prod` with `values-prod.yaml`.

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

## Important

The config chart creates a placeholder `pipelineiq-kv-secrets` Secret in both namespaces. Replace placeholder secret values before using this for real traffic, or replace the Secret template with Azure Key Vault CSI / External Secrets.

The prod Applications read from `master`, so merge this `argocd` folder to `master` before expecting prod apps to sync successfully.
