# 11 - Docker & Azure Container Registry (ACR)

Deploying zip files directly to Azure App Service works great for simple setups, but once you move into microservices or want identical runtime environments between your local machine, QA, and Production, **containers** are the standard. 

Today I containerized the `.NET 8 Web API` using a multi-stage `Dockerfile`, set up a private **Azure Container Registry (ACR)**, and configured an Azure Pipeline to automatically build, tag, and push Docker images on every `main` commit.

---
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/bf7c65fc-2667-468a-9c63-9c293918203f" />


## 1. Writing a Lean Multi-Stage `Dockerfile` for .NET
If you build and run your app inside the full `.NET SDK` image, your final container ends up around ~800 MB and includes compilers and build tools that shouldn't exist in production. Using a **multi-stage build** compiles the code inside the SDK image, copies *only* the published DLLs into the lightweight `aspnet` runtime image (~215 MB), and drops everything else:

```dockerfile
# Stage 1: Build & Publish using the full .NET 8 SDK
FROM [mcr.microsoft.com/dotnet/sdk:8.0](https://mcr.microsoft.com/dotnet/sdk:8.0) AS build
WORKDIR /src

# Copy csproj first and restore to leverage Docker layer caching!
COPY ["src/WebApi/WebApi.csproj", "src/WebApi/"]
RUN dotnet restore "src/WebApi/WebApi.csproj"

# Copy remaining source files and publish
COPY . .
WORKDIR "/src/src/WebApi"
RUN dotnet publish "WebApi.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Stage 2: Final lightweight ASP.NET Core runtime image
FROM [mcr.microsoft.com/dotnet/aspnet:8.0](https://mcr.microsoft.com/dotnet/aspnet:8.0) AS final
WORKDIR /app
EXPOSE 8080

# Run as non-root user for better container security
USER app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "WebApi.dll"]
```

(Tip: Always add a .dockerignore file next to your Dockerfile excluding bin/, obj/, .git/, and node_modules/ so COPY . . doesn't send hundreds of megabytes of local build junk into the Docker daemon.)

## 2. Setting Up ACR & Service Connection
- Create ACR in Azure: Spin up an Azure Container Registry (Basic tier is plenty for learning/dev—e.g., acrdotnetdemo01.azurecr.io). Keep Admin user disabled in favor of identity-based auth.

- Create Docker Registry Service Connection: In Azure DevOps, go to Project Settings → Service connections → New service connection → Docker Registry.

- Pick Azure Container Registry, select your subscription and acrdotnetdemo01, name the connection ACR-Docker-SvcConn, and save. (Under the hood, AzDO creates a Service Principal with AcrPush permissions on that registry.)

## 3. Building, Tagging & Pushing via Docker@2
Instead of manually running docker login, docker build, and docker push bash scripts, Azure Pipelines provides the built-in Docker@2 task.

Notice how I tag every image with both $(Build.BuildId) (for exact traceability and easy rollbacks) and latest:

```
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  dockerRegistryServiceConnection: 'ACR-Docker-SvcConn'
  imageRepository: 'dotnet-webapi'
  containerRegistry: 'acrdotnetdemo01.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
  - stage: BuildAndPushImage
    displayName: 'CI: Build & Push Docker Image'
    jobs:
      - job: DockerBuild
        displayName: 'Build & Push to ACR'
        steps:
          - task: Docker@2
            displayName: 'Build and Push Image to ACR'
            inputs:
              command: 'buildAndPush'
              repository: '$(imageRepository)'
              dockerfile: '$(dockerfilePath)'
              containerRegistry: '$(dockerRegistryServiceConnection)'
              tags: |
                $(tag)
                latest

  - stage: DeployContainer
    displayName: 'CD: Deploy Container to Dev'
    dependsOn: BuildAndPushImage
    condition: succeeded()
    jobs:
      - deployment: DeployWebAppContainer
        environment: 'dev-env'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebAppContainer@1
                  displayName: 'Deploy Image to Azure Web App for Containers'
                  inputs:
                    azureSubscription: 'Azure-Subscription-SvcConn'
                    appName: 'app-dotnet-container-dev'
                    containers: '$(containerRegistry)/$(imageRepository):$(tag)'
```

## 4. Gotchas I Ran Into
- .NET 8 default container port changed to 8080: In .NET 6/7, official ASP.NET Docker images listened on port 80 as root. Starting in .NET 8, Microsoft switched the default non-root USER app port to 8080. If your Azure Web App expects port 80, set the App Setting WEBSITES_PORT=8080 or the container will fail health checks on startup.

- Avoid deploying :latest in Production: Always pass the immutable $(Build.BuildId) tag (e.g., dotnet-webapi:412) into AzureWebAppContainer@1. If you deploy :latest, Azure won't always detect that the underlying image digest changed, and rolling back to yesterday's release becomes a guessing game.

- Docker layer caching on hosted agents: Because Microsoft-hosted agents (ubuntu-latest) are destroyed after every job, Docker starts with an empty cache each run. Copying *.csproj and running dotnet restore before COPY . . still helps during local dev, and for larger enterprise solutions you can add Cache@2 or build on a persistent self-hosted agent.

## 5. Hands-On Checklist
- [x] Added a multi-stage Dockerfile (SDK → ASP.NET 8 runtime) and .dockerignore to the repository root, and verified docker build + docker run -p 8080:8080 locally.

- [x] Provisioned an Azure Container Registry (acrdotnetdemo01) and linked it via an ACR-Docker-SvcConn service connection.

- [x] Configured Docker@2 (buildAndPush) to tag images with both $(Build.BuildId) and latest.

- [x] Deployed the versioned container image to Azure Web App for Containers using AzureWebAppContainer@1 and confirmed the Swagger UI loaded cleanly on port 8080.

- [ ] Up next in 11: Take these containerized workloads to the next level by deploying to Azure Kubernetes Service (AKS) with Kubernetes manifests and Helm charts.
