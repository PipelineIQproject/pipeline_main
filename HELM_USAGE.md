# PipelineIQ Helm Layout

The original flat manifests are still present as a safe fallback. The Helm charts are split by component.

Install shared resources first:

```powershell
helm upgrade --install pipelineiq-config ./k8s/config -f ./k8s/config/values-prod.yaml
helm upgrade --install pipelineiq-keyvault ./k8s/keyvault -n pipelineiq -f ./k8s/keyvault/values-prod.yaml
helm upgrade --install rabbitmq ./k8s/rabbitmq -n pipelineiq -f ./k8s/rabbitmq/values-prod.yaml
```

Install application services:

```powershell
helm upgrade --install frontend ./k8s/frontend -n pipelineiq -f ./k8s/frontend/values-prod.yaml
helm upgrade --install auth-service ./k8s/auth-service -n pipelineiq -f ./k8s/auth-service/values-prod.yaml
helm upgrade --install dashboard-api ./k8s/dashboard-api -n pipelineiq -f ./k8s/dashboard-api/values-prod.yaml
helm upgrade --install github-integration-service ./k8s/github-integration-service -n pipelineiq -f ./k8s/github-integration-service/values-prod.yaml
helm upgrade --install pipeline-runner-service ./k8s/pipeline-runner-service -n pipelineiq -f ./k8s/pipeline-runner-service/values-prod.yaml
helm upgrade --install workflow-analyzer-service ./k8s/workflow-analyzer-service -n pipelineiq -f ./k8s/workflow-analyzer-service/values-prod.yaml
helm upgrade --install gemini-ai-service ./k8s/gemini-ai-service -n pipelineiq -f ./k8s/gemini-ai-service/values-prod.yaml
helm upgrade --install webhook-service ./k8s/webhook-service -n pipelineiq -f ./k8s/webhook-service/values-prod.yaml
helm upgrade --install notification-service ./k8s/notification-service -n pipelineiq -f ./k8s/notification-service/values-prod.yaml
```

Install ingress after the services exist:

```powershell
helm upgrade --install api-gateway ./k8s/api-gateway -n pipelineiq -f ./k8s/api-gateway/values-prod.yaml
```

Render before applying:

```powershell
helm template frontend ./k8s/frontend -n pipelineiq -f ./k8s/frontend/values-prod.yaml
```
