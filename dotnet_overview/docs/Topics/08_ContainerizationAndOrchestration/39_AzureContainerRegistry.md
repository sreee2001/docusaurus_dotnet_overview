---
slug: azure_container_registry
title: Azure Container Registry
tags: [dotnet, containers, azure, registry, acr, docker]
---

# Azure Container Registry

## Short Introduction

Azure Container Registry (ACR) is a managed, private Docker registry service based on the open-source Docker Registry 2.0. It provides secure, scalable storage for container images and related artifacts, with built-in security scanning, geo-replication, and integration with Azure services and DevOps pipelines.

## Official Definition

Azure Container Registry is a managed, private Docker registry service that allows you to build, store, and manage container images and related artifacts in a private registry for all types of container deployments. It's based on the open-source Docker Registry 2.0.

## Setup/Usage with .NET 8+ Code

### Create ACR using Azure CLI

```bash
# Create resource group
az group create --name rg-hotel-management --location eastus

# Create Azure Container Registry
az acr create \
  --resource-group rg-hotel-management \
  --name hotelmanagementacr \
  --sku Standard \
  --admin-enabled true

# Login to ACR
az acr login --name hotelmanagementacr

# Get login server
ACR_LOGIN_SERVER=$(az acr show --name hotelmanagementacr --query loginServer --output tsv)
echo $ACR_LOGIN_SERVER
```

### Build and Push Images

```bash
# Build image locally and push
docker build -t hotelmanagementacr.azurecr.io/hotel-management-api:v1.0.0 .
docker push hotelmanagementacr.azurecr.io/hotel-management-api:v1.0.0

# Or use ACR Build (serverless build)
az acr build \
  --registry hotelmanagementacr \
  --image hotel-management-api:v1.0.0 \
  --image hotel-management-api:latest \
  .

# Import image from Docker Hub
az acr import \
  --name hotelmanagementacr \
  --source mcr.microsoft.com/dotnet/aspnet:8.0 \
  --image dotnet/aspnet:8.0
```

### ACR Tasks for Automated Builds

```bash
# Create ACR task for automated builds
az acr task create \
  --registry hotelmanagementacr \
  --name hotel-api-build-task \
  --image hotel-management-api:{{.Run.ID}} \
  --image hotel-management-api:latest \
  --context https://github.com/yourusername/hotel-management.git \
  --file Dockerfile \
  --git-access-token $(cat ~/github-pat.txt) \
  --branch main

# Run task manually
az acr task run --registry hotelmanagementacr --name hotel-api-build-task

# List task runs
az acr task list-runs --registry hotelmanagementacr --output table
```

## Use Cases

- **Private Container Storage**: Secure storage for proprietary container images
- **CI/CD Integration**: Automated builds and deployments in DevOps pipelines
- **Multi-stage Builds**: Complex build scenarios with ACR Tasks
- **Security Scanning**: Vulnerability assessment of container images
- **Geo-replication**: Global distribution of container images
- **Webhook Integration**: Trigger deployments on image updates

## When to Use vs When Not to Use

### Use Azure Container Registry when

- Need private, secure container image storage
- Building CI/CD pipelines with Azure services
- Require vulnerability scanning and compliance
- Need geo-distributed image storage
- Working with Azure Kubernetes Service or Container Apps
- Managing multiple container images and versions

### Consider alternatives when

- Using public images only (Docker Hub sufficient)
- Working primarily with other cloud providers
- Very simple, single-container applications
- Budget constraints for small projects
- No security or compliance requirements

## Market Alternatives & Pros/Cons

### Alternatives:

- **Amazon ECR**: AWS Elastic Container Registry
- **Google Container Registry**: GCP container registry
- **Docker Hub**: Public and private repositories
- **Harbor**: Open-source enterprise registry
- **JFrog Artifactory**: Universal artifact repository
- **GitLab Container Registry**: Integrated with GitLab CI/CD

### Pros:

- Integrated with Azure ecosystem
- Built-in security scanning and compliance
- Geo-replication capabilities
- Serverless builds with ACR Tasks
- Azure Active Directory integration
- Webhook and event integration

### Cons:

