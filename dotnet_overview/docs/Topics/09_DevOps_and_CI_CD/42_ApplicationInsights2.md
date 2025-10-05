## 42. Application Insights

### Short Introduction

Application Insights is Azure's application performance management (APM) service that provides real-time monitoring, diagnostics, and analytics for .NET applications. It automatically collects telemetry data including requests, dependencies, exceptions, and custom metrics, enabling developers to detect issues, optimize performance, and understand user behavior.

### Official Definition

Application Insights is a feature of Azure Monitor that provides extensible application performance management (APM) and monitoring for live web applications. It works for apps on a wide variety of platforms including .NET, Node.js, Java, and Python hosted on-premises, hybrid, or any public cloud.

### Setup/Usage with .NET 8+ Code

**Installation and Basic Setup:**

```bash
# Install Application Insights SDK
dotnet add package Microsoft.ApplicationInsights.AspNetCore
dotnet add package Microsoft.ApplicationInsights.PerfCounterCollector
dotnet add package Microsoft.ApplicationInsights.EventCounterCollector
```

**Program.cs Configuration:**

```csharp
// Program.cs
using Microsoft.ApplicationInsights.Extensibility;

var builder = WebApplication.CreateBuilder(args);

// Add Application Insights
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("ApplicationInsights");
    options.EnableAdaptiveSampling = true;
    options.EnableQuickPulseMetricStream = true;
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

**Advanced Telemetry Collection:**

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

// Controllers/BookingController.cs
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

**Configuration (appsettings.json):**

```json
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

### Use Cases

- **Performance Monitoring**: Track response times, throughput, and system resources
- **Error Tracking**: Monitor exceptions and failures with detailed stack traces
- **User Analytics**: Understand user behavior and application usage patterns
- **Business Intelligence**: Track custom business metrics and KPIs
- **Dependency Monitoring**: Monitor external service calls and database queries
- **Real-time Monitoring**: Live metrics and alerts for immediate issue detection

### When to Use vs When Not to Use

**Use Application Insights when:**

- Building production applications requiring monitoring
- Need comprehensive APM capabilities
- Working within Azure ecosystem
- Require real-time performance insights
- Need automated alerting and diagnostics
- Building business-critical applications

**Consider alternatives when:**

- Working primarily with non-Azure infrastructure
- Need open-source monitoring solutions
- Budget constraints for high-volume applications
- Simple applications with minimal monitoring needs
- Strict data residency requirements outside Azure regions

### Market Alternatives & Pros/Cons

**Alternatives:**

- **New Relic**: Comprehensive APM with good .NET support
- **Datadog**: Multi-platform monitoring with strong integrations
- **Dynatrace**: AI-powered full-stack monitoring
- **Elastic APM**: Open-source APM solution
- **Prometheus + Grafana**: Open-source metrics and monitoring
- **AWS X-Ray**: Native AWS application tracing

**Pros:**

- Deep integration with Azure services
- Automatic dependency tracking
- Rich querying capabilities with KQL
- Built-in alerting and dashboards
- Live metrics streaming
- Excellent .NET Core integration

**Cons:**

- Can be expensive for high-volume applications
- Data stored in Azure (potential compliance issues)
- Learning curve for KQL queries
- Limited customization compared to open-source alternatives
- Sampling can miss important low-frequency events

### Complete Runnable Sample

**Complete Application Insights Implementation:**

