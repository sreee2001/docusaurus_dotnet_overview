---
slug: application_insights
title: Application Insights (Observability + Instrumentation)
tags:
  [
    dotnet,
    application,
    insights,
    observability,
    instrumentation,
    APM,
    service,
    metrics,
  ]
---

# Application Insights (Observability + Instrumentation)

## Short Introduction

Application Insights is Azure's application performance management (APM) service that provides real-time monitoring, diagnostics, and analytics for .NET applications. It automatically collects telemetry data including requests, dependencies, exceptions, and custom metrics, enabling developers to detect issues, optimize performance, and understand user behavior.

## Official Definition

"Application Insights, a feature of Azure Monitor, is an extensible Application Performance Management (APM) service for developers and DevOps professionals. Use it to monitor your live applications automatically."

Application Insights is a feature of Azure Monitor that provides extensible application performance management (APM) and monitoring for live web applications. It works for apps on a wide variety of platforms including .NET, Node.js, Java, and Python hosted on-premises, hybrid, or any public cloud.

## Setup and Deployment

### Azure CLI Setup

```bash
# Create Application Insights resource
az monitor app-insights component create --app myapp --location eastus --resource-group myResourceGroup --application-type web

# Get instrumentation key
az monitor app-insights component show --app myapp --resource-group myResourceGroup --query instrumentationKey
```

### Bicep Template

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

### Minimal Sample: Deploy ASP.NET Core App to App Service with Application Insights

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

## Typical Usage and Integration with .NET Apps

### NuGet Packages

```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
dotnet add package Microsoft.ApplicationInsights.PerfCounterCollector
dotnet add package Microsoft.ApplicationInsights.EventCounterCollector
```

### Basic Configuration

```json
// appsettings.json
{
  "ApplicationInsights": {
    "ConnectionString": "InstrumentationKey=your-instrumentation-key;IngestionEndpoint=https://your-region.in.applicationinsights.azure.com/",
    "EnableAdaptiveSampling": true,
    "EnableQuickPulseMetricStream": true,
    "EnableAuthenticationTrackingJavaScript": true
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    },
    "ApplicationInsights": {
      "LogLevel": {
        "Default": "Information",
        "Microsoft": "Warning"
      }
    }
  }
}
```

### Program.cs Configuration

```csharp
using Microsoft.ApplicationInsights.Extensibility;

var builder = WebApplication.CreateBuilder(args);

// Add Application Insights
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("ApplicationInsights");
    options.EnableAdaptiveSampling = true;
    options.EnableQuickPulseMetricStream = true;
    options.EnableAuthenticationTrackingJavaScript = true;
});

// Add custom telemetry initializers
builder.Services.AddSingleton<ITelemetryInitializer, CustomTelemetryInitializer>();

// Add telemetry processors
builder.Services.AddApplicationInsightsTelemetryProcessor<CustomTelemetryProcessor>();

var app = builder.Build();

// Enable Application Insights middleware
app.UseApplicationInsightsRequestTelemetry();

app.MapControllers();
app.Run();

// Custom Telemetry Initializer
public class CustomTelemetryInitializer : ITelemetryInitializer
{
    public void Initialize(ITelemetry telemetry)
    {
        telemetry.Context.Cloud.RoleName = "HotelManagement.API";
        telemetry.Context.Component.Version = "1.0.0";

        // Add custom properties
        if (telemetry is ISupportProperties supportProperties)
        {
            supportProperties.Properties["Environment"] = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Unknown";
            supportProperties.Properties["MachineName"] = Environment.MachineName;
        }
    }
}

// Custom Telemetry Processor
public class CustomTelemetryProcessor : ITelemetryProcessor
{
    private ITelemetryProcessor Next { get; set; }

    public CustomTelemetryProcessor(ITelemetryProcessor next)
    {
        Next = next;
    }

    public void Process(ITelemetry item)
    {
        // Filter out health check requests
        if (item is RequestTelemetry request &&
            request.Url?.AbsolutePath?.Contains("/health") == true)
        {
            return; // Don't process health check requests
        }

        // Add custom logic here
        if (item is DependencyTelemetry dependency)
        {
            // Mask sensitive data in SQL commands
            if (dependency.Type == "SQL" && !string.IsNullOrEmpty(dependency.Data))
            {
                dependency.Data = MaskSensitiveData(dependency.Data);
            }
        }

        Next.Process(item);
    }

    private string MaskSensitiveData(string data)
    {
        // Implement your masking logic here
        return data.Replace("password", "***");
    }
}
```

