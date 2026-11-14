# 10 - Zero-Downtime Deployments (Blue-Green, Canary & Slot Swaps)

Up through Module 9, every deployment pushed directly over the live Production slot—meaning users hit a 10–20 second cold start while the .NET runtime restarted and rebuilt its JIT cache. For a real production API, that brief downtime isn't acceptable.

Today I implemented **Zero-Downtime Deployments** using **Azure App Service Deployment Slots**, combining **Canary traffic splitting**, **Blue-Green VIP swaps**, and **automated health checks with instant rollback** directly inside Azure Pipelines.

---
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/3a958f40-0c87-4f14-8341-3601c9e2e67c" />


## 1. How Deployment Slots Achieve Zero Downtime
A **Deployment Slot** is a live, separate instance of your Azure App Service (e.g., `app-dotnet-demo-prod-staging.azurewebsites.net`) running alongside your primary `production` slot inside the same App Service Plan.

Instead of deploying new code directly to Production:
1. **Deploy to `staging`:** Azure Pipelines pushes the new build (`v2`) to the `staging` slot while `production` (`v1`) continues serving 100% of live traffic untouched.
2. **Application Initialization (Warmup):** Azure hits your `/health` endpoint on the `staging` slot so .NET loads assemblies, warms up EF Core/DI containers, and primes caches *before* a single real user sees it.
3. **VIP Swap (Blue-Green Cutover):** Azure updates the load balancer's Virtual IP (VIP) pointer so `staging` instantly becomes `production`—zero dropped requests, zero cold starts, and `v1` stays warm on the old slot ready for an instant rollback if anything goes wrong.

*(Note: Deployment Slots require an **Azure App Service Standard (`S1`)** tier or higher—they are not available on Free or Basic (`B1`) plans.)*

---

## 2. Blue-Green vs. Canary Releases
Depending on how risky a release is, slots let you choose between two rollout strategies:

- **Blue-Green (Instant VIP Swap):** You test `v2` privately on the `staging` URL, run smoke tests in your pipeline, and swap 100% of traffic over in one atomic operation. Best for standard releases with strong automated smoke tests.
- **Canary Release (Traffic Routing / Testing in Production):** Before performing a full swap, you route a small slice of real user traffic (e.g., **10%**) to the `staging` slot while **90%** stays on `production`. You watch Application Insights for HTTP 5xx spikes or latency regressions; if metrics look healthy, you ramp to 100% and swap.

---

## 3. Sticky vs. Non-Sticky Settings (Crucial Config Rule!)
Before writing any YAML, you have to configure your App Settings and Connection Strings properly in the Azure Portal (**Configuration → Application settings**):

- **Slot-specific ("Deployment slot setting" checked):** Stays locked to that specific slot during a swap. Use this for environment identifiers (`ASPNETCORE_ENVIRONMENT=Staging` vs `Production`), slot-specific database connection strings (if staging points to a staging DB), or feature flags.
- **Non-sticky (unchecked):** Travels *with* your code when the slots swap. Use this for build version numbers or app-level settings that belong to the release itself.

---

## 4. YAML Pipeline: Deploy to Staging → Canary/Smoke Check → Swap → Auto-Rollback
Here is the production deployment stage using `AzureWebApp@1` for slot deployment, a smoke test gate against `/health`, `AzureAppServiceManage@0` for the swap, and an automated rollback step that triggers *only* if post-swap verification fails:

```yaml
stages:
  - stage: DeployProdZeroDowntime
    displayName: 'CD: Zero-Downtime Production Release'
    variables:
      - group: 'vg-dotnet-prod'
    jobs:
      - deployment: BlueGreenDeploy
        displayName: 'Deploy Staging, Smoke Test & VIP Swap'
        environment: 'prod-env'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                # 1. Deploy new build to the 'staging' slot (Production stays untouched)
                - task: AzureWebApp@1
                  displayName: '1. Deploy v2 to Staging Slot'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    appType: 'webAppLinux'
                    appName: '$(WebAppName)'
                    deployToSlotOrASE: true
                    resourceGroupName: '$(ResourceGroupName)'
                    slotName: 'staging'
                    package: '$(Pipeline.Workspace)/drop/*.zip'

                # 2. Optional Canary: Route 10% live traffic to staging & verify health
                - task: AzureCLI@2
                  displayName: '2. Route 10% Canary Traffic & Warmup'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az webapp traffic-routing set \
                        --resource-group $(ResourceGroupName) \
                        --name $(WebAppName) \
                        --distribution staging=10

                      # Hit staging /health endpoint to verify HTTP 200 before swapping
                      STATUS=$(curl -s -o /dev/null -w "\%{http_code}" https://$(WebAppName)-staging.azurewebsites.net/health)
                      if [ "$STATUS" -ne 200 ]; then
                        echo "Staging health check failed with HTTP $STATUS! Aborting swap."
                        az webapp traffic-routing clear --resource-group $(ResourceGroupName) --name$(WebAppName)
                        exit 1
                      fi

                # 3. Reset traffic routing & perform zero-downtime VIP swap
                - task: AzureCLI@2
                  displayName: '3. Clear Canary Split Before Swap'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az webapp traffic-routing clear --resource-group $(ResourceGroupName) --name$(WebAppName)

                - task: AzureAppServiceManage@0
                  displayName: '4. Swap Staging Slot into Production'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    Action: 'Swap Slots'
                    WebAppName: '$(WebAppName)'
                    ResourceGroupName: '$(ResourceGroupName)'
                    SourceSlot: 'staging'

                # 5. Post-swap production smoke test
                - bash: |
                    PROD_STATUS=$(curl -s -o /dev/null -w "\%{http_code}" https://$(WebAppName).azurewebsites.net/health)
                    echo "Post-swap Production /health returned: $PROD_STATUS"
                    if [ "$PROD_STATUS" -ne 200 ]; then
                      exit 1
                    fi
                  displayName: '5. Post-Swap Production Verification'

                # 6. Automated Rollback: Swap previous v1 back if step 5 fails!
                - task: AzureAppServiceManage@0
                  displayName: '6. [AUTO-ROLLBACK] Revert Production Slot'
                  condition: failed()
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    Action: 'Swap Slots'
                    WebAppName: '$(WebAppName)'
                    ResourceGroupName: '$(ResourceGroupName)'
                    SourceSlot: 'staging'

```


