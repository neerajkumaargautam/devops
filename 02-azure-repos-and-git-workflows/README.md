# 02 - Azure Repos & Git Workflows

Moved into Azure Repos today. As a .NET dev, I already use Git daily in Visual Studio and VS Code, so I skipped the basic `git add/commit` stuff and focused on **how teams actually lock down repositories** in Azure DevOps so nobody accidentally pushes broken C# code directly to `main`.

Here are my notes on branching strategies, Branch Policies (the real power of Azure Repos), and merge types.

---

## 1. Branching Strategies: What Actually Works?
Before turning on rules, the team has to agree on how branches live and die. In .NET shops, it usually comes down to two approaches:

<img width="2752" height="1536" alt="Gemini_Generated_Image_8rpt5o8rpt5o8rpt" src="https://github.com/user-attachments/assets/d82e98ab-bb35-4afc-a3fb-9209c30a803c" />


*   **Trunk-Based Development (Modern / CI-CD friendly):**
    *   Everyone branches off `main` (`feature/login-api`, `fix/null-ref-bug`), keeps the branch alive for **1–2 days max**, and merges right back into `main` via a Pull Request.
    *   *Why I prefer this:* No painful week-long merge conflicts in `.csproj` or `appsettings.json` files, and it plays much nicer with automated CI/CD pipelines.
*   **GitFlow (Heavy / Release-based):**
    *   Uses long-lived `main` and `develop` branches, plus `release/*` and `hotfix/*` branches.
    *   *When it makes sense:* Only if you maintain multiple versions of an installed product simultaneously. For web APIs and SPAs deployed continuously to Azure, it usually just slows everyone down.

---

## 2. Branch Policies (Locking Down `main`)
By default, anyone with Contributor access in Azure Repos can run `git push origin main` and overwrite history. **Branch Policies** disable direct pushes and force every change through a Pull Request (PR) checklist.

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/6bd2a4e7-1321-43bd-a0b6-41e485bf855e" />


To configure this in AzDO: **Repos → Branches → Click `...` on `main` → Branch policies**. These are the 4 gates I turned on:

1.  **Require a minimum number of reviewers:** Set to at least `1` (or `2` for critical prod repos). Also turned off *"Allow requestors to approve their own changes"* so nobody rubber-stamps their own PR.
2.  **Check for linked work items:** Forces the dev to link an Azure Boards User Story or Bug ID to the PR. Huge help later when you're trying to figure out *why* a certain line of code was changed six months ago.
3.  **Check for comment resolution:** Blocks the PR from completing if a reviewer left a comment that hasn't been marked "Resolved" or "Won't fix".
4.  **Build Validation (The most important one):** Automatically triggers an Azure Pipeline (`dotnet build` + `dotnet test`) whenever a PR is opened or updated. If the build fails or a unit test breaks, the **Complete** button stays greyed out.

---

## 3. Merge Strategies in Azure DevOps
When clicking **Complete** on a PR in Azure Repos, it asks you to pick a merge type. Picking the wrong one turns your Git graph into spaghetti:

*   **Squash Commit (My go-to for feature branches):** Takes all 15 messy commits from my feature branch (`"wip"`, `"fix typo"`, `"please work"`) and squashes them into **one clean commit** on `main` linked to the PR. Keeps `main` super clean and makes reverting a bad feature painless.
*   **Merge (no fast-forward):** Preserves every single commit from the feature branch and adds a merge commit on top. Useful if commit-by-commit granularity actually matters.
*   **Rebase and fast-forward:** Replays your commits on top of `main` for a completely linear history, without creating an extra merge bubble.

---

## 4. Quick `.gitignore` Gotcha for .NET + Angular Repos
When initializing a repo directly in Azure Repos, always select the **VisualStudio** `.gitignore` template (and append `node_modules/` and `dist/` if your frontend SPA lives in the same monorepo). Pushing `bin/`, `obj/`, or `.vs/` folders into Git is the fastest way to bloat a repo.

---

## 5. Hands-On Lab Checklist
Tested everything inside my `dotnet-devops-practice` project today:

- [x] Created a new repo `DotNet-FullStack-Demo` in Azure Repos with a `VisualStudio` `.gitignore`.
- [x] Pushed a basic ASP.NET Core Web API solution (`src/` + `tests/`) from local VS Code using a Personal Access Token (PAT) / Git Credential Manager.
- [x] Enabled **Branch Policies** on `main` (1 reviewer required, linked work item required, comment resolution enforced).
- [x] Tested pushing directly to `main` from terminal — confirmed Azure Repos rejected it (`TF402455: Pushes to this branch are not permitted; you must use a pull request to update this branch`).
- [x] Created a `feature/health-endpoint` branch, opened a PR, linked a Boards task, and completed it using **Squash merge**.
- [ ] *Up next in 03:* Wire up an actual automated CI pipeline (`dotnet restore`, `build`, and `test`) so **Build Validation** runs automatically on every PR.
