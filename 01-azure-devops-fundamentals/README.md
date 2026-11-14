# 01 - Azure DevOps Fundamentals

## 1. What is Azure DevOps? (The "Restaurant Kitchen" Analogy)
Imagine you are running a busy restaurant:
* Without a system, chefs bump into each other, orders get lost on sticky notes, and food goes out raw or cold.
* **Azure DevOps** is the complete digital management system for your software kitchen. It takes an idea from a customer's order all the way to a finished dish served safely on their table—without manual chaos.

In technical terms, it is a single cloud platform by Microsoft that brings **Project Managers, Developers (.NET/Frontend), QA Testers, and Cloud Admins** together in one place.

---

## 2. How Azure DevOps is Organized

<img width="2752" height="1536" alt="Gemini_Generated_Image_8m0utd8m0utd8m0u" src="https://github.com/user-attachments/assets/a5bb5c02-a715-418a-940a-5b0e30a5c86e" />


Before touching any code, understand the 3-level folder structure inside Azure DevOps:
1. **Organization (Your Company / Account):** The top-level account (e.g., `dev.azure.com/neeraj-tech`). Think of this as your entire **Company Building**.
2. **Project (Your Product / App):** Inside the organization, you create projects (e.g., `ECommerce-Platform`). Think of this as a specific **Department or Floor** inside that building.
3. **5 Core Services:** Inside each project, you get 5 tools to plan, code, build, test, and ship your app:

| Service | Layman Analogy | What It Actually Does for a .NET Developer |
| :--- | :--- | :--- |
| **1. Azure Boards** | **The Digital Sticky-Note Wall** | Tracks what needs to be built (User Stories, Tasks, Bugs) using Agile/Scrum boards. |
| **2. Azure Repos** | **The Time-Machine Vault** | Private Git repositories where your C# and Angular/React code lives safely with full history and code reviews. |
| **3. Azure Pipelines** | **The Automated Assembly Line** | Automatically compiles (`dotnet build`), tests (`dotnet test`), and deploys your app to Azure whenever you push code. |
| **4. Azure Artifacts** | **The Private Spare-Parts Store** | Stores your company's internal NuGet and npm packages so multiple .NET projects can reuse shared code easily. |
| **5. Azure Test Plans** | **The Quality Inspection Checklist** | Helps QA engineers run manual and exploratory tests to ensure nothing is broken before customers see it. |

---

## 3. Key DevOps Terms in Plain English

<img width="2752" height="1536" alt="Gemini_Generated_Image_8m0utd8m0utd8m0u (1)" src="https://github.com/user-attachments/assets/c267f4ab-0e91-48ac-bd3c-1afa7a6ebc66" />


### CI vs. CD (The Heart of Automation)
* **CI (Continuous Integration) — *"Check my work immediately"***
  * **Old way:** You write code for 2 weeks, merge it on Friday, and everything breaks ("Integration Hell").
  * **CI way:** Every time you push code to Azure Repos, an automated robot wakes up, runs `dotnet build` and `dotnet test`, and tells you within 2 minutes if you broke anything.
* **CD (Continuous Delivery / Deployment) — *"Ship it safely"***
  * **Continuous Delivery:** Your code is automatically packaged and deployed to a **Test/Staging** environment, waiting for a manager to click **"Approve"** before going live to real users.
  * **Continuous Deployment:** No human button click—if all automated tests pass, the code goes straight to **Production** automatically.

### Environments (The Safety Stages)
You never test new code directly on live customers. Code moves step-by-step through isolated environments:
1. **Local:** Your own laptop (Visual Studio / VS Code).
2. **Dev:** Shared sandbox where developers combine their code.
3. **QA / Staging:** An exact clone of the live site where testers verify everything.
4. **Production (Prod):** The real live application that end-users access.

### Process Templates (How Your Team Plans Work)
When creating a Project in Azure DevOps, it asks you to pick a template for **Azure Boards**:
* **Basic:** Simple 3-column board (*To Do → Doing → Done*). Best for solo projects or small teams.
* **Agile:** Uses *User Stories*, *Tasks*, and *Bugs*. Most common in .NET enterprise teams.
* **Scrum:** Uses *Product Backlog Items (PBIs)* and strict 2-week *Sprints*.

---

## 4. How Companies Measure Success (DORA Metrics)

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/6bad3a1a-9336-404e-918d-b3d788760082" />


Managers and DevOps architects use 4 simple numbers (called **DORA Metrics**) to check if their DevOps setup is healthy:

1. **Deployment Frequency:** *How often do we release updates?* (Good teams deploy daily or weekly, not once every 6 months).
2. **Lead Time for Changes:** *How fast does a code fix reach live users?* (Measured in hours, not weeks).
3. **Change Failure Rate:** *How often do our deployments crash the app?* (Lower is better—ideally under 15%).
4. **Mean Time to Recovery (MTTR):** *If the live site goes down, how fast can we fix or roll it back?* (Good teams recover in minutes).

---

## 5. Hands-On Checklist for Module 01
Before moving to Module 02, complete these 3 quick setup steps:
- [ ] Go to [dev.azure.com](https://dev.azure.com) and sign in with your Microsoft/GitHub account (Free tier gives you unlimited private Git repos and 1,800 free pipeline minutes/month).
- [ ] Create your first **Organization** (e.g., `neeraj-devops-lab`).
- [ ] Create a new **Project** named `DotNet-DevOps-Demo` using the **Agile** process template and explore the left-hand menu (**Boards, Repos, Pipelines, Test Plans, Artifacts**).
