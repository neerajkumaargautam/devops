# 01 - Azure DevOps Fundamentals (Explained Simply)

## 1. What is Azure DevOps? (The "Restaurant Kitchen" Analogy)
Imagine you are running a busy restaurant:
* Without a system, chefs bump into each other, orders get lost on sticky notes, and food goes out raw or cold.
* **Azure DevOps** is the complete digital management system for your software kitchen. It takes an idea from a customer's order all the way to a finished dish served safely on their table—without manual chaos.

In technical terms, it is a single cloud platform by Microsoft that brings **Project Managers, Developers (.NET/Frontend), QA Testers, and Cloud Admins** together in one place.

---

## 2. How Azure DevOps is Organized (The Hierarchy)
Before touching any code, you need to understand the 3-level folder structure inside Azure DevOps:

1. **Organization (Your Company / Account):** The top-level account (e.g., `dev.azure.com/neeraj-tech`). Think of this as your entire **Company Building**.
2. **Project (Your Product / App):** Inside the organization, you create projects (e.g., `ECommerce-Platform`). Think of this as a specific **Department or Floor** inside that building.
3. **Teams & Services:** Inside each project, you get 5 core tools (services) to build and ship your app.

---

## 3. The 5 Core Services of Azure DevOps
Every project in Azure DevOps gives you these five building blocks:

| Service | Layman Analogy | What It Actually Does for a .NET Developer |
| :--- | :--- | :--- |
| **1. Azure Boards** | **The Digital Sticky-Note Wall** | Tracks what needs to be built (User Stories, Tasks, Bugs) using Agile/Scrum boards. |
| **2. Azure Repos** | **The Time-Machine Vault** | Private Git repositories where your C# and Angular/React code lives safely with full history and code reviews. |
| **3. Azure Pipelines** | **The Automated Assembly Line** | Automatically compiles (`dotnet build`), tests (`dotnet test`), and deploys your app to Azure whenever you push code