```csharp
// Program.cs - Complete setup
using Microsoft.ApplicationInsights.Extensibility;
using Microsoft.ApplicationInsights.DependencyCollector;

var builder = WebApplication.CreateBuilder(args);

// Configure Application Insights
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("ApplicationInsights");
    options.EnableAdaptiveSampling = false; // Disable for demo, enable in production
    options.EnableQuickPulseMetricStream = true;
    options.EnableAuthenticationTrackingJavaScript = true;
});

// Configure telemetry
builder.Services.Configure<TelemetryConfiguration>(config =>
{
    config.DefaultTelemetrySink.TelemetryProcessorChainBuilder
        .Use((next) => new CustomTelemetryProcessor(next))
        .Build();
});

// Add custom services
builder.Services.AddSingleton<ITelemetryInitializer, CustomTelemetryInitializer>();
builder.Services.AddScoped<TelemetryService>();
builder.Services.AddScoped<IBookingService, BookingService>();

// Add other services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseRouting();
app.MapControllers();

// Add health checks with Application Insights
app.MapHealthChecks("/health");

app.Run();

// Services/BookingService.cs
public interface IBookingService
{
    Task<Booking> CreateAsync(CreateBookingRequest request);
    Task<Booking> GetByIdAsync(int id);
    Task<List<Booking>> GetUserBookingsAsync(string userId);
}

public class BookingService : IBookingService
{
    private readonly TelemetryClient _telemetryClient;
    private readonly ILogger<BookingService> _logger;
    private readonly HttpClient _httpClient;

    public BookingService(TelemetryClient telemetryClient, ILogger<BookingService> logger, HttpClient httpClient)
    {
        _telemetryClient = telemetryClient;
        _logger = logger;
        _httpClient = httpClient;
    }

    public async Task<Booking> CreateAsync(CreateBookingRequest request)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            // Simulate external API call
            var startTime = DateTime.UtcNow;
            var response = await _httpClient.GetAsync("https://api.external-service.com/validate");
            stopwatch.Stop();

            // Track dependency
            _telemetryClient.TrackDependency("HTTP", "api.external-service.com",
                "validate", startTime, stopwatch.Elapsed, response.IsSuccessStatusCode);

            // Simulate booking creation
            var booking = new Booking
            {
                Id = Random.Shared.Next(1000, 9999),
                CustomerId = request.CustomerId,
                RoomType = request.RoomType,
                CheckIn = request.CheckIn,
                CheckOut = request.CheckOut,
                Status = "Confirmed"
            };

            // Track custom metrics
            _telemetryClient.TrackMetric("BookingValue",
                (request.CheckOut - request.CheckIn).TotalDays * 150, // Assuming $150/night
                new Dictionary<string, string>
                {
                    ["RoomType"] = request.RoomType,
                    ["CustomerId"] = request.CustomerId
                });

            _logger.LogInformation("Booking created successfully with ID {BookingId}", booking.Id);

            return booking;
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex, new Dictionary<string, string>
            {
                ["Operation"] = "CreateBooking",
                ["CustomerId"] = request.CustomerId,
                ["RoomType"] = request.RoomType
            });

            throw;
        }
    }

    public async Task<Booking> GetByIdAsync(int id)
    {
        using var operation = _telemetryClient.StartOperation<DependencyTelemetry>("GetBooking");
        operation.Telemetry.Type = "Database";
        operation.Telemetry.Target = "BookingDatabase";

        try
        {
            // Simulate database call
            await Task.Delay(50);

            var booking = new Booking
            {
                Id = id,
                CustomerId = "CUST001",
                RoomType = "Deluxe",
                CheckIn = DateTime.Today.AddDays(1),
                CheckOut = DateTime.Today.AddDays(3),
                Status = "Confirmed"
            };

            operation.Telemetry.Success = true;
            return booking;
        }
        catch (Exception ex)
        {
            operation.Telemetry.Success = false;
            _telemetryClient.TrackException(ex);
            throw;
        }
    }

    public async Task<List<Booking>> GetUserBookingsAsync(string userId)
    {
        var properties = new Dictionary<string, string> { ["UserId"] = userId };
        var metrics = new Dictionary<string, double>();

        using var operation = _telemetryClient.StartOperation<RequestTelemetry>("GetUserBookings");

        try
        {
            // Simulate database query
            await Task.Delay(100);

            var bookings = new List<Booking>
            {
                new() { Id = 1, CustomerId = userId, RoomType = "Standard", Status = "Confirmed" },
                new() { Id = 2, CustomerId = userId, RoomType = "Deluxe", Status = "Pending" }
            };

            metrics["BookingCount"] = bookings.Count;
            _telemetryClient.TrackEvent("UserBookingsRetrieved", properties, metrics);

            return bookings;
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex, properties);
            throw;
        }
    }
}

// Models/Booking.cs
public class Booking
{
    public int Id { get; set; }
    public string CustomerId { get; set; } = string.Empty;
    public string RoomType { get; set; } = string.Empty;
    public DateTime CheckIn { get; set; }
    public DateTime CheckOut { get; set; }
    public string Status { get; set; } = string.Empty;
}

public class CreateBookingRequest
{
    public string CustomerId { get; set; } = string.Empty;
    public string RoomType { get; set; } = string.Empty;
    public DateTime CheckIn { get; set; }
    public DateTime CheckOut { get; set; }
}

// Middleware/ApplicationInsightsMiddleware.cs
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

**Kusto Queries for Analysis:**

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
