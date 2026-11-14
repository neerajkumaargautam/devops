# 16 - Pipeline Performance & Caching

Once you add SonarCloud, Trivy, EF Core bundles, and an Angular frontend build into a single CI pipeline, a fresh run on a Microsoft-hosted agent easily creeps from **2 minutes** up to **8–10 minutes**. Waiting 10 minutes on every Pull Request commit kills developer flow—and eats through parallel job minutes fast.

Today I optimized our full-stack `.NET 8 + Angular` pipeline using **dependency caching (`Cache@2`)**, **parallel agent jobs**, **shallow Git clones**, and **path-based conditional execution** to cut CI execution time roughly in half.

---
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/306ec88b-2e5a-46aa-a9ac-4da3b2827bd7" />

## 1. Caching NuGet & npm Dependencies with `Cache@2`
Because Microsoft-hosted agents (`ubuntu-latest`) start from a clean VM on every run, `dotnet restore` and `npm ci` re-download hundreds of megabytes of packages from the internet every single time. The built-in **`Cache@2`** task zips your package cache folder into Azure DevOps cloud storage keyed by a hash of your lockfile:

- **NuGet Caching (`packages.lock.json`):**
  1. Enable deterministic lockfiles in your `.csproj` files by adding `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>` and commit the generated `packages.lock.json` files to Git.
  2. Set `NUGET_PACKAGES: $(Pipeline.Workspace)/.nuget/packages` and point `Cache@2` at that directory using `key: 'nuget | "$(Agent.OS)" | **/packages.lock.json'`.
  3. On a **Cache Hit**, skip `dotnet restore` completely (or run `dotnet restore --locked-mode` in ~2 seconds).
- **Angular / npm Caching (`~/.npm`):**
  - Instead of caching the huge `node_modules/` folder directly (which often breaks native OS symlinks), cache the global **npm cache directory** (`$(Pipeline.Workspace)/.npm`) keyed by `**/package-lock.json` and run `npm ci --cache $(Pipeline.Workspace)/.npm --prefer-offline`.

---

## 2. Parallel Jobs + Conditional Execution (Monorepo Optimization)
If a developer only tweaks a C# controller in `src/WebApi/`, there is zero reason to rebuild the entire Angular SPA—and vice versa. Even when both *do* change, running them in **parallel jobs** across two agents avoids stacking their build times sequentially.

### Optimized Full-Stack CI YAML (`azure-pipelines-fast.yml`)
```yaml
trigger:
  branches:
    include:
      - main

variables:
  NUGET_PACKAGES: $(Pipeline.Workspace)/.nuget/packages
  npm_config_cache: $(Pipeline.Workspace)/.npm

stages:
  # ==========================================
  # STAGE 1: DETECT CHANGED PATHS
  # ==========================================
  - stage: DetectChanges
    displayName: '1. Detect Changed Components'
    jobs:
      - job: PathFilter
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - checkout: self
            fetchDepth: 2 # Only need HEAD and HEAD~1 to diff!

          - bash: |
              CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)
              echo "Changed files:"
              echo "$CHANGED_FILES"

              if echo "$CHANGED_FILES" | grep -q "^src/WebApi/\|^tests/"; then
                echo "##vso[task.setvariable variable=BackendChanged;isOutput=true]true"
              else
                echo "##vso[task.setvariable variable=BackendChanged;isOutput=true]false"
              fi

              if echo "$CHANGED_FILES" | grep -q "^src/ClientApp/"; then
                echo "##vso[task.setvariable variable=FrontendChanged;isOutput=true]true"
              else
                echo "##vso[task.setvariable variable=FrontendChanged;isOutput=true]false"
              fi
            name: setChanges
            displayName: 'Evaluate Git Diff Paths'

  # ==========================================
  # STAGE 2: PARALLEL & CACHED BUILDS
  # ==========================================
  - stage: BuildAndTest
    displayName: '2. Parallel Cached Build & Test'
    dependsOn: DetectChanges
    jobs:
      # Job A: .NET Backend (Only runs if BackendChanged == true)
      - job: BuildDotNet
        displayName: '.NET 8 Web API (Cached)'
        condition: eq(stageDependencies.DetectChanges.PathFilter.outputs['setChanges.BackendChanged'], 'true')
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - checkout: self
            fetchDepth: 1 # Shallow clone for maximum speed

          - task: UseDotNet@2
            inputs:
              packageType: 'sdk'
              version: '8.x'

          - task: Cache@2
            displayName: 'Restore/Save NuGet Cache'
            inputs:
              key: 'nuget | "$(Agent.OS)" | **/packages.lock.json,!**/bin/**,!**/obj/**'
              restoreKeys: |
                nuget | "$(Agent.OS)"
              path: '$(NUGET_PACKAGES)'
              cacheHitVar: 'NUGET_CACHE_RESTORED'

          - script: dotnet restore --locked-mode
            displayName: 'Restore NuGet Packages'
            condition: ne(variables.NUGET_CACHE_RESTORED, 'true')

          - script: |
              dotnet build --configuration Release --no-restore
              dotnet test --configuration Release --no-build
            displayName: 'Build & Test .NET API'

      # Job B: Angular Frontend (Runs concurrently with Job A if FrontendChanged == true)
      - job: BuildAngular
        displayName: 'Angular SPA (Cached)'
        condition: eq(stageDependencies.DetectChanges.PathFilter.outputs['setChanges.FrontendChanged'], 'true')
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - checkout: self
            fetchDepth: 1

          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'

          - task: Cache@2
            displayName: 'Restore/Save npm Cache'
            inputs:
              key: 'npm | "$(Agent.OS)" | src/ClientApp/package-lock.json'
              restoreKeys: |
                npm | "$(Agent.OS)"
              path: '$(npm_config_cache)'

          - script: |
              cd src/ClientApp
              npm ci --prefer-offline
              npm run build -- --configuration production
            displayName: 'Install & Build Angular'
```

