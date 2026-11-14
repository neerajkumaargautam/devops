# 13 - IaC Pipelines (Bicep & Terraform)

Up until now, every Azure resource our pipelines deployed to—App Services, Azure SQL, Key Vault, ACR—had to be clicked together manually in the Azure Portal beforehand. That works for a single dev sandbox, but the moment you spin up QA, UAT, or Production, manual clicking guarantees configuration drift and missing settings.

Today I moved infrastructure into Git using **Infrastructure as Code (IaC)**—comparing **Azure Bicep** and **Terraform**, and wiring both into Azure Pipelines with **`what-if` / `plan` preview stages** and **manual approval gates** so nothing gets modified or destroyed in Azure without a human reviewing the diff first.

---
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/a54dd3f9-4163-4210-946a-22a39da8abea" />


## 1. Azure Bicep vs. Terraform (Which One to Pick?)
Both let you declare your cloud resources idempotently (running the same template 10 times produces the exact same infrastructure without duplicating resources), but they manage state differently:

- **Azure Bicep (Native & Stateless):**
  - Compiles directly into Azure Resource Manager (ARM) JSON under the hood.
  - **Zero state file to manage:** Azure Resource Manager itself *is* the state—you just run `az deployment group what-if` and `az deployment group create`.
  - **Best for:** Teams 100% committed to Azure who want day-one support for every new Azure feature without maintaining remote storage accounts for `.tfstate`.
- **Terraform (Multi-Cloud & Stateful):**
  - Uses HCL (`main.tf`) and tracks every provisioned resource inside a **`terraform.tfstate`** file (stored remotely in an Azure Blob Storage container with state locking).
  - **Best for:** Organizations running multi-cloud/hybrid setups (Azure + AWS + GitHub/AzDO providers) or teams standardized on HashiCorp tooling.

---

## 2. Option A: Azure Bicep Pipeline (`what-if` → Approval Gate → `create`)
A clean Bicep template (`infra/main.bicep`) provisions the App Service Plan, Linux Web App, Azure Key Vault, and Azure SQL Server/Database together—and even wires the Web App's **Managed Identity** directly into Key Vault so the .NET API reads secrets without passwords:

```bicep
// infra/main.bicep
param envName string = 'dev'
param location string = resourceGroup().location

resource appServicePlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-dotnet-demo-${envName}'
  location: location
  sku: {
    name: envName == 'prod' ? 'S1' : 'B1'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

resource webApp 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-dotnet-demo-${envName}'
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      appSettings: [
        {
          name: 'ASPNETCORE_ENVIRONMENT'
          value: envName == 'prod' ? 'Production' : 'Development'
        }
      ]
    }
  }
}

resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-dotnet-${envName}-01'
  location: location
  properties: {
    tenantId: subscription().tenantId
    sku: {
      family: 'A'
      name: 'standard'
    }
    enableRbacAuthorization: true
  }
}
```

Bicep CI/CD YAML (azure-pipelines-bicep.yml)
Notice how Stage 1 runs az deployment group what-if so reviewers can inspect every + Create, ~ Modify, and - Delete action before approving Stage 2:

```
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - 'infra/*'

pool:
  vmImage: 'ubuntu-latest'

variables:
  resourceGroupName: 'rg-dotnet-demo-prod'
  location: 'centralindia'

stages:
  # Stage 1: Lint & Preview Changes (What-If)
  - stage: ValidateAndWhatIf
    displayName: '1. Bicep Lint & What-If Preview'
    jobs:
      - job: BicepWhatIf
        steps:
          - task: AzureCLI@2
            displayName: 'Build Bicep & Run What-If'
            inputs:
              azureSubscription: 'Azure-Subscription-SvcConn'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az bicep build --file infra/main.bicep
                az group create --name $(resourceGroupName) --location$(location)
                az deployment group what-if \
                  --resource-group $(resourceGroupName) \
                  --template-file infra/main.bicep \
                  --parameters envName=prod

  # Stage 2: Pauses on 'prod-infra-env' Approval Gate before applying!
  - stage: DeployBicepInfra
    displayName: '2. Apply Bicep Infrastructure'
    dependsOn: ValidateAndWhatIf
    condition: succeeded()
    jobs:
      - deployment: ApplyBicep
        environment: 'prod-infra-env' # Attach Manual Approval check here!
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self

                - task: AzureCLI@2
                  displayName: 'Deploy Bicep Template'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment group create \
                        --resource-group $(resourceGroupName) \
                        --template-file infra/main.bicep \
                        --parameters envName=prod
```

## 2. Option B: Terraform Pipeline (plan Artifact → Approval Gate → apply)
In Terraform, you never run terraform apply -auto-approve blindly against a fresh calculation. Instead, Stage 1 saves the exact approved execution plan into a binary file (tfplan), publishes it as a pipeline artifact, and Stage 2 applies that exact saved plan after human approval:

