# 12 - Deploying to AKS & Azure Container Apps

Once our `.NET 8 Web API` image is sitting versioned inside **Azure Container Registry (ACR)** from Module 11, we have two primary container runtime targets in Azure: **Azure Container Apps (ACA)** and **Azure Kubernetes Service (AKS)**.

Today I wired up deployment pipelines for both—exploring when serverless KEDA-powered Container Apps is the sweet spot vs. when a full AKS cluster with Kubernetes manifests and Helm charts is worth the extra operational control.

---

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/a060042b-2164-4074-911c-fcab839a9d1d" />


## 1. Azure Container Apps vs. AKS (When to Pick Which)
Both run Linux containers backed by Kubernetes under the hood, but they sit at very different points on the control-vs-simplicity spectrum:

- **Azure Container Apps (ACA) — Serverless Microservices:**
  - Zero cluster management—Microsoft manages the underlying Kubernetes control plane, nodes, and Envoy ingress for you.
  - Built-in **KEDA** auto-scaling (including scaling down to **0 replicas** during idle hours, saving serious dev/test cost) and native **Dapr** sidecar support.
  - **Best for:** Event-driven .NET APIs, background workers, and microservice teams that want container portability without maintaining YAML manifests or upgrading K8s node pools.
- **Azure Kubernetes Service (AKS) — Full Orchestration Control:**
  - Direct access to the Kubernetes API (`kubectl`), custom network policies, DaemonSets, GPU nodes, service meshes (Istio), and multi-region clusters.
  - **Best for:** Large enterprise platforms with dedicated platform/ops engineers, strict networking compliance, or complex multi-service Helm topologies.

---

## 2. Option A: Deploying to Azure Container Apps (`AzureContainerApps@1`)
Container Apps makes CD remarkably clean. Once you create a Container App Environment and link its Managed Identity (`AcrPull`) to ACR, Azure Pipelines can roll out a new revision with **`AzureContainerApps@1`**—no `deployment.yaml` required:

```yaml
stages:
  - stage: DeployACA
    displayName: 'CD: Deploy to Azure Container Apps'
    jobs:
      - deployment: DeployContainerApp
        displayName: 'Rollout .NET Web API Revision'
        environment: 'aca-dev-env'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureContainerApps@1
                  displayName: 'Deploy Image to Container App'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    containerAppName: 'ca-dotnet-webapi-dev'
                    resourceGroup: 'rg-dotnet-aca-dev'
                    imageToDeploy: 'acrdotnetdemo01.azurecr.io/dotnet-webapi:$(Build.BuildId)'
                    targetPort: 8080
                    environmentVariables: |
                      ASPNETCORE_ENVIRONMENT=Development
```
(Tip: Every time AzureContainerApps@1 deploys a new image tag, ACA creates an immutable Revision and automatically shifts 100% of traffic once the new container passes its startup/readiness probes—or you can configure multi-revision traffic splitting for canary rollouts right in the portal or CLI.)

## 2. Option B: Deploying to AKS with Helm (HelmDeploy@0)
While KubernetesManifest@1 works fine for one-off YAML files, real .NET microservice teams package their Deployment, Service, Ingress, and HorizontalPodAutoscaler into a Helm Chart (charts/dotnet-webapi/) so environment differences live cleanly in values-dev.yaml and values-prod.yaml.

#### 1. Attach ACR & Register AKS Environment
```
# Grant AKS Kubelet identity permission to pull private images from ACR
az aks update \
  --resource-group rg-dotnet-aks-dev \
  --name aks-dotnet-demo-dev \
  --attach-acr acrdotnetdemo01
```
  Then register the AKS cluster + namespace (dotnet-app) under Pipelines → Environments → New environment → Kubernetes (aks-dev-env.dotnet-app).

  #### 2. Parameterized Helm Chart Snippet (charts/dotnet-webapi/templates/deployment.yaml)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-webapi
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: dotnet-webapi
  template:
    metadata:
      labels:
        app: dotnet-webapi
    spec:
      containers:
        - name: webapi
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: {{ .Values.aspNetCoreEnvironment | quote }}
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

#### 3. Pipeline Stage Using HelmDeploy@0
Notice how --set image.tag=$(Build.BuildId) injects the exact immutable build tag into our Helm release at deploy time:

```
stages:
  - stage: DeployAKS
    displayName: 'CD: Deploy Helm Chart to AKS'
    jobs:
      - deployment: HelmReleaseAKS
        displayName: 'Helm Upgrade .NET Web API'
        environment: 'aks-dev-env.dotnet-app'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self

                - task: HelmInstaller@1
                  displayName: 'Install Helm 3'
                  inputs:
                    helmVersionToInstall: 'latest'

                - task: HelmDeploy@0
                  displayName: 'Helm Upgrade --Install'
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    azureResourceGroup: 'rg-dotnet-aks-dev'
                    kubernetesCluster: 'aks-dotnet-demo-dev'
                    namespace: 'dotnet-app'
                    command: 'upgrade'
                    chartType: 'FilePath'
                    chartPath: '$(Build.SourcesDirectory)/charts/dotnet-webapi'
                    releaseName: 'dotnet-webapi'
                    valueFile: '$(Build.SourcesDirectory)/charts/dotnet-webapi/values-dev.yaml'
                    overrideValues: 'image.tag=$(Build.BuildId)'
                    install: true
                    waitForExecution: true
```

## 3. Gotchas I Ran Into
- Forgetting checkout: self in deployment: jobs: Unlike standard job: blocks, deployment: jobs skip cloning your Git repository by default. Without - checkout: self, HelmDeploy@0 won't find charts/dotnet-webapi on disk.

- Port 8080 alignment across both services: Since .NET 8 containers run as non-root USER app on port 8080, both ACA (targetPort: 8080 + ingress target port) and AKS (containerPort: 8080 + Service targetPort: 8080) must match—otherwise health probes fail and traffic never routes to the container.

- ACA minReplicas: 0 cold start: Scaling Container Apps to 0 saves money in Dev/QA, but the first HTTP request after an idle period waits 2–5 seconds while the container spins back up. In Production, always keep minReplicas: 1 (or 2 for high availability).

## 4. Hands-On Checklist
- [x] Deployed the versioned .NET 8 Web API image from ACR to Azure Container Apps using AzureContainerApps@1 and verified automatic revision creation + HTTPS ingress on port 8080.

- [x] Attached acrdotnetdemo01 to the AKS cluster (az aks update --attach-acr) and configured /health readiness and liveness probes.

- [x] Packaged the Kubernetes manifests into a reusable Helm Chart with values-dev.yaml and values-prod.yaml overrides.

- [x] Automated helm upgrade --install via HelmDeploy@0 with --wait enabled so failed pod rollouts automatically fail the pipeline stage.

- [ ] Up next in 13: Stop clicking around the Azure Portal to create App Services, SQL Databases, and Key Vaults—automate infrastructure provisioning via IaC Pipelines (Bicep & Terraform) with what-if / plan approval gates.
