## Application Insights (Observability + Instrumentation)

### Short Introduction + Official Definition

Application Insights is Azure's application performance management (APM) service that provides deep insights into application performance, availability, and usage. It automatically collects telemetry data and provides powerful analytics and alerting capabilities.

**Official Definition**: "Application Insights, a feature of Azure Monitor, is an extensible Application Performance Management (APM) service for developers and DevOps professionals. Use it to monitor your live applications automatically."

### Setup and Deployment Steps

**Azure CLI Setup**:

```bash
# Create Application Insights resource
az monitor app-insights component create --app myapp --location eastus --resource-group myResourceGroup --application-type web

# Get instrumentation key
az monitor app-insights component show --app myapp --resource-group myResourceGroup --query instrumentationKey
```

**Bicep Template**:

```bicep
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'myapp-insights'
  location: resourceGroup().location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    Flow_Type: 'Bluefield'
    Request_Source: 'rest'
  }
}

output instrumentationKey string = appInsights.properties.InstrumentationKey
output connectionString string = appInsights.properties.ConnectionString
```

### Typical Usage and Integration with .NET Apps

**NuGet Package**:

```xml
<PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.21.0" />
```

**Basic Configuration**:

```csharp
// appsettings.json
{
  "ApplicationInsights": {
    "ConnectionString": "InstrumentationKey=your-key;IngestionEndpoint=https://eastus-8.in.applicationinsights.azure.com/;LiveEndpoint=https://eastus.livediagnostics.monitor.azure.com/"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    },
    "ApplicationInsights": {
      "LogLevel": {
        "Default": "Information"
      }
    }
  }
}

// Program.cs
using Microsoft.ApplicationInsights.Extensibility;

var builder = WebApplication.CreateBuilder(args);

// Add Application Insights
builder.Services.AddApplicationInsightsTelemetry(builder.Configuration);

// Configure sampling (optional)
builder.Services.Configure<TelemetryConfiguration>(config =>
{
    config.DefaultTelemetrySink.TelemetryProcessorChainBuilder
        .UseAdaptiveSampling(maxTelemetryItemsPerSecond: 5)
        .Build();
});

var app = builder.Build();

// Add custom middleware for additional tracking
app.UseMiddleware<RequestTelemetryMiddleware>();
```

**Custom Telemetry Tracking**:

```csharp
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.DataContracts;

[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly TelemetryClient _telemetryClient;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(TelemetryClient telemetryClient, ILogger<OrdersController> logger)
    {
        _telemetryClient = telemetryClient;
        _logger = logger;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        using var operation = _telemetryClient.StartOperation<RequestTelemetry>("CreateOrder");

        try
        {
            // Add custom properties
            _telemetryClient.TrackEvent("OrderCreationStarted", new Dictionary<string, string>
            {
                ["CustomerId"] = request.CustomerId.ToString(),
                ["OrderValue"] = request.TotalAmount.ToString("C")
            });

            // Track custom metrics
            _telemetryClient.TrackMetric("OrderValue", request.TotalAmount);

            // Simulate order processing
            var stopwatch = Stopwatch.StartNew();
            await ProcessOrderAsync(request);
            stopwatch.Stop();

            // Track dependency call
            _telemetryClient.TrackDependency("Database", "CreateOrder",
                DateTime.UtcNow.Subtract(stopwatch.Elapsed), stopwatch.Elapsed, true);

            _telemetryClient.TrackEvent("OrderCreated", new Dictionary<string, string>
            {
                ["OrderId"] = "12345",
                ["ProcessingTimeMs"] = stopwatch.ElapsedMilliseconds.ToString()
            });

            return Ok(new { OrderId = 12345 });
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex, new Dictionary<string, string>
            {
                ["CustomerId"] = request.CustomerId.ToString(),
                ["Operation"] = "CreateOrder"
            });

            _logger.LogError(ex, "Failed to create order for customer {CustomerId}", request.CustomerId);
            return StatusCode(500, "Order creation failed");
        }
    }

    private async Task ProcessOrderAsync(CreateOrderRequest request)
    {
        // Simulate processing time
        await Task.Delay(Random.Shared.Next(100, 500));

        // Simulate occasional errors
        if (Random.Shared.NextDouble() < 0.1)
        {
            throw new InvalidOperationException("Simulated order processing error");
        }
    }
}
```

**Custom Telemetry Middleware**:

```csharp
public class RequestTelemetryMiddleware
{
    private readonly RequestDelegate _next;
    private readonly TelemetryClient _telemetryClient;

    public RequestTelemetryMiddleware(RequestDelegate next, TelemetryClient telemetryClient)
    {
        _next = next;
        _telemetryClient = telemetryClient;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();

            // Track custom metrics about the request
            _telemetryClient.TrackMetric("RequestDuration", stopwatch.ElapsedMilliseconds);

            if (context.Response.StatusCode >= 400)
            {
                _telemetryClient.TrackEvent("ErrorResponse", new Dictionary<string, string>
                {
                    ["StatusCode"] = context.Response.StatusCode.ToString(),
                    ["Path"] = context.Request.Path,
                    ["Method"] = context.Request.Method
                });
            }
        }
    }
}
```

**Structured Logging Integration**:

