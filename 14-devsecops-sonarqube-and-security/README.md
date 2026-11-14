# 14 - DevSecOps & Code Quality (Shift-Left Security)

Up until now, our pipelines only checked whether the C# code compiled and passed unit tests. Today I integrated **DevSecOps** ("shifting security left") directly into the CI pipeline—so leaked API keys, vulnerable NuGet packages, messy code smells, and critical OS/container CVEs get caught automatically on Pull Requests *before* anything ever reaches ACR or Production.

---
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/c9751b5f-0b75-49a3-9b7f-974501f1cb2b" />


## 1. The 4 Layers of Shift-Left Security in .NET Pipelines
Instead of waiting for a quarterly security audit, we embed four fast, automated checks right inside CI:

1. **Secret & Credential Scanning (Gitleaks / CredScan):** Scans the Git commit diff for accidentally committed Azure storage keys, JWT secrets, or SQL passwords before the build even compiles.
2. **Software Composition Analysis (`dotnet list package --vulnerable`):** Checks every direct and transitive NuGet package in your `.sln` against the GitHub Advisory Database for known CVEs—no third-party license needed.
3. **Static Application Security Testing & Quality Gates (SonarQube / SonarCloud):** Analyzes C# Roslyn rules during `dotnet build` to flag SQL injection risks, hardcoded credentials, cognitive complexity, and drops in unit test coverage.
4. **Container Image Scanning (Trivy):** Scans the built Docker image (`dotnet-webapi:$(Build.BuildId)`) for OS-level (`apt`/Debian) and .NET runtime vulnerabilities *before* pushing it to Azure Container Registry.

---

## 2. Setting Up SonarCloud / SonarQube for C#
Unlike JavaScript or Python where Sonar scans raw text files after build, C# analysis hooks directly into MSBuild/Roslyn:

1. Install the **SonarCloud** (or **SonarQube**) extension from the Azure DevOps Marketplace into your organization.
2. Create a Service Connection (**Project Settings → Service connections → SonarCloud**) using a generated token from your Sonar account.
3. Wrap your `dotnet build` and `dotnet test` steps between **`SonarCloudPrepare@2`**, **`SonarCloudAnalyze@2`**, and **`SonarCloudPublish@2`** (which waits for the Quality Gate verdict so a `FAILED` gate breaks the build!).

---

## 3. Complete DevSecOps CI Pipeline (`azure-pipelines-devsecops.yml`)
Notice how each security layer fails fast if a `CRITICAL` or `HIGH` issue is detected:

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  imageName: 'dotnet-webapi:$(Build.BuildId)'

