---
slug: azure_devops
title: Azure DevOps
tags: [dotnet, azure, devops]
---

# Azure DevOps

## Short Introduction

Azure DevOps is a comprehensive suite of development tools and services provided by Microsoft for planning, developing, testing, and deploying applications. It includes Azure Repos, Azure Pipelines, Azure Boards, Azure Test Plans, and Azure Artifacts, providing end-to-end DevOps capabilities for .NET applications.

## Official Definition

Azure DevOps provides developer services to support teams to plan work, collaborate on code development, and build and deploy applications. It supports any language, platform, and cloud, offering both cloud-hosted and on-premises solutions for development teams.

## Setup/Usage with .NET 8+ Code

### Basic Azure DevOps Pipeline (azure-pipelines.yml)

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - src/*
      - tests/*

variables:
  buildConfiguration: "Release"
  dotNetVersion: "8.x"
  vmImageName: "ubuntu-latest"

stages:
  - stage: Build
    displayName: "Build and Test"
    jobs:
      - job: Build
        displayName: "Build Job"
        pool:
          vmImage: $(vmImageName)

        steps:
          - task: UseDotNet@2
            displayName: "Use .NET $(dotNetVersion)"
            inputs:
              packageType: "sdk"
              version: $(dotNetVersion)

          - task: DotNetCoreCLI@2
            displayName: "Restore packages"
            inputs:
              command: "restore"
              projects: "**/*.csproj"

          - task: DotNetCoreCLI@2
            displayName: "Build application"
            inputs:
              command: "build"
              projects: "**/*.csproj"
              arguments: "--configuration $(buildConfiguration) --no-restore"

          - task: DotNetCoreCLI@2
            displayName: "Run unit tests"
            inputs:
              command: "test"
              projects: "**/*Tests.csproj"
              arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage" --logger trx --results-directory $(Agent.TempDirectory)'

          - task: PublishCodeCoverageResults@1
            displayName: "Publish code coverage"
            inputs:
              codeCoverageTool: "Cobertura"
              summaryFileLocation: "$(Agent.TempDirectory)/**/coverage.cobertura.xml"

          - task: PublishTestResults@2
            displayName: "Publish test results"
            inputs:
              testResultsFormat: "VSTest"
              testResultsFiles: "$(Agent.TempDirectory)/**/*.trx"

          - task: DotNetCoreCLI@2
            displayName: "Publish application"
            inputs:
              command: "publish"
              publishWebProjects: true
              arguments: "--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)"

          - task: PublishBuildArtifacts@1
            displayName: "Publish artifacts"
            inputs:
              PathtoPublish: "$(Build.ArtifactStagingDirectory)"
              ArtifactName: "drop"

  - stage: Deploy
    displayName: "Deploy to Azure"
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Deploy
        displayName: "Deploy Job"
        pool:
          vmImage: $(vmImageName)
        environment: "production"
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: "Deploy to Azure Web App"
                  inputs:
                    azureSubscription: "Azure-Service-Connection"
                    appType: "webAppLinux"
                    appName: "hotel-management-api"
                    package: "$(Pipeline.Workspace)/drop/**/*.zip"
```

### Multi-Stage Pipeline with Docker

```yaml
# azure-pipelines-docker.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - src/*
      - Dockerfile

variables:
  dockerRegistryServiceConnection: "ACR-Service-Connection"
  imageRepository: "hotel-management-api"
  containerRegistry: "hotelmanagementacr.azurecr.io"
  dockerfilePath: "$(Build.SourcesDirectory)/Dockerfile"
  tag: "$(Build.BuildId)"
  vmImageName: "ubuntu-latest"

stages:
  - stage: Build
    displayName: Build and Push Docker Image
    jobs:
      - job: Build
        displayName: Build
        pool:
          vmImage: $(vmImageName)
        steps:
          - task: Docker@2
            displayName: Build and push Docker image
            inputs:
              command: buildAndPush
              repository: $(imageRepository)
              dockerfile: $(dockerfilePath)
              containerRegistry: $(dockerRegistryServiceConnection)
              tags: |
                $(tag)
                latest

  - stage: Deploy
    displayName: Deploy to AKS
    dependsOn: Build
    jobs:
      - deployment: Deploy
        displayName: Deploy
        pool:
          vmImage: $(vmImageName)
        environment: "production.hotel-management"
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  displayName: Deploy to Kubernetes cluster
                  inputs:
                    action: deploy
                    manifests: |
                      $(Pipeline.Workspace)/manifests/deployment.yml
                      $(Pipeline.Workspace)/manifests/service.yml
                    containers: |
                      $(containerRegistry)/$(imageRepository):$(tag)