- Azure-specific (vendor lock-in)
- Cost can be higher than alternatives
- Limited features in basic tier
- Requires Azure expertise
- Storage costs for large images

## Complete Runnable Sample

### Complete ACR Setup with Bicep

```bicep
// acr-setup.bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('Name of the Azure Container Registry')
param acrName string = 'hotelmanagementacr'

@description('SKU for the Azure Container Registry')
@allowed(['Basic', 'Standard', 'Premium'])
param acrSku string = 'Standard'

// Azure Container Registry
resource acr 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: acrName
  location: location
  sku: {
    name: acrSku
  }
  properties: {
    adminUserEnabled: true
    policies: {
      quarantinePolicy: {
        status: 'enabled'
      }
      trustPolicy: {
        type: 'Notary'
        status: 'enabled'
      }
      retentionPolicy: {
        days: 30
        status: 'enabled'
      }
    }
    encryption: {
      status: 'disabled'
    }
    dataEndpointEnabled: false
    publicNetworkAccess: 'Enabled'
    networkRuleBypassOptions: 'AzureServices'
  }
}

// Enable vulnerability scanning (Premium tier only)
resource vulnerabilityAssessment 'Microsoft.Security/assessmentMetadata@2021-06-01' = if (acrSku == 'Premium') {
  name: 'vulnerabilityAssessment'
  properties: {
    displayName: 'Container Registry Vulnerability Assessment'
    description: 'Vulnerability assessment for container images'
    remediationDescription: 'Fix identified vulnerabilities'
    category: 'Compute'
    severity: 'High'
    userImpact: 'High'
    implementationEffort: 'Low'
  }
}

// Output values
output acrLoginServer string = acr.properties.loginServer
output acrName string = acr.name
output acrId string = acr.id
```

### GitHub Actions with ACR Integration

```yaml
# .github/workflows/acr-build-deploy.yml
name: Build and Deploy to ACR

on:
  push:
    branches: [main, develop]
    paths:
      - "src/**"
      - "Dockerfile"
  pull_request:
    branches: [main]

env:
  REGISTRY_NAME: hotelmanagementacr
  REGISTRY_URL: hotelmanagementacr.azurecr.io
  REPOSITORY_NAME: hotel-management-api
  RESOURCE_GROUP: rg-hotel-management

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: "8.0.x"

      - name: Restore dependencies
        run: dotnet restore

      - name: Build application
        run: dotnet build --no-restore --configuration Release

      - name: Run tests
        run: dotnet test --no-build --configuration Release --verbosity normal

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Build and push to ACR
        run: |
          # Get build number
          BUILD_NUMBER=${{ github.run_number }}
          IMAGE_TAG="v1.0.$BUILD_NUMBER"

          # Build and push using ACR build
          az acr build \
            --registry $REGISTRY_NAME \
            --image $REPOSITORY_NAME:$IMAGE_TAG \
            --image $REPOSITORY_NAME:latest \
            --file Dockerfile \
            .
            
          # Add additional tags
          az acr import \
            --name $REGISTRY_NAME \
            --source $REGISTRY_URL/$REPOSITORY_NAME:$IMAGE_TAG \
            --image $REPOSITORY_NAME:stable \
            --force

      - name: Scan image for vulnerabilities
        run: |
          # Wait for image to be available
          sleep 30

          # Trigger vulnerability scan
          az acr repository show-tags \
            --name $REGISTRY_NAME \
            --repository $REPOSITORY_NAME \
            --output table

      - name: Clean up old images
        run: |
          # Keep only last 10 images
          az acr repository show-tags \
            --name $REGISTRY_NAME \
            --repository $REPOSITORY_NAME \
            --orderby time_desc \
            --output tsv \
            | tail -n +11 \
            | xargs -I {} az acr repository delete \
              --name $REGISTRY_NAME \
              --image $REPOSITORY_NAME:{} \
              --yes || true

  security-scan:
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.event_name == 'push'

    steps:
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ env.REGISTRY_URL }}/${{ env.REPOSITORY_NAME }}:latest"
          format: "sarif"
          output: "trivy-results.sarif"

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: "trivy-results.sarif"
```

