# 06 - Classic vs. YAML CI Pipelines

Now that agents and service connections are sorted, I spent today digging into the actual pipeline anatomy in Azure DevOps. If you look at older .NET projects, you will still run into **Classic** pipelines built with drag-and-drop UI editors, whereas every modern setup uses **YAML** (`azure-pipelines.yml`) checked straight into Git.

Here are my notes on why YAML won, how the YAML schema hierarchy actually nests together, and how triggers work so pipelines don't fire unnecessarily on documentation edits.

---

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/97f5c409-8225-428b-af18-d636438e0abc" />


## 1. Classic UI vs. YAML (Pipeline-as-Code)
In the Classic model, Azure DevOps forces you to split your workflow into two separate places in the portal: **Pipelines (Builds)** for compiling/testing and **Releases** for deploying. With YAML, everything lives in a single file inside your repo.

- **Version control & PR reviews:** Because `azure-pipelines.yml` sits next to my `.sln` file, if I need to add a new `dotnet test` flag or upgrade to .NET 9, I can test it on a feature branch and get it reviewed in a Pull Request before it ever touches `main`. Classic UI edits happen live in the browser with no PR safety net.
- **Rollback safety:** If I roll back a bad code commit in Git, the pipeline definition automatically rolls back to match that exact commit's build steps.
- **Unified CI + CD:** Instead of gluing a Classic Build to a Classic Release, a single YAML file can define multi-stage pipelines (Build → Deploy Dev → Deploy Prod).
- **When Classic still shows up:** You'll mostly see it in legacy enterprise repos that haven't migrated yet, or when someone quickly prototypes a task using the Classic UI and clicks **"View YAML"** to export the snippet into their code. *(Note: On newer Azure DevOps organizations, Microsoft even disables Classic pipeline creation by default under Project Settings).*

---

## 2. The YAML Pipeline Hierarchy
One of the easiest ways to get indentation errors in Azure Pipelines is forgetting which block belongs inside which. Every YAML pipeline follows a strict top-down nesting order:

```text
Pipeline (azure-pipelines.yml)
 ├── Trigger / PR Rules (When does it start?)
 └── Stages (Major phases: Build, Deploy_Dev, Deploy_Prod)
      └── Jobs (A unit of work assigned to ONE build agent)
           └── Steps (Sequential commands running on that same agent)
                ├── Task (Pre-built AzDO extensions like DotNetCoreCLI@2)
                └── Script / Bash / Pwsh (Raw inline shell commands)
```
(Tip: For a simple single-stage CI pipeline, Azure DevOps lets you omit stages: and jobs: and start directly with steps:—AzDO automatically wraps them in a default stage and job behind the scenes. Once you add multiple environments, you have to write out the full stages: → jobs: → steps: hierarchy.)


## 3. Controlling Triggers (CI, PR, and Path Filters)
By default, pushing to any branch can kick off a build if you don't specify triggers. In a real .NET + Angular repo, you want tight control over what triggers a run—especially so updating a README.md doesn't waste build minutes.

```
# 1. CI Trigger: Runs AFTER code is merged/pushed to these branches
trigger:
  branches:
    include:
      - main
      - release/*
  paths:
    exclude:
      - '*.md'
      - 'docs/*'

# 2. PR Trigger: Runs BEFORE a Pull Request merges (if using GitHub; in Azure Repos, use Branch Policies!)
pr:
  branches:
    include:
      - main
  paths:
    exclude:
      - '*.md'

# 3. Scheduled Trigger (Optional): Nightly clean build at 2:00 AM UTC
schedules:
  - cron: '0 2 * * *'
    displayName: 'Nightly Full Build'
    branches:
      include:
        - main
    always: true
```
(Gotcha for Azure Repos users: Putting pr: inside azure-pipelines.yml works for GitHub repos, but Azure Repos ignores the pr: block in YAML. For Azure Repos, PR validation is strictly controlled via Repos → Branches → Branch Policies → Build Validation that I set up in Module 02.)

## 4. A Complete Multi-Job CI YAML Example (.NET API + Angular UI)
Since each job can spin up its own independent agent in parallel, splitting backend (dotnet) and frontend (ng build) into two parallel jobs cuts total CI time nearly in half:

```
trigger:
  branches:
    include:
      - main
  paths:
    exclude:
      - '**/*.md'

stages:
  - stage: BuildAndTest
    displayName: 'CI: Build & Test Full-Stack'
    jobs:
      # Job 1: Build & Test .NET Backend
      - job: BackendBuild
        displayName: '.NET 8 Web API'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: 'sdk'
              version: '8.x'

          - script: |
              dotnet restore src/WebApi/WebApi.csproj
              dotnet build src/WebApi/WebApi.csproj --configuration Release --no-restore
              dotnet test tests/WebApi.Tests/WebApi.Tests.csproj --configuration Release --no-build --logger trx
            displayName: 'Restore, Build & Test API'

          - task: PublishTestResults@2
            condition: succeededOrFailed() # Still publish test results even if a test fails!
            inputs:
              testResultsFormat: 'VSTest'
              testResultsFiles: '**/*.trx'

      # Job 2: Build Angular Frontend (Runs in parallel with Job 1)
      - job: FrontendBuild
        displayName: 'Angular SPA'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'

          - script: |
              cd src/ClientApp
              npm ci
              npm run build -- --configuration production
            displayName: 'Install & Build Angular'
```

## 5. Handy Tricks I Picked Up Today
- The YAML Task Assistant: When editing azure-pipelines.yml inside the Azure DevOps web editor, click Show assistant on the right side. You can pick any task (like DotNetCoreCLI@2), fill in the form fields, and click Add—it injects properly indented YAML right where your cursor is.

- condition: succeededOrFailed(): By default, if dotnet test fails, Azure DevOps immediately aborts the job and skips the remaining steps. Adding condition: succeededOrFailed() on PublishTestResults@2 ensures the failed test report still gets uploaded so you can actually see which assertion broke in the UI.

- Separate files between jobs: Because BackendBuild and FrontendBuild run on two different VMs, they don't share a local disk. If a later stage needs outputs from both, each job has to publish its own pipeline artifact.


## 6. Hands-On Checklist
- [x] Tested the YAML Task Assistant in the AzDO portal and verified indentation rules using the VS Code Azure Pipelines extension.

- [x] Added paths: exclude: ['**/*.md'] to azure-pipelines.yml and verified doc-only commits no longer trigger unnecessary builds.

- [x] Split my full-stack CI pipeline into two parallel jobs (BackendBuild and FrontendBuild) and verified both run simultaneously.

- [x] Added condition: succeededOrFailed() to test result publishing so failing unit tests still show up in the Tests tab.

- [ ] Up next in 07: Handle app settings, connection strings, and API keys properly using Variable Groups and Azure Key Vault instead of hardcoding values in YAML.
