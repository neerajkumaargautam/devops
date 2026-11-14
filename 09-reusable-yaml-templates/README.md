# 09 - Reusable YAML Templates (DRY Pipelines)

After building the multi-stage pipeline in Module 08, one annoyance jumped right out: I had copy-pasted the exact same `AzureWebApp@1` deployment block three separate times for **Dev**, **QA**, and **Prod**. Imagine having 10 .NET microservices—updating a single deployment step would mean editing 30 places and hoping you don't introduce a typo.

Today I refactored that repetitive pipeline using **YAML Templates**, applying the **DRY (Don't Repeat Yourself)** principle so one parameterized blueprint handles every environment cleanly.

---
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/038b9b50-8cc4-4d77-9923-20597e3e7b6a" />

## 1. Template Scope & Types (`includes` vs. `extends`)
Azure DevOps lets you extract almost any level of the YAML hierarchy into its own reusable file:
- **Step templates (`steps:`):** Reuse a sequence of build/test tasks across multiple jobs.
- **Job templates (`jobs:`):** Reuse an entire agent job (e.g., standardizing how every .NET library builds and runs code coverage).
- **Stage templates (`stages:`):** Reuse an entire deployment stage (`Dev`, `QA`, `Prod`) by passing environment-specific parameters.
- **Variable templates (`variables:`):** Share common non-secret constants across repositories.

*(Note: Besides inserting templates with `- template:`, organizations also use **`extends:`** templates to enforce mandatory security/compliance wrappers around every team's pipeline.)*

---

## 2. Folder Structure for Clean Pipelines
Instead of dumping everything into one massive root YAML file, I organized my repository with a dedicated `templates/` folder:

```text
DotNet-FullStack-Demo/
├── azure-pipelines.yml              # Main orchestrator (clean & readable)
├── templates/
│   ├── dotnet-build-steps.yml       # Reusable CI build & test steps
│   └── deploy-webapp-stage.yml      # Reusable CD deployment stage
└── src/
    └── WebApi/
```
## 3. Creating the Parameterized Stage Template (`templates/deploy-webapp-stage.yml`)

Every template starts with a parameters: block where you define strongly typed inputs (string, boolean, number, object, stepList, etc.) and optional default values:

```
# templates/deploy-webapp-stage.yml
parameters:
  - name: stageName
    type: string
  - name: displayName
    type: string
  - name: dependsOn
    type: string
  - name: environmentName
    type: string
  - name: variableGroup
    type: string
  - name: azureSubscription
    type: string
    default: 'Azure-Subscription-SvcConn'
  - name: isProduction
    type: boolean
    default: false

stages:
  - stage: ${{ parameters.stageName }}
    displayName: ${{ parameters.displayName }}
    dependsOn: ${{ parameters.dependsOn }}
    # Enforce main-branch-only safety automatically when isProduction is true
    ${{ if eq(parameters.isProduction, true) }}:
      condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    ${{ else }}:
      condition: succeeded()
    variables:
      - group: ${{ parameters.variableGroup }}
    jobs:
      - deployment: DeployWebApp
        displayName: 'Deploy to ${{ parameters.environmentName }}'
        environment: ${{ parameters.environmentName }}
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy Zip to Azure App Service'
                  inputs:
                    azureSubscription: ${{ parameters.azureSubscription }}
                    appType: 'webAppLinux'
                    appName: '$(WebAppName)'
                    package: '$(Pipeline.Workspace)/drop/*.zip'
```

## 4. Refactoring the Main Pipeline (`azure-pipelines.yml`)

Now look how clean and readable the main azure-pipelines.yml becomes—the entire 3-environment CD rollout shrinks down to three concise template calls:

```
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  # Stage 1: CI Build & Publish
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
            displayName: 'Restore, Build & Test'
            inputs:
              command: 'test'
              projects: '**/*Tests/*.csproj'
              arguments: '--configuration Release'

          - task: DotNetCoreCLI@2
            displayName: 'Publish Web API'
            inputs:
              command: 'publish'
              publishWebProjects: false
              projects: '**/WebApi.csproj'
              arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)'
              zipAfterPublish: true

          - publish: '$(Build.ArtifactStagingDirectory)'
            artifact: 'drop'

  # Stage 2: Dev Deployment (via Reusable Template)
  - template: templates/deploy-webapp-stage.yml
    parameters:
      stageName: 'DeployDev'
      displayName: 'Stage 2: Deploy to Dev'
      dependsOn: 'Build'
      environmentName: 'dev-env'
      variableGroup: 'vg-dotnet-dev'

  # Stage 3: QA Deployment (via Reusable Template)
  - template: templates/deploy-webapp-stage.yml
    parameters:
      stageName: 'DeployQA'
      displayName: 'Stage 3: Deploy to QA'
      dependsOn: 'DeployDev'
      environmentName: 'qa-env'
      variableGroup: 'vg-dotnet-qa'

  # Stage 4: Production Deployment (via Reusable Template)
  - template: templates/deploy-webapp-stage.yml
    parameters:
      stageName: 'DeployProd'
      displayName: 'Stage 4: Deploy to Production'
      dependsOn: 'DeployQA'
      environmentName: 'prod-env'
      variableGroup: 'vg-dotnet-prod'
      isProduction: true

 ```

## 4. Refactoring the Main Pipeline (`azure-pipelines.yml`)

```
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  # Stage 1: CI Build & Publish
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
            displayName: 'Restore, Build & Test'
            inputs:
              command: 'test'
              projects: '**/*Tests/*.csproj'
              arguments: '--configuration Release'

          - task: DotNetCoreCLI@2
            displayName: 'Publish Web API'
            inputs:
              command: 'publish'
              publishWebProjects: false
              projects: '**/WebApi.csproj'
              arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)'
              zipAfterPublish: true

          - publish: '$(Build.ArtifactStagingDirectory)'
            artifact: 'drop'

  # Stage 2: Dev Deployment (via Reusable Template)
  - template: templates/deploy-webapp-stage.yml
    parameters:
      stageName: 'DeployDev'
      displayName: 'Stage 2: Deploy to Dev'
      dependsOn: 'Build'
      environmentName: 'dev-env'
      variableGroup: 'vg-dotnet-dev'

  # Stage 3: QA Deployment (via Reusable Template)
  - template: templates/deploy-webapp-stage.yml
    parameters:
      stageName: 'DeployQA'
      displayName: 'Stage 3: Deploy to QA'
      dependsOn: 'DeployDev'
      environmentName: 'qa-env'
      variableGroup: 'vg-dotnet-qa'

  # Stage 4: Production Deployment (via Reusable Template)
  - template: templates/deploy-webapp-stage.yml
    parameters:
      stageName: 'DeployProd'
      displayName: 'Stage 4: Deploy to Production'
      dependsOn: 'DeployQA'
      environmentName: 'prod-env'
      variableGroup: 'vg-dotnet-prod'
      isProduction: true
```
## 6. Gotchas I Hit Along the Way
- ${{ parameters.x }} vs. $(x): Template parameters are always compile-time expressions, so you must reference them with ${{ parameters.stageName }}. Using $(parameters.stageName) will fail silently or treat it as a literal string.

- Relative path resolution: Template paths are always resolved relative to the YAML file that calls them—not the root of the repository. If templates/stage.yml calls another template in the same folder, it just writes - template: steps.yml.

- Debugging expanded YAML: When nested templates and ${{ if }} conditions start getting complex, click ⋮ (three dots) → Download full YAML inside the Azure DevOps pipeline editor. It compiles all your templates into one flattened YAML file so you can inspect the exact output before running.

## 7. Hands-On Checklist
- [x] Extracted the repetitive AzureWebApp@1 deployment stage into templates/deploy-webapp-stage.yml with typed parameters:.

- [x] Added compile-time conditional logic (${{ if eq(parameters.isProduction, true) }}:) to automatically enforce main-branch protection on Prod runs.

- [x] Refactored azure-pipelines.yml to call the template for Dev, QA, and Prod and verified the compiled output using Download full YAML.

- [x] Tested cross-repository template referencing (@sharedTemplates) pinned to a Git tag.

- [ ] Up next in 10: Containerize the .NET Web API with Docker, push versioned images to Azure Container Registry (ACR), and deploy containers via Azure Pipelines.
