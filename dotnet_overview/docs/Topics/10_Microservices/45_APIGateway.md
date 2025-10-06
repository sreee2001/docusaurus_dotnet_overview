## 45. API Gateway (YARP, Azure API Management)

## Short Introduction

An API Gateway is a server that acts as an API front-end, receiving API requests, enforcing throttling and security policies, passing requests to the back-end service, and then passing the response back to the requester. YARP (Yet Another Reverse Proxy) is Microsoft's .NET-based reverse proxy toolkit, while Azure API Management provides a comprehensive managed API gateway solution.

## Official Definition

An API Gateway is an API management tool that sits between a client and a collection of backend services. It acts as a reverse proxy to accept all API calls, aggregate the various services required to fulfill them, and return the appropriate result. YARP is a library to help create reverse proxy servers that are high-performance, production-ready, and highly customizable.

## Setup/Usage with .NET 8+ Code

**YARP Implementation:**

```bash
# Install YARP package
dotnet add package Yarp.ReverseProxy
```

**Program.cs with YARP:**

```csharp
// ApiGateway/Program.cs
using Yarp.ReverseProxy.Configuration;
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);

// Add YARP
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// Add authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireAdmin", policy => policy.RequireClaim("role", "admin"));
    options.AddPolicy("RequireUser", policy => policy.RequireAuthenticatedUser());
});

// Add rate limiting
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User?.Identity?.Name ?? context.Request.Headers.Host.ToString(),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));
});

// Add CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

var app = builder.Build();

// Configure pipeline
app.UseCors("AllowAll");
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

// Map reverse proxy
app.MapReverseProxy();

app.Run();
```

**YARP Configuration (appsettings.json):**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Yarp": "Warning"
    }
  },
  "ReverseProxy": {
    "Routes": {
      "booking-route": {
        "ClusterId": "booking-cluster",
        "Match": {
          "Path": "/api/bookings/{**catch-all}"
        },
        "Transforms": [
          {
            "PathPattern": "/api/bookings/{**catch-all}"
          },
          {
            "RequestHeader": "X-Forwarded-Host",
            "Set": "{OriginalHost}"
          }
        ],
        "AuthorizationPolicy": "RequireUser"
      },
      "room-route": {
        "ClusterId": "room-cluster",
        "Match": {
          "Path": "/api/rooms/{**catch-all}"
        },
        "Transforms": [
          {
            "PathPattern": "/api/rooms/{**catch-all}"
          }
        ]
      },
      "admin-route": {
        "ClusterId": "admin-cluster",
        "Match": {
          "Path": "/api/admin/{**catch-all}"
        },
        "AuthorizationPolicy": "RequireAdmin",
        "RateLimiterPolicy": "AdminPolicy"
      }
    },
    "Clusters": {
      "booking-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "http://booking-service:8080/"
          }
        },
        "HealthCheck": {
          "Active": {
            "Enabled": "true",
            "Interval": "00:00:30",
            "Timeout": "00:00:05",
            "Policy": "ConsecutiveFailures",
            "Path": "/health"
          }
        },
        "LoadBalancingPolicy": "RoundRobin"
      },
      "room-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "http://room-service:8080/"
          },
          "destination2": {
            "Address": "http://room-service-2:8080/"
          }
        },
        "HealthCheck": {
          "Active": {
            "Enabled": "true",
            "Interval": "00:00:30",
            "Timeout": "00:00:05",
            "Policy": "ConsecutiveFailures",
            "Path": "/health"
          }
        },
        "LoadBalancingPolicy": "LeastRequests"
      },
      "admin-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "http://admin-service:8080/"
          }
        }
      }
    }
  }
}
```

**Custom YARP Middleware:**

```csharp
// Middleware/ApiGatewayMiddleware.cs
public class ApiGatewayMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ApiGatewayMiddleware> _logger;

    public ApiGatewayMiddleware(RequestDelegate next, ILogger<ApiGatewayMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Add correlation ID
        if (!context.Request.Headers.ContainsKey("X-Correlation-ID"))
        {
            context.Request.Headers.Add("X-Correlation-ID", Guid.NewGuid().ToString());
        }

        // Log request
        _logger.LogInformation("Gateway request: {Method} {Path} from {RemoteIp}",
            context.Request.Method,
            context.Request.Path,
            context.Connection.RemoteIpAddress);

        // Add request timestamp
        context.Request.Headers.Add("X-Gateway-Timestamp", DateTimeOffset.UtcNow.ToString("O"));

        await _next(context);

        // Log response
        _logger.LogInformation("Gateway response: {StatusCode} for {Method} {Path}",
            context.Response.StatusCode,
            context.Request.Method,
            context.Request.Path);
    }
}