```

## Use Cases

- **Enterprise Development**: Large-scale development with multiple teams
- **Regulated Industries**: Compliance tracking and audit trails
- **Hybrid Deployments**: On-premises and cloud integration
- **Microsoft Ecosystem**: Integration with Visual Studio and Microsoft tools
- **Work Item Tracking**: Project management and requirement tracking
- **Release Management**: Complex deployment pipelines with approvals

## When to Use vs When Not to Use

### Use Azure DevOps when

- Working in Microsoft-centric environments
- Need integrated work item tracking and planning
- Require enterprise-grade security and compliance
- Building complex, multi-stage deployment pipelines
- Need on-premises DevOps capabilities
- Working with large teams requiring structured processes

### Consider alternatives when

- Primarily using non-Microsoft technologies
- Need simpler, lightweight CI/CD solutions
- Budget constraints (GitHub Actions often more cost-effective)
- Open-source focused development
- Small teams with minimal process requirements

## Market Alternatives & Pros/Cons

### Alternatives:

- **GitHub Actions**: Integrated with GitHub, simpler setup
- **GitLab CI/CD**: Complete DevOps platform with GitOps
- **Jenkins**: Open-source, highly customizable
- **CircleCI**: Fast builds, good Docker support
- **TeamCity**: JetBrains' CI/CD solution
- **AWS CodePipeline**: Native AWS integration

### Pros:

- Comprehensive integrated suite
- Strong Microsoft ecosystem integration
- Enterprise security and compliance features
- Flexible deployment options (cloud/on-premises)
- Advanced work item tracking and reporting
- Mature testing and release management

### Cons:

- Can be complex for simple projects
- Licensing costs for larger teams
- Less community-driven compared to open alternatives
- Learning curve for non-Microsoft environments
- YAML pipeline syntax can be verbose

## Complete Runnable Sample

<!-- ### Complete Enterprise Pipeline -->

```yaml
# azure-pipelines-enterprise.yml
name: $(Date:yyyyMMdd)$(Rev:.r)

