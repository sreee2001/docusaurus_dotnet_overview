## 43. Health Checks

### Short Introduction

Health Checks in .NET provide a systematic way to monitor the health and availability of an application and its dependencies. They offer HTTP endpoints that can be called by load balancers, monitoring systems, and orchestrators to determine if an application instance should receive traffic or needs to be restarted.

### Official Definition

Health Checks are diagnostic checks that can be performed on an application to determine its health status. They provide a consistent way to expose health information about an application and its dependencies, enabling monitoring systems to make informed decisions about traffic routing and instance management.

### Setup/Usage with .NET 8+ Code

**Basic Health Checks Setup:**

```csharp
// Program.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;

var builder = WebApplication.CreateBuilder(args);

// Add basic health checks
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy("Application is running"))
    .AddCheck<DatabaseHealthCheck>("database")
    .AddCheck<RedisHealthCheck>("redis")
    .AddCheck<ExternalApiHealthCheck>("external-api");

// Add health checks UI (optional)
builder.Services.AddHealthChecksUI(options =>
{
    options.SetEvaluationTimeInSeconds(30);
    options.MaximumHistoryEntriesPerEndpoint(50);
    options.AddHealthCheckEndpoint("Hotel Management API", "/health");
}).AddInMemoryStorage();

var app = builder.Build();

// Configure health check endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = WriteHealthCheckResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = WriteHealthCheckResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false, // Only checks with no dependencies
    ResponseWriter = WriteHealthCheckResponse
});

// Add health checks UI
app.MapHealthChecksUI(options =>
{
    options.UIPath = "/health-ui";
    options.ApiPath = "/health-api";
});

app.Run();

// Custom response writer
static async Task WriteHealthCheckResponse(HttpContext context, HealthReport report)
{
    context.Response.ContentType = "application/json";

    var result = new
    {
        status = report.Status.ToString(),
        totalDuration = report.TotalDuration.TotalMilliseconds,
        checks = report.Entries.Select(entry => new
        {
            name = entry.Key,
            status = entry.Value.Status.ToString(),
            duration = entry.Value.Duration.TotalMilliseconds,
            description = entry.Value.Description,
            data = entry.Value.Data,
            exception = entry.Value.Exception?.Message,
            tags = entry.Value.Tags
        })
    };

    await context.Response.WriteAsync(JsonSerializer.Serialize(result, new JsonSerializerOptions
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        WriteIndented = true
    }));
}
```

**Custom Health Checks Implementation:**

