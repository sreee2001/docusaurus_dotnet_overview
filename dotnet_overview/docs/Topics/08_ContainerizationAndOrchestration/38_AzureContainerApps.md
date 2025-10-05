## 38. Azure Container Apps

### Short Introduction

Azure Container Apps is a fully managed serverless container platform that allows you to run containerized applications without managing infrastructure. It provides automatic scaling, traffic splitting, and built-in integration with Azure services, making it ideal for microservices and event-driven applications.

### Official Definition

Azure Container Apps is a fully managed environment that enables you to run microservices and containerized applications on a serverless platform. It's built on Kubernetes but abstracts away cluster management complexity while providing advanced microservices capabilities.

### Setup/Usage with .NET 8+ Code

**Azure CLI Setup:**

```bash
# Install Azure CLI extension
az extension add --name containerapp --upgrade

# Create resource group
az group create --name rg-hotel-management --location eastus

# Create Container Apps environment
az containerapp env create \
  --name hotel-management-env \
  --resource-group rg-hotel-management \
  --location eastus
```

**Container App Configuration:**

```yaml
# containerapp.yaml
properties:
  managedEnvironmentId: /subscriptions/{subscription-id}/resourceGroups/rg-hotel-management/providers/Microsoft.App/managedEnvironments/hotel-management-env
  configuration:
    secrets:
      - name: connection-string
        value: "Server=tcp:hotel-sql-server.database.windows.net,1433;Initial Catalog=HotelManagement;Persist Security Info=False;User ID=adminuser;Password=YourPassword123!;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
      - name: jwt-secret
        value: "your-super-secret-jwt-key-that-is-at-least-256-bits-long"
    ingress:
      external: true
      targetPort: 8080
      traffic:
        - weight: 100
          latestRevision: true
    registries:
      - server: hotelmanagementacr.azurecr.io
        username: hotelmanagementacr
        passwordSecretRef: registry-password
  template:
    containers:
      - image: hotelmanagementacr.azurecr.io/hotel-management-api:latest
        name: hotel-api
        env:
          - name: ASPNETCORE_ENVIRONMENT
            value: "Production"
          - name: ConnectionStrings__DefaultConnection
            secretRef: connection-string
          - name: JwtSettings__SecretKey
            secretRef: jwt-secret
        resources:
          cpu: 0.5
          memory: 1Gi
        probes:
          - type: Liveness
            httpGet:
              path: "/health"
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 30
          - type: Readiness
            httpGet:
              path: "/health/ready"
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
    scale:
      minReplicas: 1
      maxReplicas: 10
      rules:
        - name: http-rule
          http:
            requests: 100
```

**Deployment with Azure CLI:**

```bash
# Create container app
az containerapp create \
  --name hotel-management-api \
  --resource-group rg-hotel-management \
  --environment hotel-management-env \
  --image hotelmanagementacr.azurecr.io/hotel-management-api:latest \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 10 \
  --cpu 0.5 \
  --memory 1Gi \
  --secrets "connection-string=Server=tcp:hotel-sql-server.database.windows.net,1433;Initial Catalog=HotelManagement;User ID=adminuser;Password=YourPassword123!" \
  --env-vars "ASPNETCORE_ENVIRONMENT=Production" "ConnectionStrings__DefaultConnection=secretref:connection-string"

# Update container app
az containerapp update \
  --name hotel-management-api \
  --resource-group rg-hotel-management \
  --image hotelmanagementacr.azurecr.io/hotel-management-api:v2.0.0

# Enable HTTPS
az containerapp ingress enable \
  --name hotel-management-api \
  --resource-group rg-hotel-management \
  --type external \
  --target-port 8080 \
  --transport https
```

### Use Cases

- **Microservices Architecture**: Event-driven and HTTP-based microservices
- **API Backends**: REST APIs with automatic scaling
- **Background Processing**: Event-driven background jobs
- **Web Applications**: Containerized web apps with global distribution
- **Serverless Workloads**: Pay-per-use scaling applications
- **Development and Testing**: Quick deployment for CI/CD pipelines

### When to Use vs When Not to Use

**Use Azure Container Apps when:**

- Building microservices or API-first applications
- Need automatic scaling with zero-to-many instances
- Want serverless benefits without vendor lock-in
- Require event-driven architectures (KEDA integration)
- Need simple container deployment without Kubernetes complexity
- Building cloud-native applications on Azure

**Consider alternatives when:**

- Need full Kubernetes control and customization
- Running Windows containers (limited support)
- Require specific networking configurations
- Need persistent storage with StatefulSets
- Working with legacy applications requiring VMs
- Multi-cloud deployment requirements

### Market Alternatives & Pros/Cons

**Alternatives:**