// Transforms/CustomTransforms.cs
public class AddCustomHeaderTransform : ITransformProvider
{
    public void ValidateRoute(RouteConfig route, IList<Exception> errors)
    {
        // Validation logic if needed
    }

    public void ValidateCluster(ClusterConfig cluster, IList<Exception> errors)
    {
        // Validation logic if needed
    }

    public void Apply(TransformBuilderContext context)
    {
        // Add custom transforms
        context.AddRequestTransform(transformContext =>
        {
            transformContext.ProxyRequest.Headers.Add("X-Gateway-Version", "1.0");
            transformContext.ProxyRequest.Headers.Add("X-Request-ID", Guid.NewGuid().ToString());
            return ValueTask.CompletedTask;
        });

        context.AddResponseTransform(transformContext =>
        {
            transformContext.HttpContext.Response.Headers.Add("X-Powered-By", "YARP Gateway");
            return ValueTask.CompletedTask;
        });
    }
}
```

**Azure API Management Integration:**

```csharp
// For comparison - Azure API Management would be configured via ARM template
// azuredeploy.json (excerpt)
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "apiManagementServiceName": {
      "type": "string",
      "defaultValue": "hotel-management-apim"
    }
  },
  "resources": [
    {
      "type": "Microsoft.ApiManagement/service",
      "apiVersion": "2021-08-01",
      "name": "[parameters('apiManagementServiceName')]",
      "location": "[resourceGroup().location]",
      "sku": {
        "name": "Developer",
        "capacity": 1
      },
      "properties": {
        "publisherEmail": "admin@hotelmanagement.com",
        "publisherName": "Hotel Management",
        "notificationSenderEmail": "apimgmt-noreply@mail.windowsazure.com"
      }
    },
    {
      "type": "Microsoft.ApiManagement/service/apis",
      "apiVersion": "2021-08-01",
      "name": "[concat(parameters('apiManagementServiceName'), '/booking-api')]",
      "dependsOn": [
        "[resourceId('Microsoft.ApiManagement/service', parameters('apiManagementServiceName'))]"
      ],
      "properties": {
        "displayName": "Booking API",
        "apiRevision": "1",
        "subscriptionRequired": true,
        "serviceUrl": "http://booking-service:8080",
        "path": "bookings",
        "protocols": ["https"]
      }
    }
  ]
}
```

### Use Cases

- **Single Entry Point**: Centralized access to multiple microservices
- **Cross-Cutting Concerns**: Authentication, logging, rate limiting, CORS
- **API Versioning**: Route requests to different API versions
- **Load Balancing**: Distribute requests across multiple service instances
- **Protocol Translation**: Convert between different protocols (HTTP, gRPC)
- **Request/Response Transformation**: Modify requests and responses

## When to Use vs When Not to Use

**Use API Gateway when:**

- Running microservices architecture
- Need centralized authentication and authorization
- Require request routing and load balancing
- Need to implement cross-cutting concerns
- Want to hide internal service complexity from clients
- Need API versioning and backwards compatibility

**Consider alternatives when:**

- Simple monolithic applications
- Direct service-to-service communication is preferred
- Low latency requirements (additional network hop)
- Limited operational complexity tolerance
- Small number of services with simple routing needs

### Market Alternatives & Pros/Cons

### Alternatives:

- **Kong**: Open-source API gateway with enterprise features
- **Istio**: Service mesh with gateway capabilities
- **Ambassador**: Kubernetes-native API gateway
- **AWS API Gateway**: Managed API gateway service
- **Traefik**: Modern reverse proxy and load balancer
- **Nginx Plus**: Commercial version with API gateway features

### Pros:

- Centralized cross-cutting concerns
- Single entry point for clients
- Built-in load balancing and health checks
- Request/response transformation capabilities
- Integration with .NET ecosystem (YARP)
- Reduced client complexity

### Cons:

- Additional network latency
- Single point of failure risk
- Operational complexity
- Potential bottleneck
- Additional infrastructure component to manage

### Complete Runnable Sample

**Production-Ready YARP Gateway:**

```csharp
// ApiGateway/Program.cs - Complete implementation
using Yarp.ReverseProxy.Configuration;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// Add authentication and authorization
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
        options.RequireHttpsMetadata = !builder.Environment.IsDevelopment();
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireAdmin", policy =>
        policy.RequireClaim("role", "admin"));
    options.AddPolicy("RequireUser", policy =>
        policy.RequireAuthenticatedUser());
    options.AddPolicy("RequireCustomer", policy =>
        policy.RequireClaim("role", "customer"));
});

