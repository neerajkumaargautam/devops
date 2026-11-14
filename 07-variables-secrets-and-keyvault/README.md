# 07 - Variables, Secrets & Azure Key Vault

Once you start deploying a real ASP.NET Core API across Dev, QA, and Prod, you immediately hit the config problem: SQL connection strings, JWT signing keys, and third-party API secrets cannot live in `appsettings.json` or hardcoded inside `azure-pipelines.yml`.

Today I worked through the three ways Azure DevOps handles configuration and secrets—starting from basic YAML variables, moving to shared **Variable Groups**, and finally pulling live secrets directly from **Azure Key Vault**.

---

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/c40d7968-9199-4a06-98e2-b092c0533fb3" />


## 1. The 3 Levels of Config in Azure DevOps
Depending on whether a value is public or sensitive—and whether one pipeline or ten pipelines need it—you pick one of three spots:

1. **Inline YAML Variables:** Defined right at the top of `azure-pipelines.yml` (e.g., `buildConfiguration: 'Release'`). Great for non-sensitive build flags, terrible for secrets or values that change per environment.
2. **Pipeline UI Variables:** Added via the **Variables** button in the Azure DevOps pipeline editor. You can toggle the little padlock icon to mark a value as **Secret** so AzDO encrypts it and masks it with `***` in build logs.
3. **Variable Groups (Pipelines → Library):** Shared collections of variables that multiple pipelines can import. You can create one group per environment (`vg-dotnet-dev`, `vg-dotnet-prod`) and link them to **Azure Key Vault** so secrets stay centralized in Azure rather than duplicated inside DevOps.

---

## 2. Variable Syntax Gotcha (`$()`, `${{}}`, `$[]`)
Azure Pipelines has three different ways to read a variable, and mixing them up caused a few head-scratching errors until I wrote down when each one gets evaluated:

- **`${{ variables.myVar }}` (Compile-time):** Evaluated *before* the pipeline even starts running. Use this when passing parameters into YAML templates or choosing conditional blocks (`${{ if eq(variables['Build.SourceBranchName'], 'main') }}:`).
- **`$(myVar)` (Runtime macro):** Evaluated right before a task runs. This is what you use 90% of the time inside task inputs and scripts.
- **`$[variables.myVar]` (Runtime expression):** Evaluated dynamically during execution—required for things like `counter()` versioning or passing output variables from Job A into Job B.

*(Tip: Secret variables are NEVER automatically mapped into environment variables inside `script:` or `bash:` blocks for security reasons. If a bash/pwsh script needs a secret, you have to explicitly map it under `env:`.)*

---

## 3. Linking Azure Key Vault to a Variable Group
In production .NET teams, security admins usually manage secrets inside **Azure Key Vault**, and pipelines just read them at runtime:

1. In the Azure Portal, create a **Key Vault** (e.g., `kv-dotnet-lab-dev`) and add a secret—for example, `ConnectionStrings--DefaultConnection` (using double hyphens `--` instead of colons `:` because Key Vault secret names only allow alphanumeric characters and dashes, and .NET Core automatically translates `--` into `:` configuration hierarchy).
2. In Azure DevOps, go to **Pipelines → Library → + Variable group**.
3. Name it `vg-keyvault-dev`, toggle **Link secrets from an Azure key vault as variables** ON, select your `Azure-Subscription-SvcConn` service connection, pick `kv-dotnet-lab-dev`, and select the specific secrets to expose.

---

## 4. Replacing `appsettings.json` Values in YAML
Instead of writing messy regex scripts to edit `appsettings.json` before deploying, Azure DevOps has a built-in **`FileTransform@1`** task that automatically matches variable names to JSON keys (including nested keys using dot notation like `ConnectionStrings.DefaultConnection`):

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  # Import the Key Vault-backed Variable Group
  - group: 'vg-keyvault-dev'
  # Define regular inline variables alongside it
  - name: 'AspNetCoreEnvironment'
    value: 'Development'

steps:
  # 1. Transform appsettings.json using matching pipeline/Key Vault variables
  - task: FileTransform@1
    displayName: 'Transform appsettings.json'
    inputs:
      folderPath: '$(System.DefaultWorkingDirectory)/src/WebApi'
      fileType: 'json'
      targetFiles: '**/appsettings.json'

  # 2. Explicitly map a secret if a script step needs to read it
  - bash: |
      echo "Environment is $(AspNetCoreEnvironment)"
      echo "Secret length is ${#MY_MAPPED_SECRET}" # Never echo the raw secret!
    displayName: 'Verify Mapped Secret Safely'
    env:
      MY_MAPPED_SECRET: $(ApiSigningKey)
```

## 5. Gotchas I Hit Along the Way
Key Vault RBAC vs. Access Policies: If your Key Vault uses Azure RBAC (the newer default) instead of legacy Vault Access Policies, linking the Variable Group will throw a Forbidden error unless your Service Connection's Service Principal has the Key Vault Secrets User role assigned on that Key Vault.

Pipeline authorization prompt: The very first time a YAML file references a new group:, the pipeline pauses with "This pipeline needs permission to access a resource before this run can continue". You just click View → Permit once in the UI and subsequent runs flow automatically.

Double-dash -- vs. Dot . naming: Azure Key Vault doesn't allow dots . in secret names, whereas FileTransform@1 looks for dots (ConnectionStrings.DefaultConnection) to match JSON hierarchy. Either name your secret ConnectionStrings--DefaultConnection and let ASP.NET Core bind it directly from environment variables, or map it to a dotted pipeline variable in YAML.

## 6. Hands-On Checklist
- [x] Spun up a test Azure Key Vault (kv-dotnet-lab-dev) and added a dummy SQL connection string and JWT secret.

- [x] Assigned the Key Vault Secrets User RBAC role to my Azure-Subscription-SvcConn identity.

- [x] Created a vg-keyvault-dev Variable Group in Pipelines → Library and linked the Key Vault secrets.

- [x] Tested FileTransform@1 on a sample appsettings.json and confirmed the JSON keys updated cleanly during the build without leaking secrets in the logs.

- [ ] Up next in 08: Put everything together into a Multi-Stage CD Pipeline (Build → Dev → QA → Prod) using Environments and manual approval gates.