- **AWS Fargate**: Serverless containers on AWS
- **Google Cloud Run**: Serverless containers on GCP
- **Azure Kubernetes Service (AKS)**: Full Kubernetes control
- **Azure App Service**: Platform-as-a-Service for web apps
- **AWS Lambda**: Function-as-a-Service
- **Azure Functions**: Serverless functions

**Pros:**

- Fully managed with no infrastructure overhead
- Automatic scaling including scale-to-zero
- Built-in traffic splitting and blue-green deployments
- KEDA integration for event-driven scaling
- Pay-per-use pricing model
- Simple deployment and management

**Cons:**

- Limited to Linux containers primarily
- Less control compared to full Kubernetes
- Azure-specific (vendor lock-in)
- Newer service with evolving features
- Limited persistent storage options
- Cold start latency for scale-to-zero scenarios

### Complete Runnable Sample

**Bicep Template for Infrastructure:**

```bicep
// main.bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('Name of the Container Apps Environment')
param environmentName string = 'hotel-management-env'

@description('Name of the Container App')
param containerAppName string = 'hotel-management-api'

@description('Container image')
param containerImage string = 'hotelmanagementacr.azurecr.io/hotel-management-api:latest'

// Container Apps Environment
resource environment 'Microsoft.App/managedEnvironments@2023-05-01' = {
  name: environmentName
  location: location
  properties: {
    daprAIInstrumentationKey: ''
    daprAIConnectionString: ''
    vnetConfiguration: {}
    zoneRedundant: false
  }
}

// Container App
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: containerAppName
  location: location
  properties: {
    managedEnvironmentId: environment.id
    configuration: {
      secrets: [
        {
          name: 'connection-string'
          value: 'Server=tcp:hotel-sql-server.database.windows.net,1433;Initial Catalog=HotelManagement;User ID=adminuser;Password=YourPassword123!;'
        }
        {
          name: 'jwt-secret'
          value: 'your-super-secret-jwt-key-that-is-at-least-256-bits-long'
        }
      ]
      ingress: {
        external: true
        targetPort: 8080
        allowInsecure: false
        traffic: [
          {
            weight: 100
            latestRevision: true
          }
        ]
      }
    }
    template: {
      containers: [
        {
          image: containerImage
          name: 'hotel-api'
          env: [
            {
              name: 'ASPNETCORE_ENVIRONMENT'
              value: 'Production'
            }
            {
              name: 'ASPNETCORE_URLS'
              value: 'http://+:8080'
            }
            {
              name: 'ConnectionStrings__DefaultConnection'
              secretRef: 'connection-string'
            }
            {
              name: 'JwtSettings__SecretKey'
              secretRef: 'jwt-secret'
            }
          ]
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
          probes: [
            {
              type: 'Liveness'
              httpGet: {
                path: '/health'
                port: 8080
              }
              initialDelaySeconds: 15
              periodSeconds: 30
            }
            {
              type: 'Readiness'
              httpGet: {
                path: '/health/ready'
                port: 8080
              }
              initialDelaySeconds: 5
              periodSeconds: 10
            }
          ]
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 10
        rules: [
          {
            name: 'http-scaling'
            http: {
              requests: 100
            }
          }
        ]
      }
    }
  }
}

output containerAppFQDN string = containerApp.properties.configuration.ingress.fqdn
```

**Deployment Script:**

```bash
#!/bin/bash
# deploy.sh

# Variables
RESOURCE_GROUP="rg-hotel-management"
LOCATION="eastus"
ACR_NAME="hotelmanagementacr"
APP_NAME="hotel-management-api"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Deploy infrastructure
az deployment group create \
  --resource-group $RESOURCE_GROUP \
  --template-file main.bicep \
  --parameters containerAppName=$APP_NAME

# Get the FQDN
FQDN=$(az deployment group show \
  --resource-group $RESOURCE_GROUP \
  --name main \
  --query properties.outputs.containerAppFQDN.value \
  --output tsv)

echo "Application deployed at: https://$FQDN"

# Test the deployment
curl -k "https://$FQDN/health"
```

**GitHub Actions Workflow:**

```yaml
# .github/workflows/deploy-to-containerapp.yml
name: Deploy to Azure Container Apps

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  REGISTRY_NAME: hotelmanagementacr
  RESOURCE_GROUP: rg-hotel-management
  CONTAINER_APP_NAME: hotel-management-api

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: "8.0.x"

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore -c Release

      - name: Test
        run: dotnet test --no-build --verbosity normal

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Build and push Docker image
        run: |
          az acr build --registry $REGISTRY_NAME \
            --image $CONTAINER_APP_NAME:${{ github.sha }} \
            --image $CONTAINER_APP_NAME:latest .

      - name: Deploy to Container App
        run: |
          az containerapp update \
            --name $CONTAINER_APP_NAME \
            --resource-group $RESOURCE_GROUP \
            --image $REGISTRY_NAME.azurecr.io/$CONTAINER_APP_NAME:${{ github.sha }}
```
