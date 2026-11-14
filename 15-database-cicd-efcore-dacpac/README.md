# 15 - Database CI/CD (EF Core Migration Bundles & DACPAC)

Deploying stateless .NET Web API code is easy—you just swap a zip file or container image. Deploying **database schema changes** is where real production releases get nerve-wracking. Running `context.Database.Migrate()` on app startup breaks horribly when you have multiple container/slot instances booting simultaneously, and installing the full `.NET SDK` + source code on a CD release agent just to run `dotnet ef database update` is slow and messy.

Today I automated **Azure SQL schema deployments** properly inside Azure Pipelines using two standard .NET approaches:
1. **EF Core Migration Bundles (`efbundle`)** for Code-First teams.
2. **SQL Database Projects (`.dacpac`)** for State-Based / DBA-managed schemas.

---
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/c57686c9-b08f-4a03-8302-a2ca98a4e4d6" />

## 1. Migration-Based (EF Core) vs. State-Based (DACPAC)
Before writing YAML, it helps to understand which model fits your team:

- **EF Core Migration Bundles (Code-First / Sequential):**
  - You compile all your C# `Migrations/*.cs` files during CI into a **single, self-contained executable (`efbundle`)**.
  - On the CD agent, `efbundle` runs without needing the `.NET SDK`, `dotnet-ef` CLI tools, or source code—it checks `__EFMigrationsHistory` in Azure SQL and applies *only* the missing migrations in exact order.
- **SQL Database Project / DACPAC (State-Based / Declarative):**
  - You define the desired end-state of tables, views, and stored procedures in a `.sqlproj` project.
  - During CI, MSBuild compiles it into a `Database.dacpac` artifact. During CD, `SqlPackage` (`SqlAzureDacpacDeployment@1`) compares the `.dacpac` against the live Azure SQL database, generates a diff script on the fly, and brings the DB into alignment.

---

## 2. Option A: EF Core Migration Bundles (`efbundle`)
Introduced in EF Core 6+, `dotnet ef migrations bundle` solves every headache of running EF migrations in CI/CD pipelines.

#### Step 1: Build the Self-Contained Bundle in CI
Notice the `--self-contained` and `--target-runtime linux-x64` flags—this creates a standalone binary that runs on any `ubuntu-latest` CD agent with zero SDK dependencies:

```yaml
stages:
  - stage: BuildDatabaseBundle
    displayName: 'CI: Build EF Core Migration Bundle'
    jobs:
      - job: BuildEFBundle
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: 'sdk'
              version: '8.x'

          - bash: |
              dotnet tool install --global dotnet-ef --version 8.*
              dotnet ef migrations bundle \
                --project src/Infrastructure/Infrastructure.csproj \
                --startup-project src/WebApi/WebApi.csproj \
                --configuration Release \
                --self-contained \
                --target-runtime linux-x64 \
                --output $(Build.ArtifactStagingDirectory)/efbundle
            displayName: 'Compile Self-Contained efbundle'

          - publish: '$(Build.ArtifactStagingDirectory)'
            artifact: 'db-bundle'
```
#### Step 2: Execute the Bundle in CD (Before App Deployment!)
Always run the database migration job before deploying your new Web API code, and pass the connection string securely from your Key Vault Variable Group via environment variables so it never shows up in process arguments:

```
- stage: MigrateDatabaseProd
    displayName: 'CD: Apply EF Migrations to Azure SQL'
    dependsOn: BuildDatabaseBundle
    condition: succeeded()
    variables:
      - group: 'vg-dotnet-prod' # Contains Key Vault secret 'SqlAdminConnectionString'
    jobs:
      - deployment: RunEFMigrations
        environment: 'prod-db-env' # Attach DBA / Tech Lead Approval Gate here!
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - bash: |
                    chmod +x $(Pipeline.Workspace)/db-bundle/efbundle
                    $(Pipeline.Workspace)/db-bundle/efbundle --connection "$DB_CONN_STRING"
                  displayName: 'Execute Idempotent EF Core Bundle'
                  env:
                    DB_CONN_STRING: $(SqlAdminConnectionString)
```