```csharp
// HealthChecks/DatabaseHealthCheck.cs
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<DatabaseHealthCheck> _logger;

    public DatabaseHealthCheck(IConfiguration configuration, ILogger<DatabaseHealthCheck> logger)
    {
        _configuration = configuration;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            var connectionString = _configuration.GetConnectionString("DefaultConnection");

            using var connection = new SqlConnection(connectionString);
            await connection.OpenAsync(cancellationToken);

            using var command = connection.CreateCommand();
            command.CommandText = "SELECT 1";
            command.CommandTimeout = 5; // 5 second timeout

            var result = await command.ExecuteScalarAsync(cancellationToken);

            var data = new Dictionary<string, object>
            {
                ["server"] = connection.DataSource,
                ["database"] = connection.Database,
                ["connection_time"] = DateTime.UtcNow
            };

            return result?.ToString() == "1"
                ? HealthCheckResult.Healthy("Database connection successful", data)
                : HealthCheckResult.Unhealthy("Database query failed", null, data);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database health check failed");

            return HealthCheckResult.Unhealthy("Database connection failed", ex, new Dictionary<string, object>
            {
                ["error"] = ex.Message,
                ["check_time"] = DateTime.UtcNow
            });
        }
    }
}

// HealthChecks/RedisHealthCheck.cs
public class RedisHealthCheck : IHealthCheck
{
    private readonly IConnectionMultiplexer _connectionMultiplexer;
    private readonly ILogger<RedisHealthCheck> _logger;

    public RedisHealthCheck(IConnectionMultiplexer connectionMultiplexer, ILogger<RedisHealthCheck> logger)
    {
        _connectionMultiplexer = connectionMultiplexer;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            var database = _connectionMultiplexer.GetDatabase();
            var key = $"health_check_{Guid.NewGuid()}";
            var value = "ping";

            // Test write
            await database.StringSetAsync(key, value, TimeSpan.FromSeconds(30));

            // Test read
            var retrievedValue = await database.StringGetAsync(key);

            // Clean up
            await database.KeyDeleteAsync(key);

            var data = new Dictionary<string, object>
            {
                ["server"] = _connectionMultiplexer.Configuration,
                ["is_connected"] = _connectionMultiplexer.IsConnected,
                ["check_time"] = DateTime.UtcNow
            };

            return retrievedValue == value
                ? HealthCheckResult.Healthy("Redis connection successful", data)
                : HealthCheckResult.Unhealthy("Redis read/write test failed", null, data);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Redis health check failed");

            return HealthCheckResult.Unhealthy("Redis connection failed", ex, new Dictionary<string, object>
            {
                ["error"] = ex.Message,
                ["check_time"] = DateTime.UtcNow
            });
        }
    }
}

// HealthChecks/ExternalApiHealthCheck.cs
public class ExternalApiHealthCheck : IHealthCheck
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _configuration;
    private readonly ILogger<ExternalApiHealthCheck> _logger;

    public ExternalApiHealthCheck(HttpClient httpClient, IConfiguration configuration, ILogger<ExternalApiHealthCheck> logger)
    {
        _httpClient = httpClient;
        _configuration = configuration;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            var apiUrl = _configuration["ExternalApi:BaseUrl"];
            var healthEndpoint = $"{apiUrl}/health";

            using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
            cts.CancelAfter(TimeSpan.FromSeconds(10)); // 10 second timeout

            var response = await _httpClient.GetAsync(healthEndpoint, cts.Token);
            var responseTime = DateTime.UtcNow;

            var data = new Dictionary<string, object>
            {
                ["url"] = healthEndpoint,
                ["status_code"] = (int)response.StatusCode,
                ["response_time"] = responseTime,
                ["is_success"] = response.IsSuccessStatusCode
            };

            if (response.IsSuccessStatusCode)
            {
                var content = await response.Content.ReadAsStringAsync(cancellationToken);
                data["response_content"] = content.Length > 200 ? content[..200] + "..." : content;

                return HealthCheckResult.Healthy($"External API is healthy (Status: {response.StatusCode})", data);
            }
            else
            {
                return HealthCheckResult.Degraded($"External API returned {response.StatusCode}", null, data);
            }
        }
        catch (TaskCanceledException ex) when (ex.InnerException is TimeoutException)
        {
            _logger.LogWarning("External API health check timed out");
            return HealthCheckResult.Unhealthy("External API health check timed out", ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "External API health check failed");
            return HealthCheckResult.Unhealthy("External API health check failed", ex);
        }
    }
}

// HealthChecks/MemoryHealthCheck.cs
public class MemoryHealthCheck : IHealthCheck
{
    private readonly ILogger<MemoryHealthCheck> _logger;
    private const long ThresholdBytes = 1024 * 1024 * 1024; // 1 GB

    public MemoryHealthCheck(ILogger<MemoryHealthCheck> logger)
    {
        _logger = logger;
    }

    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            var process = Process.GetCurrentProcess();
            var workingSet = process.WorkingSet64;
            var gcTotalMemory = GC.GetTotalMemory(false);

            var data = new Dictionary<string, object>
            {
                ["working_set_mb"] = workingSet / (1024 * 1024),
                ["gc_total_memory_mb"] = gcTotalMemory / (1024 * 1024),
                ["gen0_collections"] = GC.CollectionCount(0),
                ["gen1_collections"] = GC.CollectionCount(1),
                ["gen2_collections"] = GC.CollectionCount(2),
                ["check_time"] = DateTime.UtcNow
            };

            if (workingSet > ThresholdBytes)
            {
                return Task.FromResult(HealthCheckResult.Degraded(
                    $"Memory usage is high: {workingSet / (1024 * 1024)} MB", null, data));
            }

            return Task.FromResult(HealthCheckResult.Healthy(
                $"Memory usage is normal: {workingSet / (1024 * 1024)} MB", data));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Memory health check failed");
            return Task.FromResult(HealthCheckResult.Unhealthy("Memory health check failed", ex));
        }
    }
}
```

### Use Cases

- **Load Balancer Integration**: Determine which instances should receive traffic
- **Container Orchestration**: Kubernetes liveness and readiness probes
- **Monitoring Systems**: Automated health monitoring and alerting
- **Deployment Strategies**: Blue-green and rolling deployments
- **Auto-scaling**: Scale up/down based on application health
- **Circuit Breaker Patterns**: Fail fast when dependencies are unhealthy

### When to Use vs When Not to Use

**Use Health Checks when:**

- Running in containerized or orchestrated environments
- Using load balancers for traffic distribution
- Need automated monitoring and alerting
- Implementing high-availability architectures
- Running microservices that depend on external services
- Need visibility into application and dependency health

**Consider alternatives when:**

- Simple applications with no external dependencies
- Development or testing environments only
- Applications that don't require high availability
- Legacy systems that can't be easily modified
- When basic uptime monitoring is sufficient

### Market Alternatives & Pros/Cons

**Alternatives:**

- **Custom HTTP endpoints**: Simple health status endpoints
- **Application Performance Monitoring**: New Relic, Datadog health monitoring
- **Infrastructure monitoring**: Nagios, Zabbix, Prometheus
- **Cloud provider health checks**: AWS ELB, Azure Load Balancer
- **Service mesh health checks**: Istio, Linkerd health monitoring

**Pros:**