## 3. Extra Quick Wins That Shave Off Minutes
- Shallow Checkout (fetchDepth: 1): By default, Azure Pipelines clones the entire Git commit history on every job. For a 5-year-old enterprise repo, cloning history alone can take 60+ seconds. Setting fetchDepth: 1 downloads only the single latest commit snapshot. (Exception: keep fetchDepth: 0 on dedicated SonarCloud/Gitleaks jobs that need full blame history.)

- Skip dotnet restore during build / test / publish: Every dotnet build, dotnet test, and dotnet publish command runs an implicit dotnet restore unless you pass --no-restore (and --no-build on dotnet test). Passing those flags avoids redundant dependency graph checks across every step.

- Matrix Builds for Multi-Target Libraries: If you maintain a shared NuGet class library targeting both net8.0 and net9.0 (or testing across Linux and Windows), use strategy: matrix: to spin up parallel test runners automatically without duplicating YAML jobs.

## 4. Gotchas I Ran Into
- Cross-stage output variable syntax: Reading an output variable (isOutput=true) from a previous stage uses stageDependencies.<StageName>.<JobName>.outputs['<StepName>.<VarName>'] (whereas reading from another job in the same stage uses dependencies.<JobName>.outputs[...]). Mixing those two prefixes up evaluates to an empty string!

- Cache immutability & size limits: Azure DevOps Cache@2 entries are immutable once written and expire after 7 days of inactivity. If your cache archive exceeds ~2 GB (such as caching all of node_modules), uploading/downloading the tarball takes longer than running npm ci directly—always cache ~/.npm instead of node_modules.

- Hosted parallel job limits: Free Azure DevOps public projects get 10 parallel jobs, while private organizations start with 1 free Microsoft-hosted parallel job out of the box. Additional parallel jobs require purchasing extra hosted runners or adding self-hosted agents.

## 5. Hands-On Checklist
- [x] Enabled <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile> in .csproj files and committed packages.lock.json to Git.

- [x] Configured Cache@2 for both NuGet ($(Pipeline.Workspace)/.nuget/packages) and npm ($(Pipeline.Workspace)/.npm) and verified Cache hit logs on the second run.

- [x] Added --no-restore and --no-build flags across dotnet CLI commands and fetchDepth: 1 on build checkouts.

- [x] Split .NET and Angular builds into parallel jobs gated by a git diff path-detection stage so backend-only commits skip frontend builds completely.

- [ ] Up next in 17: Lock down enterprise access with Governance, RBAC & Compliance—configuring Azure DevOps security groups, Workload Identity Federation (OIDC) least-privilege service principals, pipeline resource permissions, and audit logs.