// Add rate limiting with different policies
builder.Services.AddRateLimiter(options =>
{
    // Global rate limiter
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: GetPartitionKey(context),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 1000,
                Window = TimeSpan.FromMinutes(1)
            }));

    // Admin policy - higher limits
    options.AddPolicy("AdminPolicy", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: GetPartitionKey(context),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 10000,
                Window = TimeSpan.FromMinutes(1)
            }));

    // Customer policy - standard limits
    options.AddPolicy("CustomerPolicy", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: GetPartitionKey(context),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));

    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        await context.HttpContext.Response.WriteAsync("Rate limit exceeded. Try again later.", token);
    };
});

// Add CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("HotelManagementCors", policy =>
    {
        policy.WithOrigins(
                builder.Configuration.GetSection("Cors:AllowedOrigins").Get<string[]>() ?? Array.Empty<string>())
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

// Add health checks
builder.Services.AddHealthChecks();

// Add custom services
builder.Services.AddSingleton<ITransformProvider, AddCustomHeaderTransform>();
builder.Services.AddScoped<ApiKeyValidationService>();

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseCors("HotelManagementCors");

// Custom middleware
app.UseMiddleware<ApiGatewayMiddleware>();
app.UseMiddleware<ApiKeyValidationMiddleware>();

app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

// Health checks
app.MapHealthChecks("/health");

// Map reverse proxy
app.MapReverseProxy(proxyPipeline =>
{
    proxyPipeline.Use(async (context, next) =>
    {
        // Custom proxy middleware
        var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault() ?? Guid.NewGuid().ToString();
        context.Request.Headers["X-Correlation-ID"] = correlationId;

        await next();

        context.Response.Headers["X-Correlation-ID"] = correlationId;
    });
});

app.Run();

// Helper method for rate limiting partition key
static string GetPartitionKey(HttpContext context)
{
    return context.User?.Identity?.Name ??
           context.Request.Headers["X-API-Key"].FirstOrDefault() ??
           context.Connection.RemoteIpAddress?.ToString() ??
           "anonymous";
}

// Services/ApiKeyValidationService.cs
public class ApiKeyValidationService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<ApiKeyValidationService> _logger;
    private readonly Dictionary<string, ApiKeyInfo> _validApiKeys;

    public ApiKeyValidationService(IConfiguration configuration, ILogger<ApiKeyValidationService> logger)
    {
        _configuration = configuration;
        _logger = logger;
        _validApiKeys = LoadApiKeysFromConfiguration();
    }

    public bool IsValidApiKey(string apiKey, out ApiKeyInfo? keyInfo)
    {
        keyInfo = null;
        if (string.IsNullOrEmpty(apiKey))
            return false;

        if (_validApiKeys.TryGetValue(apiKey, out keyInfo))
        {
            if (keyInfo.ExpiresAt > DateTime.UtcNow)
            {
                return true;
            }
            else
            {
                _logger.LogWarning("Expired API key used: {ApiKeyId}", keyInfo.Id);
            }
        }
        else
        {
            _logger.LogWarning("Invalid API key used: {ApiKey}", apiKey[..8] + "...");
        }

        return false;
    }

    private Dictionary<string, ApiKeyInfo> LoadApiKeysFromConfiguration()
    {
        var apiKeys = new Dictionary<string, ApiKeyInfo>();
        var apiKeySection = _configuration.GetSection("ApiKeys");

        foreach (var section in apiKeySection.GetChildren())
        {
            var keyInfo = new ApiKeyInfo
            {
                Id = section["Id"] ?? string.Empty,
                Name = section["Name"] ?? string.Empty,
                Key = section["Key"] ?? string.Empty,
                ExpiresAt = DateTime.Parse(section["ExpiresAt"] ?? DateTime.MaxValue.ToString()),
                Scopes = section.GetSection("Scopes").Get<string[]>() ?? Array.Empty<string>()
            };

            if (!string.IsNullOrEmpty(keyInfo.Key))
            {
                apiKeys[keyInfo.Key] = keyInfo;
            }
        }

        return apiKeys;
    }
}