```csharp
public class ProductService
{
    private readonly ILogger<ProductService> _logger;
    private readonly TelemetryClient _telemetryClient;

    public ProductService(ILogger<ProductService> logger, TelemetryClient telemetryClient)
    {
        _logger = logger;
        _telemetryClient = telemetryClient;
    }

    public async Task<Product> GetProductAsync(int productId)
    {
        using (_logger.BeginScope(new Dictionary<string, object> { ["ProductId"] = productId }))
        {
            _logger.LogInformation("Retrieving product {ProductId}", productId);

            var stopwatch = Stopwatch.StartNew();

            try
            {
                // Simulate database call
                await Task.Delay(Random.Shared.Next(50, 200));

                var product = new Product { Id = productId, Name = $"Product {productId}" };

                _logger.LogInformation("Successfully retrieved product {ProductId} in {ElapsedMs}ms",
                    productId, stopwatch.ElapsedMilliseconds);

                return product;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to retrieve product {ProductId}", productId);
                throw;
            }
        }
    }
}
```

**Health Checks Integration**:

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddApplicationInsightsPublisher();

// Custom health check
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly ILogger<DatabaseHealthCheck> _logger;

    public DatabaseHealthCheck(ILogger<DatabaseHealthCheck> logger)
    {
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Simulate database connectivity check
            await Task.Delay(100, cancellationToken);

            _logger.LogInformation("Database health check passed");
            return HealthCheckResult.Healthy("Database is responsive");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database health check failed");
            return HealthCheckResult.Unhealthy("Database is not responsive", ex);
        }
    }
}
```

### Use Cases

- Application performance monitoring and optimization
- Error tracking and exception analysis
- User behavior analytics and usage patterns
- Infrastructure monitoring and dependency tracking
- Real-time alerting on performance issues
- A/B testing and feature flag monitoring

### When to Use vs Alternatives

**Use Application Insights when**:

- Azure-hosted applications requiring deep integration
- Comprehensive APM solution needed
- Integration with Azure Monitor ecosystem important
- Built-in analytics and alerting capabilities desired
- Automatic instrumentation for common frameworks preferred

**Don't use when**:

- Multi-cloud monitoring strategy required
- Cost optimization for simple monitoring needs
- Existing monitoring infrastructure in place
- Real-time streaming analytics primary requirement

**Alternatives**:

- **Azure**: Azure Monitor Logs, Azure Monitor Metrics
- **AWS**: CloudWatch, X-Ray
- **GCP**: Cloud Monitoring, Cloud Trace
- **Open Source**: Prometheus + Grafana, Jaeger, Zipkin
- **Commercial**: New Relic, Datadog, Dynatrace

### Market Pros/Cons and Cost Considerations

**Pros**:

- Automatic instrumentation for ASP.NET Core applications
- Rich analytics and query capabilities (KQL)
- Integration with Azure DevOps and GitHub
- Smart detection of performance anomalies
- Comprehensive dependency mapping

**Cons**:

- Can be expensive for high-volume applications
- Learning curve for advanced analytics queries
- Data retention limits on different tiers
- Vendor lock-in to Azure ecosystem

**Cost Considerations**:

- Pay-per-GB of data ingested (~$2.30/GB for first 5GB/month)
- First 5GB per month included with many Azure services
- Data retention: 90 days included, extended retention available
- Additional costs for multi-step web tests and alerts
- Enterprise features require higher-tier pricing

### Minimal Sample: Deploy ASP.NET Core App to App Service with Application Insights

**Complete Deployment Script**:

```bash
# Variables
RESOURCE_GROUP="hotel-demo-rg"
APP_NAME="hotel-api-$(date +%s)"
LOCATION="eastus"
APP_SERVICE_PLAN="hotel-plan"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create App Service plan
az appservice plan create --name $APP_SERVICE_PLAN --resource-group $RESOURCE_GROUP --sku B1

# Create Application Insights
az monitor app-insights component create \
    --app $APP_NAME-insights \
    --location $LOCATION \
    --resource-group $RESOURCE_GROUP \
    --application-type web

# Get Application Insights connection string
INSIGHTS_CONNECTION=$(az monitor app-insights component show \
    --app $APP_NAME-insights \
    --resource-group $RESOURCE_GROUP \
    --query connectionString -o tsv)

# Create web app
az webapp create \
    --resource-group $RESOURCE_GROUP \
    --plan $APP_SERVICE_PLAN \
    --name $APP_NAME \
    --runtime "DOTNET|8.0"

# Configure Application Insights
az webapp config appsettings set \
    --resource-group $RESOURCE_GROUP \
    --name $APP_NAME \
    --settings "APPLICATIONINSIGHTS_CONNECTION_STRING=$INSIGHTS_CONNECTION"

# Deploy app (assuming you have a zip package)
# az webapp deployment source config-zip --resource-group $RESOURCE_GROUP --name $APP_NAME --src app.zip

echo "App deployed to: https://$APP_NAME.azurewebsites.net"
echo "Application Insights: https://portal.azure.com/#@/resource/subscriptions/.../resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Insights/components/$APP_NAME-insights"
```

This completes the Cloud Technologies section covering all Azure services with comprehensive setup instructions, code examples, and practical guidance for .NET Core applications. Each service includes real-world usage patterns and cost considerations to help make informed architectural decisions.
