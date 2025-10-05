## 30. Azure App Service

### Short Introduction + Official Definition

Azure App Service is Microsoft's fully managed platform-as-a-service (PaaS) offering for hosting web applications, REST APIs, and mobile backends. It provides built-in auto-scaling, load balancing, security, and DevOps integration without requiring infrastructure management.

**Official Definition**: "Azure App Service is an HTTP-based service for hosting web applications, REST APIs, and mobile back ends. You can develop in your favorite language, be it .NET, .NET Core, Java, Ruby, Node.js, PHP, or Python."

### Setup and Deployment Steps

**Azure CLI Setup**:

```bash
# Login to Azure
az login

# Create resource group
az group create --name myResourceGroup --location "East US"

# Create App Service plan
az appservice plan create --name myAppServicePlan --resource-group myResourceGroup --sku B1

# Create web app
az webapp create --resource-group myResourceGroup --plan myAppServicePlan --name myUniqueAppName --runtime "DOTNET|8.0"

# Deploy from local folder
az webapp up --name myUniqueAppName --resource-group myResourceGroup
```

**Bicep Template**:

```bicep
param appName string = 'myapp${uniqueString(resourceGroup().id)}'
param location string = resourceGroup().location

resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: '${appName}-plan'
  location: location
  sku: {
    name: 'B1'
    tier: 'Basic'
  }
  properties: {
    reserved: false
  }
}

resource webApp 'Microsoft.Web/sites@2022-03-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      netFrameworkVersion: 'v8.0'
      metadata: [
        {
          name: 'CURRENT_STACK'
          value: 'dotnet'
        }
      ]
    }
  }
}
```

### Typical Usage and Integration with .NET Apps

```csharp
// Program.cs for App Service deployment
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Configure for App Service
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
});

var app = builder.Build();

// Configure pipeline for App Service
app.UseForwardedHeaders();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

**Configuration in appsettings.json**:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Server=tcp:myserver.database.windows.net,1433;Database=mydb;User ID=myuser;Password={your_password};Encrypt=True;"
  }
}
```

### Use Cases

- Web applications and APIs requiring quick deployment
- Applications needing built-in scaling and load balancing
- DevOps integration with Azure DevOps or GitHub Actions
- Applications requiring SSL certificates and custom domains
- Staging and production slot deployments
- Integration with other Azure services (SQL Database, Storage, etc.)

### When to Use vs Alternatives

**Use Azure App Service when**:

- You need rapid deployment without infrastructure management
- Auto-scaling based on metrics is important
- Built-in CI/CD integration is valuable
- SSL and custom domain management should be automated

**Don't use when**:

- You need full control over the underlying OS
- Custom software installation is required
- Cost optimization for always-on applications is critical
- Multi-cloud deployment is a requirement

**Alternatives**:

- **AWS**: Elastic Beanstalk, AWS Lambda (serverless)
- **GCP**: App Engine, Cloud Run
- **Self-hosted**: Docker containers on VMs, Kubernetes

### Market Pros/Cons and Cost Considerations

**Pros**:

- Zero infrastructure management
- Built-in auto-scaling and load balancing
- Integrated with Azure ecosystem
- Easy SSL certificate management
- Deployment slots for blue-green deployments

**Cons**:

- Vendor lock-in to Azure
- Limited customization of underlying infrastructure
- Can be expensive for high-traffic applications
- Cold start issues with lower-tier plans

**Cost Considerations**:

- Free tier available (limited resources)
- Basic plans start at ~$13/month
- Standard plans (~$56/month) include staging slots
- Premium plans (~$112/month) include advanced scaling
