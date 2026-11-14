# 04 - Azure Artifacts & Private NuGet Feeds

In real-world .NET teams, we rarely dump everything into one giant solution. Common DTOs, auth middleware, logging wrappers, and shared models live in separate class libraries so multiple Web APIs can reuse them without copy-pasting code. Today I set up **Azure Artifacts** to pack and publish a shared C# library as a private NuGet package and consume it cleanly inside another project.

---

## 1. How Azure Artifacts Fits In
Think of an Azure Artifacts **Feed** as your private, internal NuGet.org. Even better, when you enable **Upstream Sources**, your private feed acts as a single proxy—meaning your `nuget.config` only needs to point to *one* URL to fetch both your internal packages and public packages from `nuget.org` (and it caches those public packages in case `nuget.org` ever hiccups).

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/c6b6c634-58f7-4d97-b506-9898cf7fa60b" />


---

## 2. Setting Up the Feed & Local `nuget.config`
1. In Azure DevOps, go to **Artifacts → Create Feed**.
2. Give it a name (e.g., `dotnet-shared-feed`), keep **Upstream sources** checked so `nuget.org` packages flow through automatically, and pick either **Project-scoped** or **Organization-scoped** (Org-scoped is usually better if multiple projects need the same libraries).
3. Click **Connect to feed → dotnet** and drop a `nuget.config` file in the root of the consuming repository:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="dotnet-shared-feed" value="[https://pkgs.dev.azure.com/](https://pkgs.dev.azure.com/)<your-org>/_packaging/dotnet-shared-feed/nuget/v3/index.json" />
  </packageSources>
</configuration>
```

## 3. CI Pipeline to Pack & Push a NuGet Package
Whenever I update the shared class library (Shared.Contracts.csproj), I want the pipeline to pack it into a .nupkg with an auto-incrementing version and push it straight to the feed.

Using NuGetAuthenticate@1 paired with DotNetCoreCLI@2 is the cleanest approach on modern Ubuntu/Windows agents:

```
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  major: '1'
  minor: '0'
  # Auto-increments patch number on every build: 1.0.0, 1.0.1, 1.0.2...
  patch: $[counter(format('{0}.{1}', variables['major'], variables['minor']), 0)]
  packageVersion: '$(major).$(minor).$(patch)'

steps:
  - task: UseDotNet@2
    displayName: 'Use .NET 8 SDK'
    inputs:
      packageType: 'sdk'
      version: '8.x'

  # 1. Authenticate build agent with Azure Artifacts
  - task: NuGetAuthenticate@1
    displayName: 'Authenticate with Azure Artifacts'

  # 2. Build the class library
  - task: DotNetCoreCLI@2
    displayName: 'Build Library'
    inputs:
      command: 'build'
      projects: '**/Shared.Contracts.csproj'
      arguments: '--configuration Release'

  # 3. Pack into .nupkg with dynamic version
  - task: DotNetCoreCLI@2
    displayName: 'Pack NuGet Package ($(packageVersion))'
    inputs:
      command: 'pack'
      packagesToPack: '**/Shared.Contracts.csproj'
      nobuild: true
      versioningScheme: 'byEnvVar'
      versionEnvVar: 'packageVersion'

  # 4. Push .nupkg to private Azure Artifacts feed
  - task: DotNetCoreCLI@2
    displayName: 'Push to Azure Artifacts'
    inputs:
      command: 'push'
      packagesToPush: '$(Build.ArtifactStagingDirectory)/*.nupkg'
      nuGetFeedType: 'internal'
      publishVstsFeed: 'dotnet-shared-feed'
```

## 4. Consuming the Package in Another Pipeline
When my main WebApi solution restores packages inside its own CI pipeline, it also needs permission to read from dotnet-shared-feed. All it takes is adding NuGetAuthenticate@1 right before dotnet restore:

```
- task: NuGetAuthenticate@1
    displayName: 'Authenticate NuGet Feed'

  - task: DotNetCoreCLI@2
    displayName: 'Restore Solution'
    inputs:
      command: 'restore'
      projects: '**/*.sln'
      feedsToUse: 'config'
      nugetConfigPath: 'nuget.config'
```

## 5. Gotchas I Ran Into
- Immutability rule (409 Conflict): Once you push version 1.0.1 to an Azure Artifacts feed, you cannot overwrite it—even if you delete it from the feed, that version number stays burned forever. Using counter() in YAML variables guarantees every pipeline run gets a fresh patch version.

- Cross-project permission errors (401 Unauthorized): If your feed lives in Project A (or at the Org level) and your Web API pipeline runs in Project B, the restore will fail until you go to Artifacts → Feed Settings → Permissions and add [Project B] Build Service as a Reader (or Contributor if pushing).

- Local Visual Studio auth: Locally on Windows, Visual Studio handles Azure DevOps auth automatically once signed in. If you use dotnet restore from VS Code/terminal on Linux/Mac, install the Azure Artifacts Credential Provider (dotnet tool install -g Microsoft.Artifacts.CredentialProvider.NuGet.Tool) and pass --interactive on the first run.


## 6. Hands-On Checklist
- [x] Created an org-scoped feed dotnet-shared-feed with nuget.org upstream enabled.

- [x] Built a small Shared.Contracts class library containing common API response wrappers and DTOs.

- [x] Configured the pack + push YAML pipeline with counter() semantic versioning and verified 1.0.0 appeared in the feed.

- [x] Added nuget.config + NuGetAuthenticate@1 to the DotNet-FullStack-Demo Web API pipeline and restored the private package cleanly.

- [ ] Up next in 05: Dig into Build Agents (Microsoft-hosted vs. spinning up my own self-hosted agent) and Service Connections so pipelines can actually talk to my Azure subscription.
