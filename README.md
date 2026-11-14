# DevOps: Zero to Expert

A structured, hands-on repository covering Azure DevOps engineering from core foundations to enterprise-grade cloud architecture.

## Module Roadmap

### Phase 1: Foundations & Source Control (`01` – `05`)
| # | Module | Status | Key Topics (.NET & Azure DevOps Focus) |
| :--- | :--- | :--- | :--- |
| [01](./01-azure-devops-fundamentals/README.md) | Azure DevOps Fundamentals | 🚧 In Progress | Organizations, Projects, Azure Boards, Process Templates (Agile/Scrum), DORA metrics |
| [02](./02-azure-repos-and-git-workflows/README.md) | Azure Repos & Git Workflows | ⏳ Planned | Branching strategies (Trunk-Based vs. GitFlow), Branch Policies, PR Reviewers, Squash vs. Merge |
| [03](./03-dotnet-build-and-test-basics/README.md) | .NET Build & Test Automation | ⏳ Planned | `dotnet restore/build/test/publish`, Solution (`.sln`) handling, xUnit/NUnit test & code coverage reports |
| [04](./04-azure-artifacts-and-nuget/README.md) | Azure Artifacts & Package Feeds | ⏳ Planned | Private NuGet & npm feeds, Upstream sources, Semantic Versioning (`GitVersion`), publishing shared class libraries |
| [05](./05-agents-and-service-connections/README.md) | Agents & Service Connections | ⏳ Planned | Microsoft-hosted vs. Self-hosted agents (Windows/Linux), Agent Pools, Azure Resource Manager (ARM) Service Connections |

---

### Phase 2: CI/CD Pipelines & YAML Mastery (`06` – `10`)
| # | Module | Status | Key Topics (.NET & Azure DevOps Focus) |
| :--- | :--- | :--- | :--- |
| [06](./06-classic-vs-yaml-ci-pipelines/README.md) | CI Pipelines in YAML | ⏳ Planned | Pipeline anatomy (Triggers, Stages, Jobs, Steps), CI triggers, Path filters for monorepos (`.NET API` + `Frontend`) |
| [07](./07-variables-secrets-and-keyvault/README.md) | Variables, Secrets & Key Vault | ⏳ Planned | Pipeline variables, Variable Groups, Secure Files, linking **Azure Key Vault** secrets to pipelines |
| [08](./08-multi-stage-cd-and-environments/README.md) | Multi-Stage CD & Environments | ⏳ Planned | `Dev → QA → Staging → Prod` stages, Azure DevOps Environments, Manual Approval Gates, Business Hours checks |
| [09](./09-reusable-yaml-templates/README.md) | Reusable YAML Templates | ⏳ Planned | DRY pipelines using `extends`, Job/Step/Stage templates, Parameters, Central Template Repositories |
| [10](./10-deployment-strategies/README.md) | Zero-Downtime Deployments | ⏳ Planned | Azure App Service Deployment Slots (Swap), Blue-Green, Canary releases, and automated rollback triggers |

---

### Phase 3: Containers, IaC & DevSecOps on Azure (`11` – `15`)
| # | Module | Status | Key Topics (.NET & Azure DevOps Focus) |
| :--- | :--- | :--- | :--- |
| [11](./11-docker-and-acr-pipelines/README.md) | Docker & Azure Container Registry | ⏳ Planned | Multi-stage `Dockerfile` for ASP.NET Core, building & pushing tagged images to **ACR** via Azure Pipelines |
| [12](./12-deploying-to-aks-and-container-apps/README.md) | Deploying to AKS & Azure Container Apps | ⏳ Planned | Deploying containerized .NET microservices to **Azure Container Apps** and **AKS** (Kubernetes manifests & Helm) |
| [13](./13-iac-pipelines-bicep-terraform/README.md) | IaC Pipelines (Bicep & Terraform) | ⏳ Planned | Provisioning Azure resources (App Service, SQL, Key Vault) inside Azure Pipelines with `what-if` / `plan` approval gates |
| [14](./14-devsecops-sonarqube-and-security/README.md) | DevSecOps & Code Quality | ⏳ Planned | **SonarQube / SonarCloud** quality gates for C#, `dotnet list package --vulnerable`, Trivy container scanning, Credential Scanner |
| [15](./15-database-cicd-efcore-dacpac/README.md) | Database CI/CD (EF Core & SQL) | ⏳ Planned | Automating Azure SQL schema updates via **EF Core Migration Bundles** and **DACPAC** tasks in release pipelines |

---

### Phase 4: Enterprise Mastery & Capstone (`16` – `18`)
| # | Module | Status | Key Topics (.NET & Azure DevOps Focus) |
| :--- | :--- | :--- | :--- |
| [16](./16-pipeline-optimization-and-caching/README.md) | Pipeline Performance & Caching | ⏳ Planned | NuGet & npm `Cache@2` tasks, parallel jobs, conditional execution, minimizing build times |
| [17](./17-governance-rbac-and-auditing/README.md) | Governance, RBAC & Compliance | ⏳ Planned | Security groups, Least-privilege Service Principals (Workload Identity Federation / OIDC), Pipeline permissions & audit logs |
| [18](./18-capstone-dotnet-enterprise-cicd/README.md) | Capstone: Full-Stack .NET Enterprise Pipeline | ⏳ Planned | End-to-end production pipeline: **Angular + ASP.NET Core Web API + Azure SQL**, Bicep infra, SonarQube, Docker/ACR, multi-stage deployment with Approval Gates |

---

## Tech Stack Covered
* **Platform:** Azure DevOps Services (Boards, Repos, Pipelines, Artifacts, Test Plans)
* **Application Stack:** .NET (C#, ASP.NET Core Web API, EF Core) & SPA Frontend (Angular / TypeScript)
* **Azure Services:** App Service, Azure Container Registry (ACR), Azure Kubernetes Service (AKS), Azure Container Apps, Azure Key Vault, Azure SQL
* **IaC & Security:** Azure Bicep, Terraform, SonarQube, Trivy, Workload Identity Federation (OIDC)
