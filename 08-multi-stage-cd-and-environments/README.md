# 08 - Multi-Stage CD & Environments

Up until now, my pipelines stopped right after `dotnet publish` and uploading a zip artifact. Today I connected the **CD (Continuous Delivery)** half—taking that single build artifact and promoting it across **Dev → QA → Prod** using Azure DevOps **Environments** and manual approval gates.

Here is how `deployment` jobs differ from regular jobs, how Environments lock down production, and the full multi-stage YAML I put together.

---
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/210ac707-d51e-477b-959c-6226e9da6985" />


## 1. Regular `job:` vs. `deployment:` Job
When deploying to an environment, you don't use a standard `job:` block—you use a special **`deployment:`** job. Why?

- **Deployment history tracking:** Every run logs which commit, work items, and pipeline build landed in `dev-app` vs `prod-app` directly inside **Pipelines → Environments**.
- **Automatic artifact download:** Unlike standard jobs where you have to write a `DownloadBuildArtifacts` step manually, a `deployment:` job automatically downloads all pipeline artifacts produced in earlier stages into `$(Pipeline.Workspace)`.
- **Environment hooks:** It unlocks `runOnce`, `rolling`, and `canary` deployment lifecycle hooks (`preDeploy`, `deploy`, `routeTraffic`, `postRouteTraffic`).

---

## 2. Setting Up Environments & Approval Gates
An **Environment** in Azure DevOps (`Pipelines → Environments`) is a logical target representing where your app runs (e.g., `dev-env`, `qa-env`, `prod-env`). Even better, you can attach **Approvals and Checks** directly to the environment so *no YAML pipeline* can deploy to Production without sign-off—even if someone edits the YAML file!

1. Go to **Pipelines → Environments → New environment** and create three environments: `dev-env`, `qa-env`, and `prod-env` (select **None** for resource type when deploying to Azure App Service).
2. Open `prod-env` → click the **`⋮` (three dots)** in the top right → **Approvals and checks**.
3. Add **Approvals** and assign yourself (or a Tech Lead / QA group). You can also add **Branch control** (`refs/heads/main`) so nobody can accidentally deploy a random feature branch to Production.

---

## 3. Complete Multi-Stage CI/CD YAML (`azure-pipelines.yml`)
Notice how each deployment stage uses `dependsOn:` to enforce order, `condition: succeeded()` so Prod never deploys if QA fails, and imports its own environment-specific Variable Group:

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  # ==========================================
  # STAGE 1: CI (Build, Test & Publish Drop)
  # ==========================================
  - stage: Build
    displayName: 'Stage 1: Build & Test .NET API'
    jobs:
      - job: BuildJob
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: 'sdk'
              version: '8.x'

          - task: DotNetCoreCLI@2
            displayName: 'Restore & Build'
            inputs:
              command: 'build'
              projects: '**/*.sln'
              arguments: '--configuration Release'

          - task: DotNetCoreCLI@2
            displayName: 'Run Unit Tests'
            inputs:
              command: 'test'
              projects: '**/*Tests/*.csproj'
              arguments: '--configuration Release --no-build'

          - task: DotNetCoreCLI@2
            displayName: 'Publish Web API Zip'
            inputs:
              command: 'publish'
              publishWebProjects: false
              projects: '**/WebApi.csproj'
              arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)'
              zipAfterPublish: true

          - publish: '$(Build.ArtifactStagingDirectory)'
            artifact: 'drop'

  # ==========================================
  # STAGE 2: CD -> DEV (Automatic on main)
  # ==========================================
  - stage: DeployDev
    displayName: 'Stage 2: Deploy to Dev'
    dependsOn: Build
    condition: succeeded()
    variables:
      - group: 'vg-dotnet-dev'
    jobs:
      - deployment: DeployToDev
        environment: 'dev-env'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy Zip to Dev App Service'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    appType: 'webAppLinux'
                    appName: '$(WebAppName)' # e.g., app-dotnet-demo-dev
                    package: '$(Pipeline.Workspace)/drop/*.zip'

  # ==========================================
  # STAGE 3: CD -> QA (Automatic after Dev)
  # ==========================================
  - stage: DeployQA
    displayName: 'Stage 3: Deploy to QA'
    dependsOn: DeployDev
    condition: succeeded()
    variables:
      - group: 'vg-dotnet-qa'
    jobs:
      - deployment: DeployToQA
        environment: 'qa-env'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy Zip to QA App Service'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    appType: 'webAppLinux'
                    appName: '$(WebAppName)' # e.g., app-dotnet-demo-qa
                    package: '$(Pipeline.Workspace)/drop/*.zip'

  # ==========================================
  # STAGE 4: CD -> PROD (Pauses for Approval!)
  # ==========================================
  - stage: DeployProd
    displayName: 'Stage 4: Deploy to Production'
    dependsOn: DeployQA
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    variables:
      - group: 'vg-dotnet-prod'
    jobs:
      - deployment: DeployToProd
        environment: 'prod-env'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy Zip to Prod App Service'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    appType: 'webAppLinux'
                    appName: '$(WebAppName)' # e.g., app-dotnet-demo-prod
                    package: '$(Pipeline.Workspace)/drop/*.zip'
```

4. Gotchas I Ran Into
- $(Pipeline.Workspace)/drop path: In older classic pipelines, artifacts landed in $(System.DefaultWorkingDirectory). In modern YAML deployment: jobs, download: current automatically places your published artifact inside $(Pipeline.Workspace)/<artifact-name>/.

- Approval timeout: By default, manual approvals on an Environment wait up to 30 days before timing out and marking the stage as rejected. You can tighten that down to a few hours or days in the approval settings.

- Notice the YAML bloat: Writing the exact same AzureWebApp@1 block three separate times for Dev, QA, and Prod works, but it's already getting repetitive—which is the exact problem YAML templates solve next.

5. Hands-On Checklist
- [x] Created dev-env, qa-env, and prod-env under Pipelines → Environments.

- [x] Added a Manual Approval check and main Branch control check on prod-env.

- [x] Updated azure-pipelines.yml to use deployment: jobs with strategy: runOnce and AzureWebApp@1.

- [x] Triggered a run on main—watched Build, DeployDev, and DeployQA turn green automatically, then saw DeployProd pause with a "Review" button waiting for my approval.

- [ ] Up next in 09: Refactor this repetitive 120-line YAML file using Reusable YAML Templates so we don't copy-paste deployment steps across every environment and microservice.