(Tip: Always generate a dry-run SQL preview script in CI using dotnet ef migrations script --idempotent -o $(Build.ArtifactStagingDirectory)/migration-preview.sql and publish it as an artifact—DBAs love being able to read the exact SQL diff before approving prod-db-env!)

## 2. Option B: State-Based Deployments with .dacpac
If your team uses SQL Server Data Tools (SSDT) or Azure Data Studio SQL Projects (.sqlproj), use SqlAzureDacpacDeployment@1 on a Windows agent (windows-latest has SqlPackage.exe pre-installed):

```
- stage: DeployDacpacProd
    displayName: 'CD: Deploy Azure SQL DACPAC'
    jobs:
      - deployment: DeploySqlDacpac
        environment: 'prod-db-env'
        pool:
          vmImage: 'windows-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: SqlAzureDacpacDeployment@1
                  displayName: 'Publish Database.dacpac to Azure SQL'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    AuthenticationType: 'server'
                    ServerName: '$(SqlServerFqdn)'
                    DatabaseName: '$(SqlDatabaseName)'
                    SqlUsername: '$(SqlAdminUser)'
                    SqlPassword: '$(SqlAdminPassword)'
                    deployType: 'DacpacTask'
                    DeploymentAction: 'Publish'
                    DacpacFile: '$(Pipeline.Workspace)/dacpac-drop/AppDatabase.dacpac'
                    AdditionalArguments: '/p:BlockOnPossibleDataLoss=true'
                    IpDetectionMethod: 'AutoDetect'
```

## 4. Gotchas I Ran Into
- Azure SQL Firewall blocking hosted agents: Microsoft-hosted agents (ubuntu-latest / windows-latest) get a different dynamic public IP on every run. SqlAzureDacpacDeployment@1 has IpDetectionMethod: 'AutoDetect' built in to temporarily open and clean up a firewall rule—but if you run efbundle via bash on Linux, you either need "Allow Azure services and resources to access this server" enabled on Azure SQL, or a quick az sql server firewall-rule create / delete wrapper around the efbundle step.

- Least-privilege DB credentials: Your running ASP.NET Core Web API should connect to Azure SQL with db_datareader and db_datawriter permissions only—never db_owner / ddladmin. Keep the schema-altering ddladmin credentials strictly isolated inside the CI/CD migration stage!

- Zero-downtime Expand-and-Contract rule: Because your old v1 Web API is still serving requests for a few seconds while efbundle runs (or during a Blue-Green slot swap), never drop or rename a column in the same release. Release 1 adds the new nullable column (Expand), Release 2 switches the C# code to read/write the new column, and Release 3 drops the old column (Contract).

- /p:BlockOnPossibleDataLoss=true in DACPAC: Always keep BlockOnPossibleDataLoss=true enabled on Production .dacpac deployments so an accidental table refactor never drops live production data.

## 5. Hands-On Checklist
- [x] Added dotnet ef migrations bundle --self-contained --target-runtime linux-x64 to the CI pipeline alongside an idempotent migration-preview.sql artifact for reviewer inspection.

- [x] Created a dedicated prod-db-env Environment with a Manual Approval gate so schema changes require explicit sign-off before running.

- [x] Verified efbundle --connection "$DB_CONN_STRING" applies pending migrations cleanly and exits with 0 ("No migrations were applied. The database is already up to date.") when re-run.

- [x] Tested .dacpac publishing with /p:BlockOnPossibleDataLoss=true and IpDetectionMethod: 'AutoDetect' against an Azure SQL test database.

- [ ] Up next in 16: Speed up our growing pipeline with Pipeline Performance & Caching—wiring up NuGet & npm Cache@2 tasks, parallel jobs, and conditional execution to cut build times in half.
