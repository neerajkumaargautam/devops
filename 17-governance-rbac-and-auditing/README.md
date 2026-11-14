# 17 - Governance, RBAC & Compliance

When you're the only developer on a sandbox project, making yourself Project Collection Administrator with subscription-wide `Owner` rights doesn't hurt anyone. In an enterprise organization, though, one overly permissive pipeline or a leaked client secret can wipe out production infrastructure or expose customer data.

Today I locked down the entire Azure DevOps and Azure RBAC surface across four layers: **Security Groups**, **secretless Workload Identity Federation (OIDC)** with least-privilege scoping, **Pipeline Resource Permissions & Checks**, and **Audit Log streaming**.

---
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/d56ee05b-5f1c-42ad-9542-004c5e481735" />


## 1. Azure DevOps Security Groups & Least-Privilege RBAC
Never assign permissions to individual users directly—always map Microsoft Entra ID (Azure AD) groups into Azure DevOps **Security Groups** (`Project Settings → Permissions`) so onboarding and offboarding happen automatically:

- **Readers:** Stakeholders, QA observers, or auditors who need to view boards, pipeline runs, and test results without modifying code or triggering releases.
- **Contributors:** Feature developers who can push to feature branches, open Pull Requests, and trigger CI builds—*but cannot* bypass branch policies, edit production Environments, or modify Service Connections.
- **Release Managers (Custom Group):** A custom security group granted **User** or **Administrator** permissions specifically on `prod-env` and release pipelines so only authorized leads can approve Production rollouts.
- **Project Administrators:** Strictly limited to 2–3 platform/tech leads for break-glass configuration changes.

*(Tip: Always apply **Deny** overrides carefully in Azure DevOps—unlike Azure RBAC where Deny is explicit at policy level, in Azure DevOps an explicit `Deny` on a group overrides an `Allow` from every other group a user belongs to!)*

---

## 2. Workload Identity Federation (OIDC) & Least-Privilege Service Principals
Legacy Azure Resource Manager service connections generated a 2-year **Client Secret** inside Entra ID. If that secret expired on a Friday night, every deployment broke; if someone extracted it, they could authenticate from outside Azure DevOps.

With **Workload Identity Federation (OIDC)**:
1. **Zero stored secrets:** Azure DevOps acts as an OIDC identity provider and exchanges a short-lived, single-run JWT token with Microsoft Entra ID at runtime. Nothing ever expires, and there is no secret to rotate.
2. **Scope to Resource Group—not Subscription:** When creating the Service Connection (`Project Settings → Service connections → Azure Resource Manager → Workload Identity federation (automatic)`), **always select a specific Resource Group** (e.g., `rg-dotnet-prod`) instead of leaving Resource Group blank. Leaving it blank grants `Contributor` across your *entire* Azure Subscription!
3. **Separate Service Connections per Environment:** Create `Azure-Dev-SvcConn` scoped strictly to `rg-dotnet-dev` and `Azure-Prod-SvcConn` scoped strictly to `rg-dotnet-prod` so a dev pipeline can never accidentally touch production resources.

### Custom Least-Privilege Azure Role (Even Tighter Than `Contributor`)
If your CD pipeline only deploys web apps and swaps slots (rather than creating brand-new infrastructure), strip `Contributor` down to a custom Entra ID role with only the exact actions needed:

```json
{
  "Name": "AzDO WebApp Slot Deployer",
  "IsCustom": true,
  "Description": "Allows deploying code and swapping slots on Azure App Service without deleting resources.",
  "Actions": [
    "Microsoft.Web/sites/Read",
    "Microsoft.Web/sites/config/Write",
    "Microsoft.Web/sites/publishxml/Action",
    "Microsoft.Web/sites/slots/Read",
    "Microsoft.Web/sites/slots/config/Write",
    "Microsoft.Web/sites/slots/slotsswap/Action",
    "Microsoft.Web/sites/slotsswap/Action"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/rg-dotnet-prod"
  ]
}
```