- Built into .NET ecosystem
- Standardized approach across applications
- Integration with monitoring and orchestration systems
- Customizable and extensible
- Minimal performance overhead
- Rich reporting capabilities

**Cons:**

- Requires proper implementation to be effective
- Can add complexity to simple applications
- Health check failures might not always indicate real issues
- Requires monitoring system integration for full value
- Potential security considerations for exposed endpoints

### Complete Runnable Sample

**Production-Ready Health Checks Implementation:**

```csharp
// Program.cs - Complete health checks setup
using Microsoft.Extensions.Diagnostics.HealthChecks;
using System.Text.Json;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddHttpClient<ExternalApiHealthCheck>();

// Configure health checks with different tags
builder.Services.AddHealthChecks()
    // Readiness checks - required for the app to be ready to serve traffic
    .AddCheck<DatabaseHealthCheck>("database", HealthStatus.Unhealthy, new[] { "ready", "database" })
    .AddCheck<RedisHealthCheck>("redis", HealthStatus.Unhealthy, new[] { "ready", "cache" })

    // Liveness checks - required for the app to be considered alive
    .AddCheck("self", () => HealthCheckResult.Healthy("Application is running"), new[] { "live" })
    .AddCheck<MemoryHealthCheck>("memory", HealthStatus.Degraded, new[] { "live", "memory" })

    // Optional dependency checks - can be degraded without failing
    .AddCheck<ExternalApiHealthCheck>("external-api", HealthStatus.Degraded, new[] { "external" })
    .AddCheck("disk-space", () =>
    {
        var drives = DriveInfo.GetDrives().Where(d => d.IsReady);
        var criticalDrives = drives.Where(d => d.AvailableFreeSpace < d.TotalSize * 0.1);

        if (criticalDrives.Any())
        {
            return HealthCheckResult.Degraded($"Low disk space on: {string.Join(", ", criticalDrives.Select(d => d.Name))}");
        }

        return HealthCheckResult.Healthy("Disk space is adequate");
    }, new[] { "infrastructure" });

// Add health checks UI for development
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddHealthChecksUI(options =>
    {
        options.SetEvaluationTimeInSeconds(15);
        options.MaximumHistoryEntriesPerEndpoint(100);
        options.AddHealthCheckEndpoint("API Health", "/health");
        options.AddHealthCheckEndpoint("Ready Check", "/health/ready");
        options.AddHealthCheckEndpoint("Live Check", "/health/live");
    }).AddInMemoryStorage();
}

var app = builder.Build();

// Configure health check endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = WriteDetailedHealthCheckResponse,
    AllowCachingResponses = false
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = WriteDetailedHealthCheckResponse,
    AllowCachingResponses = false
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResponseWriter = WriteSimpleHealthCheckResponse,
    AllowCachingResponses = false
});

// Kubernetes-style probes
app.MapHealthChecks("/readyz", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = WriteSimpleHealthCheckResponse
});

app.MapHealthChecks("/livez", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResponseWriter = WriteSimpleHealthCheckResponse
});

// Health checks UI (development only)
if (app.Environment.IsDevelopment())
{
    app.MapHealthChecksUI(options => options.UIPath = "/health-ui");
}

app.MapControllers();
app.Run();

// Detailed response writer
static async Task WriteDetailedHealthCheckResponse(HttpContext context, HealthReport report)
{
    context.Response.ContentType = "application/json";

    var response = new
    {
        status = report.Status.ToString(),
        totalDuration = $"{report.TotalDuration.TotalMilliseconds:F2}ms",
        timestamp = DateTime.UtcNow.ToString("O"),
        checks = report.Entries.ToDictionary(
            kvp => kvp.Key,
            kvp => new
            {
                status = kvp.Value.Status.ToString(),
                duration = $"{kvp.Value.Duration.TotalMilliseconds:F2}ms",
                description = kvp.Value.Description,
                data = kvp.Value.Data,
                exception = kvp.Value.Exception?.Message,
                tags = kvp.Value.Tags
            })
    };

    var json = JsonSerializer.Serialize(response, new JsonSerializerOptions
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        WriteIndented = true
    });

    await context.Response.WriteAsync(json);
}

// Simple response writer for load balancers
static async Task WriteSimpleHealthCheckResponse(HttpContext context, HealthReport report)
{
    context.Response.ContentType = "text/plain";

    var status = report.Status switch
    {
        HealthStatus.Healthy => "Healthy",
        HealthStatus.Degraded => "Degraded",
        HealthStatus.Unhealthy => "Unhealthy",
        _ => "Unknown"
    };

    await context.Response.WriteAsync(status);
}

// Docker health check script
// docker-healthcheck.sh
#!/bin/bash
curl -f http://localhost:8080/health/live || exit 1

// Kubernetes deployment with health checks
// k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hotel-management-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hotel-management-api
  template:
    metadata:
      labels:
        app: hotel-management-api
    spec:
      containers:
      - name: api
        image: hotel-management-api:latest
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /livez
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 30
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```