steps:
  - checkout: self
    fetchDepth: 0 # Full git history required for accurate Sonar blame & Gitleaks diff

  - task: UseDotNet@2
    displayName: 'Use .NET 8 SDK'
    inputs:
      packageType: 'sdk'
      version: '8.x'

  # ==========================================
  # 1. SECRET SCANNING (Gitleaks)
  # ==========================================
  - bash: |
      docker run --rm -v $(Build.SourcesDirectory):/repo zricethezav/gitleaks:latest \
        detect --source="/repo" --verbose --redact
    displayName: '1. Scan Repo for Leaked Secrets (Gitleaks)'

  # ==========================================
  # 2. NUGET VULNERABILITY AUDIT
  # ==========================================
  - bash: |
      dotnet restore
      # Fails the pipeline if any Critical or High severity NuGet vulnerability exists
      VULN_OUTPUT=$(dotnet list package --vulnerable --include-transitive)
      echo "$VULN_OUTPUT"
      if echo "$VULN_OUTPUT" | grep -E "Critical|High"; then
        echo "##vso[task.logissue type=error]High or Critical NuGet vulnerabilities found!"
        exit 1
      fi
    displayName: '2. Audit NuGet Packages for Known CVEs'

  # ==========================================
  # 3. SONARCLOUD SAST & CODE COVERAGE GATE
  # ==========================================
  - task: SonarCloudPrepare@2
    displayName: '3a. Prepare SonarCloud C# Analysis'
    inputs:
      SonarCloud: 'SonarCloud-SvcConn'
      organization: 'neerajkumaargautam'
      scannerMode: 'MSBuild'
      projectKey: 'neerajkumaargautam_dotnet-fullstack-demo'
      projectName: 'DotNet-FullStack-Demo'
      extraProperties: |
        sonar.cs.opencover.reportsPaths=$(Agent.TempDirectory)/**/coverage.opencover.xml

  - task: DotNetCoreCLI@2
    displayName: '3b. Build Solution'
    inputs:
      command: 'build'
      projects: '**/*.sln'
      arguments: '--configuration $(buildConfiguration) --no-restore'

  - task: DotNetCoreCLI@2
    displayName: '3c. Run Unit Tests with OpenCover'
    inputs:
      command: 'test'
      projects: '**/*Tests/*.csproj'
      arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage;Format=opencover"'
      publishTestResults: true

  - task: SonarCloudAnalyze@2
    displayName: '3d. Run SonarCloud Code Analysis'

  - task: SonarCloudPublish@2
    displayName: '3e. Publish & Enforce Sonar Quality Gate'
    inputs:
      pollingTimeoutSec: '300'

  # ==========================================
  # 4. CONTAINER IMAGE CVE SCAN (Trivy)
  # ==========================================
  - bash: |
      docker build -t $(imageName) .
      # Scan local image BEFORE pushing to ACR; fail on unfixed CRITICAL/HIGH CVEs
      docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
        aquasec/trivy:latest image \
        --severity CRITICAL,HIGH \
        --ignore-unfixed \
        --exit-code 1 \
        $(imageName)
    displayName: '4. Trivy Container Vulnerability Scan'
```

## 4. Gotchas I Ran Into
- fetchDepth: 0 for SonarCloud & Gitleaks: By default, Azure Pipelines performs a shallow clone (fetchDepth: 1) to save time. SonarCloud needs commit history to distinguish "New Code" on a PR from legacy code, and Gitleaks needs commit diffs—always set fetchDepth: 0 on - checkout: self.

- SonarCloud Quality Gate doesn't fail the build automatically unless configured: SonarCloudPublish@2 posts the Quality Gate badge to the pipeline summary, but if you want a red Quality Gate to actually block a Pull Request merge, go to Repos → Branches → main → Branch policies → Status Checks and add SonarCloud/quality gate as a Required check!

- Trivy --ignore-unfixed flag: Base Linux images occasionally report low-level OS advisories that have no upstream patch yet. Passing --ignore-unfixed prevents false-positive pipeline outages on unpatchable OS packages while still blocking every exploitable CVE that has a fix available.

- Microsoft Defender for Cloud / CredScan: If your organization uses Microsoft Security DevOps (MicrosoftSecurityDevOps@1), you can run CredScan, Trivy, Terrascan, and Roslyn analyzers in a single unified task that uploads SARIF security alerts directly into the Azure DevOps Advanced Security tab.

5. Hands-On Checklist
- [x] Added Gitleaks secret detection to the CI pipeline and verified a dummy connection string triggers an immediate build failure with --redact hiding the secret from logs.

- [x] Wired dotnet list package --vulnerable --include-transitive to catch vulnerable NuGet dependencies automatically.

- [x] Integrated SonarCloud (Prepare → Build/Test → Analyze → Publish) with OpenCover code coverage and attached SonarCloud/quality gate as a required PR status check on main.

- [x] Added Trivy image scanning (--severity CRITICAL,HIGH --exit-code 1) between docker build and docker push so vulnerable containers never enter ACR.

- [ ] Up next in 15: Automate Database CI/CD (EF Core & Azure SQL)—building self-contained EF Core Migration Bundles and DACPAC deployments safely inside release pipelines.