```
stages:
  # Stage 1: Init, Validate & Save Binary Plan
  - stage: TerraformPlan
    displayName: '1. Terraform Validate & Plan'
    jobs:
      - job: PlanJob
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: TerraformInstaller@1
            displayName: 'Install Terraform'
            inputs:
              terraformVersion: 'latest'

          - task: TerraformTaskV4@4
            displayName: 'Terraform Init (Remote Azure Blob State)'
            inputs:
              provider: 'azurerm'
              command: 'init'
              workingDirectory: '$(System.DefaultWorkingDirectory)/terraform'
              backendServiceArm: 'Azure-Subscription-SvcConn'
              backendAzureRmResourceGroupName: 'rg-tfstate-mgmt'
              backendAzureRmStorageAccountName: 'sttfstatedotnet01'
              backendAzureRmContainerName: 'tfstate'
              backendAzureRmKey: 'prod.terraform.tfstate'

          - task: TerraformTaskV4@4
            displayName: 'Terraform Plan (-out=tfplan)'
            inputs:
              provider: 'azurerm'
              command: 'plan'
              workingDirectory: '$(System.DefaultWorkingDirectory)/terraform'
              environmentServiceNameAzureRM: 'Azure-Subscription-SvcConn'
              commandOptions: '-out=$(Build.ArtifactStagingDirectory)/tfplan'

          - publish: '$(Build.ArtifactStagingDirectory)/tfplan'
            artifact: 'tfplan-artifact'

  # Stage 2: Human Reviews Plan Output -> Approves -> Applies Exact Plan
  - stage: TerraformApply
    displayName: '2. Terraform Apply'
    dependsOn: TerraformPlan
    condition: succeeded()
    jobs:
      - deployment: ApplyJob
        environment: 'prod-infra-env'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self

                - task: TerraformInstaller@1
                  inputs:
                    terraformVersion: 'latest'

                - task: TerraformTaskV4@4
                  displayName: 'Terraform Init'
                  inputs:
                    provider: 'azurerm'
                    command: 'init'
                    workingDirectory: '$(System.DefaultWorkingDirectory)/terraform'
                    backendServiceArm: 'Azure-Subscription-SvcConn'
                    backendAzureRmResourceGroupName: 'rg-tfstate-mgmt'
                    backendAzureRmStorageAccountName: 'sttfstatedotnet01'
                    backendAzureRmContainerName: 'tfstate'
                    backendAzureRmKey: 'prod.terraform.tfstate'

                - task: TerraformTaskV4@4
                  displayName: 'Terraform Apply Saved Plan'
                  inputs:
                    provider: 'azurerm'
                    command: 'apply'
                    workingDirectory: '$(System.DefaultWorkingDirectory)/terraform'
                    environmentServiceNameAzureRM: 'Azure-Subscription-SvcConn'
                    commandOptions: '$(Pipeline.Workspace)/tfplan-artifact/tfplan'
```
## 3. Gotchas I Ran Into
- Service Connection RBAC permissions (Contributor vs. User Access Administrator): A standard Contributor service connection can create App Services, SQL Databases, and Key Vaults—but if your Bicep or Terraform code assigns Azure RBAC roles (like granting the Web App's Managed Identity Key Vault Secrets User access), apply will fail with AuthorizationFailed unless the Service Principal also has User Access Administrator (or Role Based Access Control Administrator).

- Protecting Terraform remote state: Never commit terraform.tfstate to Git—it contains plaintext secrets and resource IDs. Always store it in an Azure Storage Account with versioning + soft-delete enabled, and remember that TerraformTaskV4@4 requires terraform init to run in both the Plan job and the Apply job since each runs on a fresh hosted agent.

- Globally unique Azure names: Storage Accounts, Key Vaults, and App Services require globally unique names across all of Azure. Append uniqueString(resourceGroup().id) in Bicep (or random_string in Terraform) so new environments never collide on naming.

## 4. Hands-On Checklist
- [x] Authored infra/main.bicep to provision a Linux App Service Plan, .NET 8 Web App with System-Assigned Managed Identity, and RBAC-enabled Key Vault.

- [x] Created a dedicated prod-infra-env Environment in Azure DevOps with a Manual Approval gate so reviewers inspect what-if / plan diffs before any cloud mutation occurs.

- [x] Tested both the Azure Bicep (az deployment group what-if → create) and Terraform (plan -out=tfplan → apply) pipelines with remote Blob state storage.

- [x] Verified idempotency by re-running the pipeline with zero code changes and confirming 0 to add, 0 to change, 0 to destroy.

- [ ] Up next in 14: Shift security left with DevSecOps & Code Quality—integrating SonarQube / SonarCloud quality gates for C#, dotnet list package --vulnerable, Trivy container image scanning, and credential leak checks.