### ACR Integration with Kubernetes

```yaml
# k8s/acr-integration.yaml
apiVersion: v1
kind: Secret
metadata:
  name: acr-secret
  namespace: hotel-management
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hotel-api
  namespace: hotel-management
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hotel-api
  template:
    metadata:
      labels:
        app: hotel-api
    spec:
      imagePullSecrets:
        - name: acr-secret
      containers:
        - name: hotel-api
          image: hotelmanagementacr.azurecr.io/hotel-management-api:latest
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
```

### PowerShell Script for ACR Management

```powershell
# acr-management.ps1
param(
    [Parameter(Mandatory=$true)]
    [string]$RegistryName,

    [Parameter(Mandatory=$true)]
    [string]$ResourceGroup,

    [string]$ImageName = "hotel-management-api",

    [string]$Tag = "latest"
)

# Function to get ACR credentials
function Get-ACRCredentials {
    param($RegistryName)

    $credentials = az acr credential show --name $RegistryName | ConvertFrom-Json
    return @{
        Username = $credentials.username
        Password = $credentials.passwords[0].value
        LoginServer = "$RegistryName.azurecr.io"
    }
}

# Function to list images
function Get-ACRImages {
    param($RegistryName)

    Write-Host "Images in registry $RegistryName:"
    $repositories = az acr repository list --name $RegistryName | ConvertFrom-Json

    foreach ($repo in $repositories) {
        Write-Host "Repository: $repo"
        $tags = az acr repository show-tags --name $RegistryName --repository $repo | ConvertFrom-Json
        foreach ($tag in $tags) {
            $manifest = az acr repository show --name $RegistryName --image "$repo`:$tag" | ConvertFrom-Json
            Write-Host "  Tag: $tag, Created: $($manifest.createdTime), Size: $($manifest.imageSize)"
        }
    }
}

# Function to clean up old images
function Remove-OldACRImages {
    param(
        $RegistryName,
        $Repository,
        [int]$KeepCount = 5
    )

    Write-Host "Cleaning up old images for $Repository, keeping $KeepCount latest"

    $tags = az acr repository show-tags --name $RegistryName --repository $Repository --orderby time_desc | ConvertFrom-Json

    if ($tags.Count -gt $KeepCount) {
        $tagsToDelete = $tags | Select-Object -Skip $KeepCount

        foreach ($tag in $tagsToDelete) {
            Write-Host "Deleting $Repository`:$tag"
            az acr repository delete --name $RegistryName --image "$Repository`:$tag" --yes
        }
    }
}

# Main execution
try {
    Write-Host "Managing ACR: $RegistryName"

    # Get credentials
    $creds = Get-ACRCredentials -RegistryName $RegistryName
    Write-Host "Login Server: $($creds.LoginServer)"

    # List current images
    Get-ACRImages -RegistryName $RegistryName

    # Clean up old images
    Remove-OldACRImages -RegistryName $RegistryName -Repository $ImageName -KeepCount 5

    Write-Host "ACR management completed successfully"
}
catch {
    Write-Error "Error managing ACR: $($_.Exception.Message)"
    exit 1
}
```

### Deployment Commands

```bash
# Deploy ACR with Bicep
az deployment group create \
  --resource-group rg-hotel-management \
  --template-file acr-setup.bicep \
  --parameters acrName=hotelmanagementacr acrSku=Standard

# Get ACR credentials for Kubernetes
ACR_NAME="hotelmanagementacr"
ACR_UNAME=$(az acr credential show -n $ACR_NAME --query="username" -o tsv)
ACR_PASSWD=$(az acr credential show -n $ACR_NAME --query="passwords[0].value" -o tsv)

# Create Docker registry secret in Kubernetes
kubectl create secret docker-registry acr-secret \
  --namespace=hotel-management \
  --docker-server=$ACR_NAME.azurecr.io \
  --docker-username=$ACR_UNAME \
  --docker-password=$ACR_PASSWD \
  --docker-email=admin@hotelmanagement.com

# Test image pull
docker pull hotelmanagementacr.azurecr.io/hotel-management-api:latest
```