## Advanced Telemetry Collection

### Custom Telemetry Tracking Service

```csharp
// Services/TelemetryService.cs
public class TelemetryService
{
    private readonly TelemetryClient _telemetryClient;
    private readonly ILogger<TelemetryService> _logger;

    public TelemetryService(TelemetryClient telemetryClient, ILogger<TelemetryService> logger)
    {
        _telemetryClient = telemetryClient;
        _logger = logger;
    }

    public void TrackCustomEvent(string eventName, Dictionary<string, string> properties = null, Dictionary<string, double> metrics = null)
    {
        _telemetryClient.TrackEvent(eventName, properties, metrics);
    }

    public void TrackBusinessMetric(string metricName, double value, Dictionary<string, string> properties = null)
    {
        _telemetryClient.TrackMetric(metricName, value, properties);
    }

    public void TrackUserAction(string actionName, string userId, Dictionary<string, string> additionalProperties = null)
    {
        var properties = new Dictionary<string, string>
        {
            ["UserId"] = userId,
            ["Action"] = actionName,
            ["Timestamp"] = DateTime.UtcNow.ToString("O")
        };

        if (additionalProperties != null)
        {
            foreach (var prop in additionalProperties)
            {
                properties[prop.Key] = prop.Value;
            }
        }

        _telemetryClient.TrackEvent("UserAction", properties);
    }

    public IDisposable StartOperation(string operationName)
    {
        return _telemetryClient.StartOperation<RequestTelemetry>(operationName);
    }

    public void TrackDependency(string dependencyType, string target, string dependencyName,
        DateTime startTime, TimeSpan duration, bool success)
    {
        _telemetryClient.TrackDependency(dependencyType, target, dependencyName,
            startTime, duration, success);
    }
}
```

### Example 1: Controller Integration

```csharp
[ApiController]
[Route("api/[controller]")]
public class BookingController : ControllerBase
{
    private readonly IBookingService _bookingService;
    private readonly TelemetryService _telemetryService;

    public BookingController(IBookingService bookingService, TelemetryService telemetryService)
    {
        _bookingService = bookingService;
        _telemetryService = telemetryService;
    }

    [HttpPost]
    public async Task<IActionResult> CreateBooking([FromBody] CreateBookingRequest request)
    {
        using var operation = _telemetryService.StartOperation("CreateBooking");

        try
        {
            var booking = await _bookingService.CreateAsync(request);

            // Track business metrics
            _telemetryService.TrackBusinessMetric("BookingsCreated", 1, new Dictionary<string, string>
            {
                ["RoomType"] = request.RoomType,
                ["Duration"] = (request.CheckOut - request.CheckIn).TotalDays.ToString()
            });

            // Track user action
            _telemetryService.TrackUserAction("CreateBooking", request.CustomerId, new Dictionary<string, string>
            {
                ["BookingId"] = booking.Id.ToString(),
                ["RoomType"] = request.RoomType
            });

            return Ok(booking);
        }
        catch (Exception ex)
        {
            _telemetryService.TrackCustomEvent("BookingCreationFailed", new Dictionary<string, string>
            {
                ["Error"] = ex.Message,
                ["CustomerId"] = request.CustomerId
            });

            throw;
        }
    }
}
```

### Example 2: Orders Controller