(Tip: Always add a .dockerignore file next to your Dockerfile excluding bin/, obj/, .git/, and node_modules/ so COPY . . doesn't send hundreds of megabytes of local build junk into the Docker daemon.)

## 2. Setting Up ACR & Service Connection
- Create ACR in Azure: Spin up an Azure Container Registry (Basic tier is plenty for learning/dev—e.g., acrdotnetdemo01.azurecr.io). Keep Admin user disabled in favor of identity-based auth.

- Create Docker Registry Service Connection: In Azure DevOps, go to Project Settings → Service connections → New service connection → Docker Registry.

- Pick Azure Container Registry, select your subscription and acrdotnetdemo01, name the connection ACR-Docker-SvcConn, and save. (Under the hood, AzDO creates a Service Principal with AcrPush permissions on that registry.)

## 3. Building, Tagging & Pushing via Docker@2
Instead of manually running docker login, docker build, and docker push bash scripts, Azure Pipelines provides the built-in Docker@2 task.

Notice how I tag every image with both $(Build.BuildId) (for exact traceability and easy rollbacks) and latest:

```
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  dockerRegistryServiceConnection: 'ACR-Docker-SvcConn'
  imageRepository: 'dotnet-webapi'
  containerRegistry: 'acrdotnetdemo01.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
  - stage: BuildAndPushImage
    displayName: 'CI: Build & Push Docker Image'
    jobs:
      - job: DockerBuild
        displayName: 'Build & Push to ACR'
        steps:
          - task: Docker@2
            displayName: 'Build and Push Image to ACR'
            inputs:
              command: 'buildAndPush'
              repository: '$(imageRepository)'
              dockerfile: '$(dockerfilePath)'
              containerRegistry: '$(dockerRegistryServiceConnection)'
              tags: |
                $(tag)
                latest

  - stage: DeployContainer
    displayName: 'CD: Deploy Container to Dev'
    dependsOn: BuildAndPushImage
    condition: succeeded()
    jobs:
      - deployment: DeployWebAppContainer
        environment: 'dev-env'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebAppContainer@1
                  displayName: 'Deploy Image to Azure Web App for Containers'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    appName: 'app-dotnet-container-dev'
                    containers: '$(containerRegistry)/$(imageRepository):$(tag)'
```

## 4. Gotchas I Ran Into
- .NET 8 default container port changed to 8080: In .NET 6/7, official ASP.NET Docker images listened on port 80 as root. Starting in .NET 8, Microsoft switched the default non-root USER app port to 8080. If your Azure Web App expects port 80, set the App Setting WEBSITES_PORT=8080 or the container will fail health checks on startup.

- Avoid deploying :latest in Production: Always pass the immutable $(Build.BuildId) tag (e.g., dotnet-webapi:412) into AzureWebAppContainer@1. If you deploy :latest, Azure won't always detect that the underlying image digest changed, and rolling back to yesterday's release becomes a guessing game.

- Docker layer caching on hosted agents: Because Microsoft-hosted agents (ubuntu-latest) are destroyed after every job, Docker starts with an empty cache each run. Copying *.csproj and running dotnet restore before COPY . . still helps during local dev, and for larger enterprise solutions you can add Cache@2 or build on a persistent self-hosted agent.

## 5. Hands-On Checklist
- [x] Added a multi-stage Dockerfile (SDK → ASP.NET 8 runtime) and .dockerignore to the repository root, and verified docker build + docker run -p 8080:8080 locally.

- [x] Provisioned an Azure Container Registry (acrdotnetdemo01) and linked it via an ACR-Docker-SvcConn service connection.

- [x] Configured Docker@2 (buildAndPush) to tag images with both $(Build.BuildId) and latest.

- [x] Deployed the versioned container image to Azure Web App for Containers using AzureWebAppContainer@1 and confirmed the Swagger UI loaded cleanly on port 8080.

- [ ] Up next in 11: Take these containerized workloads to the next level by deploying to Azure Kubernetes Service (AKS) with Kubernetes manifests and Helm charts.
