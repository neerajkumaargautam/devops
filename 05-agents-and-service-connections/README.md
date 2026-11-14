# 05 - Agents & Service Connections

Got out of the code sandbox today to focus on the infrastructure side of things. Before our pipelines can do anything useful (like deployment), we need two things: something to run the code (Agents) and permission to access our cloud (Service Connections). 

I spent today sorting out when to use Microsoft’s compute vs. self-hosting, and how to safely link Azure DevOps to our Azure subscription without hardcoding secrets.

---

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/5ba3ee84-229b-4f79-a50c-c226d0c88f0c" />


## 1. Where Does the Pipeline Actually Run? (The Agent Game)
Every time a pipeline is triggered, it requests a machine (an "Agent") from Azure DevOps to execute our YAML steps. We have two choices:

### Microsoft-Hosted Agents (Easy mode)
*   **What it is:** You ask for a specific OS (like `ubuntu-latest` or `windows-latest`) using the `pool` tag, and Microsoft spins up an on-demand, virtual machine.
*   **The behavior:** They are ephemeral (temporary). After your build runs, the VM is completely destroyed. 
*   **Pros:** Zero maintenance. Pre-loaded with everything we need (`dotnet`, Git, Docker, Node, etc.). High-performing for standard builds.
*   **Cons:** You can't cache files locally (though you can use pipeline caching task configurations to offset this). They have no access to secure, private on-premise networks.

### Self-Hosted Agents (Control mode)
*   **What it is:** You install a lightweight agent service on your own physical VM, container, or local machine.
*   **The behavior:** Persistence. Because the agent stays alive across jobs, dependencies (like `nuget` or `npm` caches) remain local, which speeds up builds.
*   **Pros:** Access to your private networks (ideal if you need to run migrations on a firewalled Azure SQL Database or deploy directly to a private Kubernetes cluster). Cheaper for continuous build cycles.
*   **Cons:** You are now the "Ops" person. You have to patch the OS, upgrade SDKs, and maintain the disk space yourself.

*(Tip: In a hybrid team, we usually run standard CI builds on Microsoft-hosted agents, but deploy to on-premise or tightly firewalled Azure environments using self-hosted agents.)*

---

## 2. Connect Your Build to the Cloud (Service Connections)
How does our pipeline get permission to deploy a Web API to Azure without us sticking passwords in the source control? 

A **Service Connection** is the bridge. It securely authenticates our build agent with an external cloud (or other tool, like Docker Hub or Kubernetes) so that tasks can perform secure operations.

```text
[ Azure Pipeline YAML ]
         │ (References connection "Azure-Subscription-SvcConn")
         ▼
[ Service Connection ] ── (Azure DevOps encrypted secret / OIDC OAuth) ──► [ Azure Cloud RBAC ]

```

### Authentication Under the Hood: Service Principals & OIDC
When setting up an Azure Resource Manager connection, you used to create an App Registration and a client secret (a secret password that would eventually expire, meaning breaking pipelines once a year).

Today, the modern, standard method is Workload Identity Federation (OIDC):

It does not require a password/secret at all.

Instead, Azure DevOps and Azure establish trust using OAuth and temporary identity tokens. It's much safer and requires zero maintenance or credential rotation.


## 3. Configuring and Testing in My Lab
To test both of these out today, I set up the foundation for future deployment:

#### Part A: Registering a Self-Hosted Linux Agent (Optional lab testing)
If you want to test self-hosted workflows, you can set one up on an Azure VM or even local Windows Subsystem for Linux (WSL):

In Azure DevOps, go to Project Settings → Agent pools.

Select Default → New Agent and select your platform (Linux/macOS/Windows). Download the bundle.

Generate a Personal Access Token (PAT) in AzDO with Agent Pools (Read & Manage) scopes.

Run ./config.sh on the machine, input the URL (https://dev.azure.com/<your-org>), provide your PAT, and execute ./run.sh to watch it go active.

(Tip: You can target this private VM in your YAML by pointing your pool name: to 'Default' or whatever custom Pool name you registered.)

#### Part B: Connecting to Azure (OIDC)
I created my secure pipeline-to-Azure connection:

Navigated to Project Settings → Service connections → New Service connection.

Selected Azure Resource Manager and chose the recommended Workload Identity Federation (automatic).

Set the Scope level to Subscription and logged in to authorize with my Azure account.

Azure DevOps automatically generated the trust settings under the hood. (It also created a corresponding Service Principal in Azure with Contributor access inside my designated subscription).


## 4. Referencing a Connection in Your YAML
Now, when I write my deploy script, I can use official Azure DevOps tasks and point them to my target Azure cloud using only that connection's descriptive name:

```
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest' # Microsoft-hosted VM for the build

steps:
  # Verify connection with an Azure CLI task
  - task: AzureCLI@2
    displayName: 'Acknowledge Azure Access'
    inputs:
      azureSubscription: 'Azure-Subscription-SvcConn' # The exact name of my service connection
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        az account show
        az group list --output table
```

## 5. Hands-On Checklist
- [x] Located Agent Pools in Project Settings to confirm Microsoft-hosted pool limits (Azure Pipelines contains 1,800 free minutes for private repositories).

- [x] Configured an Azure Resource Manager service connection named Azure-Subscription-SvcConn using OIDC.

- [x] Added a simple AzureCLI@2 inline task to verify connection access to the Azure subscription.

- [x] Verified the pipeline was prompted with "Permit" dialogs on the first run (Azure DevOps branch security and permission gates working).