trigger:
  batch: true
  branches:
    include:
      - main
      - develop
      - release/*
  paths:
    exclude:
      - docs/*
      - README.md

variables:
  - group: "HotelManagement-Variables"
  - name: buildConfiguration
    value: "Release"
  - name: dotNetVersion
    value: "8.x"
  - name: majorVersion
    value: "1"
  - name: minorVersion
    value: "0"
  - name: patchVersion
    value: $[counter(variables['minorVersion'], 0)]

pool:
  vmImage: "ubuntu-latest"

stages:
  - stage: Validate
    displayName: "Code Quality and Security"
    jobs:
      - job: StaticAnalysis
        displayName: "Static Code Analysis"
        steps:
          - task: UseDotNet@2
            displayName: "Install .NET SDK"
            inputs:
              packageType: "sdk"
              version: $(dotNetVersion)

          - task: DotNetCoreCLI@2
            displayName: "Restore packages"
            inputs:
              command: "restore"
              projects: "**/*.csproj"

          - task: SonarCloudPrepare@1
            displayName: "Prepare SonarCloud analysis"
            inputs:
              SonarCloud: "SonarCloud-ServiceConnection"
              organization: "hotel-management"
              scannerMode: "MSBuild"
              projectKey: "hotel-management-api"

          - task: DotNetCoreCLI@2
            displayName: "Build for analysis"
            inputs:
              command: "build"
              projects: "**/*.csproj"
              arguments: "--configuration $(buildConfiguration)"

          - task: SonarCloudAnalyze@1
            displayName: "Run SonarCloud analysis"

          - task: SonarCloudPublish@1
            displayName: "Publish SonarCloud results"

          - task: WhiteSource@21
            displayName: "WhiteSource security scan"
            inputs:
              cwd: "$(System.DefaultWorkingDirectory)"

  - stage: Build
    displayName: "Build and Test"
    dependsOn: Validate
    jobs:
      - job: BuildAndTest
        displayName: "Build and Test Application"
        steps:
          - task: UseDotNet@2
            displayName: "Install .NET SDK"
            inputs:
              packageType: "sdk"
              version: $(dotNetVersion)

          - task: DotNetCoreCLI@2
            displayName: "Restore packages"
            inputs:
              command: "restore"
              projects: "**/*.csproj"
              feedsToUse: "select"
              vstsFeed: "hotel-management-feed"

          - task: DotNetCoreCLI@2
            displayName: "Build application"
            inputs:
              command: "build"
              projects: "**/*.csproj"
              arguments: "--configuration $(buildConfiguration) --no-restore -p:Version=$(majorVersion).$(minorVersion).$(patchVersion)"

          - task: DotNetCoreCLI@2
            displayName: "Run unit tests"
            inputs:
              command: "test"
              projects: "**/*UnitTests.csproj"
              arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage" --logger trx --results-directory $(Agent.TempDirectory)/TestResults'

          - task: DotNetCoreCLI@2
            displayName: "Run integration tests"
            inputs:
              command: "test"
              projects: "**/*IntegrationTests.csproj"
              arguments: "--configuration $(buildConfiguration) --no-build --logger trx --results-directory $(Agent.TempDirectory)/TestResults"

          - task: PublishCodeCoverageResults@1
            displayName: "Publish code coverage"
            inputs:
              codeCoverageTool: "Cobertura"
              summaryFileLocation: "$(Agent.TempDirectory)/TestResults/**/coverage.cobertura.xml"

          - task: PublishTestResults@2
            displayName: "Publish test results"
            inputs:
              testResultsFormat: "VSTest"
              testResultsFiles: "$(Agent.TempDirectory)/TestResults/**/*.trx"
              publishRunAttachments: true

          - task: DotNetCoreCLI@2
            displayName: "Create NuGet packages"
            inputs:
              command: "pack"
              packagesToPack: "**/HotelManagement.Core.csproj;**/HotelManagement.Shared.csproj"
              arguments: "--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/packages -p:PackageVersion=$(majorVersion).$(minorVersion).$(patchVersion)"

          - task: DotNetCoreCLI@2
            displayName: "Publish web application"
            inputs:
              command: "publish"
              publishWebProjects: true
              arguments: "--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/webapp --no-build -p:Version=$(majorVersion).$(minorVersion).$(patchVersion)"

          - task: CopyFiles@2
            displayName: "Copy deployment scripts"
            inputs:
              SourceFolder: "deploy"
              Contents: "**"
              TargetFolder: "$(Build.ArtifactStagingDirectory)/deploy"

          - task: PublishBuildArtifacts@1
            displayName: "Publish build artifacts"
            inputs:
              PathtoPublish: "$(Build.ArtifactStagingDirectory)"
              ArtifactName: "drop"

  - stage: DeployDev
    displayName: "Deploy to Development"
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    variables:
      - group: "Dev-Environment"
    jobs:
      - deployment: DeployToDev
        displayName: "Deploy to Dev Environment"
        pool:
          vmImage: "ubuntu-latest"
        environment: "hotel-management-dev"
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureResourceManagerTemplateDeployment@3
                  displayName: "Deploy ARM template"
                  inputs:
                    deploymentScope: "Resource Group"
                    azureResourceManagerConnection: "Azure-Dev-ServiceConnection"
                    subscriptionId: "$(subscriptionId)"
                    action: "Create Or Update Resource Group"
                    resourceGroupName: "$(resourceGroupName)"
                    location: "$(location)"
                    templateLocation: "Linked artifact"
                    csmFile: "$(Pipeline.Workspace)/drop/deploy/infrastructure.json"
                    csmParametersFile: "$(Pipeline.Workspace)/drop/deploy/parameters.dev.json"

                - task: AzureWebApp@1
                  displayName: "Deploy to Azure Web App"
                  inputs:
                    azureSubscription: "Azure-Dev-ServiceConnection"
                    appType: "webAppLinux"
                    appName: "$(webAppName)"
                    package: "$(Pipeline.Workspace)/drop/webapp/**/*.zip"
                    appSettings: |
                      -ASPNETCORE_ENVIRONMENT Development
                      -ConnectionStrings__DefaultConnection "$(connectionString)"
                      -ApplicationInsights__InstrumentationKey "$(instrumentationKey)"

  - stage: DeployProd
    displayName: "Deploy to Production"
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    variables:
      - group: "Prod-Environment"
    jobs:
      - deployment: DeployToProd
        displayName: "Deploy to Production Environment"
        pool:
          vmImage: "ubuntu-latest"
        environment: "hotel-management-prod"
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureKeyVault@2
                  displayName: "Get secrets from Key Vault"
                  inputs:
                    azureSubscription: "Azure-Prod-ServiceConnection"
                    KeyVaultName: "$(keyVaultName)"
                    SecretsFilter: "*"

                - task: AzureResourceManagerTemplateDeployment@3
                  displayName: "Deploy ARM template"
                  inputs:
                    deploymentScope: "Resource Group"
                    azureResourceManagerConnection: "Azure-Prod-ServiceConnection"
                    subscriptionId: "$(subscriptionId)"
                    action: "Create Or Update Resource Group"
                    resourceGroupName: "$(resourceGroupName)"
                    location: "$(location)"
                    templateLocation: "Linked artifact"
                    csmFile: "$(Pipeline.Workspace)/drop/deploy/infrastructure.json"
                    csmParametersFile: "$(Pipeline.Workspace)/drop/deploy/parameters.prod.json"

                - task: AzureWebApp@1
                  displayName: "Deploy to Azure Web App"
                  inputs:
                    azureSubscription: "Azure-Prod-ServiceConnection"
                    appType: "webAppLinux"
                    appName: "$(webAppName)"
                    package: "$(Pipeline.Workspace)/drop/webapp/**/*.zip"
                    deploymentMethod: "zipDeploy"
                    appSettings: |
                      -ASPNETCORE_ENVIRONMENT Production
                      -ConnectionStrings__DefaultConnection "$(prod-connection-string)"
                      -ApplicationInsights__InstrumentationKey "$(prod-instrumentation-key)"

                - task: AzureCLI@2
                  displayName: "Run database migrations"
                  inputs:
                    azureSubscription: "Azure-Prod-ServiceConnection"
                    scriptType: "bash"
                    scriptLocation: "inlineScript"
                    inlineScript: |
                      az webapp config appsettings set \
                        --resource-group $(resourceGroupName) \
                        --name $(webAppName) \
                        --settings RunMigrations=true

                      # Wait for app to restart and run migrations
                      sleep 60

                      az webapp config appsettings delete \
                        --resource-group $(resourceGroupName) \
                        --name $(webAppName) \
                        --setting-names RunMigrations
```