## 3. Locking Down Protected Resources (Service Connections, Variable Groups & Environments)
Even if a developer edits azure-pipelines.yml on a feature branch and tries to point it at Azure-Prod-SvcConn or vg-dotnet-prod, Azure DevOps blocks it if you lock down Protected Resources:

- Disable "Grant access permission to all pipelines": Open your Production Service Connection (Azure-Prod-SvcConn) and Variable Group (vg-dotnet-prod) → ⋮ → Security → uncheck Open access. Explicitly add only your official DotNet-Prod-Release pipeline under Pipeline permissions.

- Enforce Branch Control & Required Templates on prod-env:

   Go to Pipelines → Environments → prod-env → ⋮ → Approvals and checks.

   Add Branch control: set Allowed branches to refs/heads/main (blocks any feature/hotfix branch from deploying directly to Prod).

   Add Required template: enforce that any pipeline targeting prod-env must extend your hardened templates/prod-release-guardrail.yml template so no developer can delete SonarCloud or Trivy checks from their YAML!

## 4. Compliance & Audit Log Streaming (Organization Settings → Auditing)
For SOC 2, ISO 27001, or internal enterprise compliance, you need an immutable trail of who changed a branch policy, who approved a production gate, and who accessed a secret Variable Group:

- Navigate to Organization Settings → Policies and ensure Log Audit Events is toggled ON.

- Inspect events under Organization Settings → Auditing (tracks Git.BranchPolicyUpdated, Library.VariableGroupModified, Release.ApprovalCompleted, Security.PermissionChanged, etc.).

- Stream to Azure Monitor / Log Analytics: Click Streams → New stream → Azure Monitor Logs, enter your Log Analytics Workspace ID and Shared Key. Because Azure DevOps retains audit logs natively for only 90 days, streaming them to Log Analytics lets you retain logs for 1–7 years and trigger real-time Sentinel/Monitor alerts if anyone disables a branch policy or grants themselves Admin permissions.

## 5. Gotchas I Ran Into
- "Open access" trap on new Variable Groups: When creating a new Variable Group in Pipelines → Library, Azure DevOps leaves "Allow access to all pipelines" unchecked by default—except if you click "Authorize resources" from a PR branch run, which permanently grants that pipeline access! Always verify Security → Pipeline permissions on production Variable Groups and Service Connections.

- OIDC Subject Mismatch when renaming projects or pipelines: Under the hood, Workload Identity Federation binds the Entra ID Federated Credential's subject identifier to your exact Azure DevOps Organization, Project name, and Service Connection name (sc://<Org>/<Project>/<SvcConnName>). If you rename the Project or Service Connection later, authentication will fail with AADSTS700213 until you update the federated credential subject in Entra ID.

- Auditing is Org-level, not Project-level: You won't find the Auditing tab inside Project Settings—it lives at the top-level Organization Settings and requires Project Collection Administrator privileges to configure streams.

## 6. Hands-On Checklist
- [x] Mapped Entra ID groups into Azure DevOps Readers, Contributors, and a custom Release Managers security group with least-privilege permissions.

- [x] Migrated all Azure Resource Manager Service Connections to Workload Identity Federation (OIDC)—eliminating client secrets—and scoped separate connections to rg-dotnet-dev and rg-dotnet-prod.

- [x] Restricted Pipeline permissions on Azure-Prod-SvcConn and vg-dotnet-prod (disabled Open Access) and enforced refs/heads/main Branch control + Required template checks on prod-env.

- [x] Enabled Organization Auditing and configured an Azure Monitor Logs stream for long-term compliance retention and policy-tampering alerts.

- [ ] Up next in 18 (Capstone): Bring all 17 modules together into the Full-Stack .NET Enterprise Pipeline—orchestrating Bicep infrastructure, SonarCloud/Trivy DevSecOps, cached parallel Angular + .NET builds, ACR containers, EF Core migration bundles, and Zero-Downtime multi-stage releases with Approval Gates!
