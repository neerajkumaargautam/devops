# 03 - .NET Build & Test Automation

Moved from Git workflows into actual CI today. Before writing complex deployment pipelines, I wanted to make sure I understood how Azure DevOps builds a `.NET` solution from scratch, runs `xUnit` tests, and publishes test results + code coverage back into the pipeline UI so PR reviewers can actually see them.

Here are the CLI commands, YAML tasks, and a couple of path gotchas I ran into.

---

## 1. Local CLI vs. Azure Pipeline Tasks
Everything an Azure Pipeline does under the hood is just running the exact same `dotnet` CLI commands we run locally in Visual Studio or terminal—just on a clean, headless build agent.

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/3f16c8ea-ca45-43f5-8f53-43cfe982d0ee" />


### The Local Baseline
Before trusting a pipeline, I always verify the exact sequence locally from the root folder where the `.sln` lives:

```
# 1. Pull NuGet dependencies
dotnet restore

# 2. Compile in Release mode without re-running restore
dotnet build --configuration Release --no-restore

# 3. Run unit tests, output TRX results for AzDO, and collect code coverage
dotnet test --configuration Release --no-build --logger trx --collect:"XPlat Code Coverage"

# 4. Package the Web API into a clean publish folder ready for deployment
dotnet publish src/WebApi/WebApi.csproj --configuration Release --no-build --output ./publish

# 4. Package the Web API into a clean publish folder ready for deployment
dotnet publish src/WebApi/WebApi.csproj --configuration Release --no-build --output ./publish
```

*(Tip: Passing `--no-restore` on `build` and `--no-build` on `test` and `publish` saves noticeable time on build agents because it stops .NET from redoing work it already finished in the previous step.)*


## 2. Mapping CLI Commands to `DotNetCoreCLI@2`

In Azure Pipelines YAML, we use the built-in DotNetCoreCLI@2 task (and UseDotNet@2 if we need to pin a specific SDK version like .NET 8 or .NET 9 instead of relying on whatever the hosted agent happens to have).

Here is the working CI pipeline (azure-pipelines.yml) I wired up for my DotNet-FullStack-Demo repo:

```
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

steps:
  # 1. Pin the exact .NET SDK version
  - task: UseDotNet@2
    displayName: 'Use .NET 8 SDK'
    inputs:
      packageType: 'sdk'
      version: '8.x'

  # 2. Restore NuGet packages
  - task: DotNetCoreCLI@2
    displayName: 'Restore NuGet Packages'
    inputs:
      command: 'restore'
      projects: '**/*.sln'

  # 3. Build the solution
  - task: DotNetCoreCLI@2
    displayName: 'Build Solution ($(buildConfiguration))'
    inputs:
      command: 'build'
      projects: '**/*.sln'
      arguments: '--configuration $(buildConfiguration) --no-restore'

  # 4. Run Unit Tests & Publish Results to AzDO UI
  - task: DotNetCoreCLI@2
    displayName: 'Run xUnit Tests'
    inputs:
      command: 'test'
      projects: '**/*Tests/*.csproj'
      arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'
      publishTestResults: true

  # 5. Publish Code Coverage Report (Cobertura)
  - task: PublishCodeCoverageResults@2
    displayName: 'Publish Code Coverage'
    inputs:
      summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'

  # 6. Publish Web API Artifact
  - task: DotNetCoreCLI@2
    displayName: 'Publish Web API'
    inputs:
      command: 'publish'
      publishWebProjects: false
      projects: '**/WebApi.csproj'
      arguments: '--configuration $(buildConfiguration) --output$(Build.ArtifactStagingDirectory)'
      zipAfterPublish: true

  # 7. Upload Artifact so CD stages can grab it later
  - task: PublishBuildArtifacts@1
    displayName: 'Upload Build Artifact'
    inputs:
      PathtoPublish: '$(Build.ArtifactStagingDirectory)'
      ArtifactName: 'drop'

```