public class ApiKeyInfo
{
    public string Id { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public string Key { get; set; } = string.Empty;
    public DateTime ExpiresAt { get; set; }
    public string[] Scopes { get; set; } = Array.Empty<string>();
}

// Middleware/ApiKeyValidationMiddleware.cs
public class ApiKeyValidationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ApiKeyValidationService _apiKeyService;
    private readonly ILogger<ApiKeyValidationMiddleware> _logger;

    public ApiKeyValidationMiddleware(
        RequestDelegate next,
        ApiKeyValidationService apiKeyService,
        ILogger<ApiKeyValidationMiddleware> logger)
    {
        _next = next;
        _apiKeyService = apiKeyService;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Skip API key validation for health checks and auth endpoints
        if (context.Request.Path.StartsWithSegments("/health") ||
            context.Request.Path.StartsWithSegments("/auth"))
        {
            await _next(context);
            return;
        }

        var apiKey = context.Request.Headers["X-API-Key"].FirstOrDefault();

        if (!string.IsNullOrEmpty(apiKey))
        {
            if (_apiKeyService.IsValidApiKey(apiKey, out var keyInfo))
            {
                // Add API key info to context for downstream services
                context.Items["ApiKeyInfo"] = keyInfo;
                context.Request.Headers["X-API-Key-Id"] = keyInfo!.Id;
                context.Request.Headers["X-API-Key-Name"] = keyInfo.Name;

                _logger.LogInformation("Valid API key used: {ApiKeyName}", keyInfo.Name);
            }
            else
            {
                context.Response.StatusCode = 401;
                await context.Response.WriteAsync("Invalid API key");
                return;
            }
        }
        // If no API key provided, let it pass through for JWT validation

        await _next(context);
    }
}

// docker-compose.yml for complete setup
version: '3.8'
services:
  api-gateway:
    build:
      context: ./ApiGateway
      dockerfile: Dockerfile
    ports:
      - "5000:8080"
      - "5001:8081"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=https://+:8081;http://+:8080
      - Auth__Authority=https://your-identity-server
      - Auth__Audience=hotel-management-api
    depends_on:
      - booking-service
      - room-service
      - customer-service
    networks:
      - hotel-network

  booking-service:
    build: ./BookingService
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    networks:
      - hotel-network

  room-service:
    build: ./RoomService
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    networks:
      - hotel-network

  customer-service:
    build: ./CustomerService
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    networks:
      - hotel-network

networks:
  hotel-network:
    driver: bridge
```