```csharp
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

## Custom Telemetry Middleware

```csharp
public class ApplicationInsightsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly TelemetryClient _telemetryClient;

    public ApplicationInsightsMiddleware(RequestDelegate next, TelemetryClient telemetryClient)
    {
        _next = next;
        _telemetryClient = telemetryClient;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var startTime = DateTime.UtcNow;
        var stopwatch = Stopwatch.StartNew();

        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();

            // Track custom request telemetry
            var telemetry = new RequestTelemetry
            {
                Name = $"{context.Request.Method} {context.Request.Path}",
                Url = new Uri($"{context.Request.Scheme}://{context.Request.Host}{context.Request.Path}{context.Request.QueryString}"),
                Timestamp = startTime,
                Duration = stopwatch.Elapsed,
                ResponseCode = context.Response.StatusCode.ToString(),
                Success = context.Response.StatusCode < 400
            };

            // Add custom properties
            telemetry.Properties["UserAgent"] = context.Request.Headers["User-Agent"].ToString();
            telemetry.Properties["ClientIP"] = context.Connection.RemoteIpAddress?.ToString();

            _telemetryClient.TrackRequest(telemetry);
        }
    }
}
```

## Structured Logging Integration

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

## Health Checks Integration

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

## Use Cases

- Application performance monitoring and optimization
- Error tracking and exception analysis
- User behavior analytics and usage patterns
- Infrastructure monitoring and dependency tracking
- Real-time alerting on performance issues
- A/B testing and feature flag monitoring
- Business intelligence and custom metrics

## When to Use vs When Not to Use

### Use Application Insights when

- Azure-hosted applications requiring deep integration
- Comprehensive APM solution needed
- Integration with Azure Monitor ecosystem important
- Built-in analytics and alerting capabilities desired
- Automatic instrumentation for common frameworks preferred
- Building production applications requiring monitoring
- Need automated alerting and diagnostics
- Building business-critical applications

### Don't use when

- Multi-cloud monitoring strategy required
- Cost optimization for simple monitoring needs
- Existing monitoring infrastructure in place
- Real-time streaming analytics primary requirement
- Working primarily with non-Azure infrastructure
- Need open-source monitoring solutions
- Budget constraints for high-volume applications
- Simple applications with minimal monitoring needs
- Strict data residency requirements outside Azure regions

## Market Alternatives & Pros/Cons

### Alternatives

- **Azure**: Azure Monitor Logs, Azure Monitor Metrics
- **AWS**: CloudWatch, X-Ray
- **GCP**: Cloud Monitoring, Cloud Trace
- **Open Source**: Prometheus + Grafana, Jaeger, Zipkin, Elastic APM
- **Commercial**: New Relic, Datadog, Dynatrace

### Pros

- Deep integration with Azure services
- Automatic instrumentation for ASP.NET Core applications
- Rich analytics and query capabilities (KQL)
- Integration with Azure DevOps and GitHub
- Smart detection of performance anomalies
- Comprehensive dependency mapping
- Built-in alerting and dashboards
- Live metrics streaming
- Excellent .NET Core integration

### Cons

- Can be expensive for high-volume applications
- Data stored in Azure (potential compliance issues)
- Learning curve for advanced analytics queries (KQL)
- Data retention limits on different tiers
- Vendor lock-in to Azure ecosystem
- Limited customization compared to open-source alternatives
- Sampling can miss important low-frequency events

### Cost Considerations

- Pay-per-GB of data ingested (~$2.30/GB for first 5GB/month)
- First 5GB per month included with many Azure services
- Data retention: 90 days included, extended retention available
- Additional costs for multi-step web tests and alerts
- Enterprise features require higher-tier pricing

## Kusto Queries for Analysis

```kusto
// Performance analysis
requests
| where timestamp > ago(1h)
| summarize avg(duration), count() by name
| order by avg_duration desc

// Error analysis
exceptions
| where timestamp > ago(24h)
| summarize count() by type, outerMessage
| order by count_ desc

// Custom business metrics
customMetrics
| where name == "BookingValue"
| where timestamp > ago(7d)
| summarize sum(value), avg(value) by bin(timestamp, 1h)
| render timechart

// User behavior analysis
customEvents
| where name == "UserAction"
| where timestamp > ago(1d)
| summarize count() by tostring(customDimensions["Action"])
| order by count_ desc

// Dependency performance
dependencies
| where timestamp > ago(1h)
| summarize avg(duration), count(), successRate=avg(toint(success)) by name, type
| order by avg_duration desc
```
