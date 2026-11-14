# 04 - Azure Artifacts & Private NuGet Feeds

In real-world .NET teams, we rarely dump everything into one giant solution. Common DTOs, auth middleware, logging wrappers, and shared models live in separate class libraries so multiple Web APIs can reuse them without copy-pasting code. Today I set up **Azure Artifacts** to pack and publish a shared C# library as a private NuGet package and consume it cleanly inside another project.

---

## 1. How Azure Artifacts Fits In
Think of an Azure Artifacts **Feed** as your private, internal NuGet.org. Even better, when you enable **Upstream Sources**, your private feed acts as a single proxy—meaning your `nuget.config` only needs to point to *one* URL to fetch both your internal packages and public packages from `nuget.org` (and it caches those public packages in case `nuget.org` ever hiccups).

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/c6b6c634-58f7-4d97-b506-9898cf7fa60b" />


---

## 2. Setting Up the Feed & Local `nuget.config`
1. In Azure DevOps, go to **Artifacts → Create Feed**.
2. Give it a name (e.g., `dotnet-shared-feed`), keep **Upstream sources** checked so `nuget.org` packages flow through automatically, and pick either **Project-scoped** or **Organization-scoped** (Org-scoped is usually better if multiple projects need the same libraries).
3. Click **Connect to feed → dotnet** and drop a `nuget.config` file in the root of the consuming repository:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="dotnet-shared-feed" value="[https://pkgs.dev.azure.com/](https://pkgs.dev.azure.com/)<your-org>/_packaging/dotnet-shared-feed/nuget/v3/index.json" />
  </packageSources>
</configuration>
