## 46. Service Discovery

## Short Introduction

Service Discovery is a mechanism that allows services in a distributed system to automatically locate and communicate with each other without hard-coded network locations. It provides dynamic registration and discovery of service instances, enabling resilient communication in microservices architectures where services can scale up/down and move between hosts.

## Official Definition

Service Discovery is a key component of most distributed systems and service-oriented architectures. The service instances have dynamically assigned network locations. Moreover, the set of service instances changes dynamically because of autoscaling, failures, and upgrades.

## Setup/Usage with .NET 8+ Code

**Consul Service Discovery Implementation:**

```bash
# Install Consul packages
dotnet add package Consul
dotnet add package Microsoft.Extensions.Hosting
```

**Service Registration:**

```csharp
// Services/ConsulServiceRegistry.cs
using Consul;

public interface IServiceRegistry
{
    Task RegisterServiceAsync(ServiceRegistration registration);
    Task DeregisterServiceAsync(string serviceId);
    Task<IEnumerable<ServiceInstance>> DiscoverServicesAsync(string serviceName);
}

public class ConsulServiceRegistry : IServiceRegistry, IDisposable
{
    private readonly IConsulClient _consulClient;
    private readonly ILogger<ConsulServiceRegistry> _logger;

    public ConsulServiceRegistry(IConsulClient consulClient, ILogger<ConsulServiceRegistry> logger)
    {
        _consulClient = consulClient;
        _logger = logger;
    }

    public async Task RegisterServiceAsync(ServiceRegistration registration)
    {
        var consulRegistration = new AgentServiceRegistration
        {
            ID = registration.ServiceId,
            Name = registration.ServiceName,
            Address = registration.Address,
            Port = registration.Port,
            Tags = registration.Tags.ToArray(),
            Check = new AgentServiceCheck
            {
                HTTP = $"http://{registration.Address}:{registration.Port}/health",
                Interval = TimeSpan.FromSeconds(30),
                Timeout = TimeSpan.FromSeconds(5),
                DeregisterCriticalServiceAfter = TimeSpan.FromMinutes(1)
            }
        };

        await _consulClient.Agent.ServiceRegister(consulRegistration);
        _logger.LogInformation("Service {ServiceName} registered with ID {ServiceId}",
            registration.ServiceName, registration.ServiceId);
    }

    public async Task DeregisterServiceAsync(string serviceId)
    {
        await _consulClient.Agent.ServiceDeregister(serviceId);
        _logger.LogInformation("Service {ServiceId} deregistered", serviceId);
    }

    public async Task<IEnumerable<ServiceInstance>> DiscoverServicesAsync(string serviceName)
    {
        var services = await _consulClient.Health.Service(serviceName, "", true);

        return services.Response.Select(service => new ServiceInstance
        {
            ServiceId = service.Service.ID,
            ServiceName = service.Service.Service,
            Address = service.Service.Address,
            Port = service.Service.Port,
            Tags = service.Service.Tags?.ToList() ?? new List<string>(),
            IsHealthy = service.Checks.All(check => check.Status == HealthStatus.Passing)
        });
    }

    public void Dispose()
    {
        _consulClient?.Dispose();
    }
}

// Models/ServiceRegistration.cs
public class ServiceRegistration
{
    public string ServiceId { get; set; } = string.Empty;
    public string ServiceName { get; set; } = string.Empty;
    public string Address { get; set; } = string.Empty;
    public int Port { get; set; }
    public List<string> Tags { get; set; } = new();
}

public class ServiceInstance
{
    public string ServiceId { get; set; } = string.Empty;
    public string ServiceName { get; set; } = string.Empty;
    public string Address { get; set; } = string.Empty;
    public int Port { get; set; }
    public List<string> Tags { get; set; } = new();
    public bool IsHealthy { get; set; }
}
```

**Program.cs Integration:**

```csharp
// Program.cs
using Consul;

var builder = WebApplication.CreateBuilder(args);

// Add Consul client
builder.Services.AddSingleton<IConsulClient>(provider =>
{
    var consulConfig = new ConsulClientConfiguration
    {
        Address = new Uri(builder.Configuration["Consul:Address"] ?? "http://localhost:8500")
    };
    return new ConsulClient(consulConfig);
});

// Add service registry
builder.Services.AddSingleton<IServiceRegistry, ConsulServiceRegistry>();
builder.Services.AddHostedService<ServiceRegistrationService>();

// Add HTTP client factory with service discovery
builder.Services.AddHttpClient<IBookingServiceClient, BookingServiceClient>();
builder.Services.AddSingleton<IServiceDiscoveryHttpClientFactory, ServiceDiscoveryHttpClientFactory>();

var app = builder.Build();

// Configure pipeline
app.MapControllers();
app.MapHealthChecks("/health");

app.Run();

// Services/ServiceRegistrationService.cs
public class ServiceRegistrationService : BackgroundService
{
    private readonly IServiceRegistry _serviceRegistry;
    private readonly IConfiguration _configuration;
    private readonly ILogger<ServiceRegistrationService> _logger;
    private string? _serviceId;

    public ServiceRegistrationService(
        IServiceRegistry serviceRegistry,
        IConfiguration configuration,
        ILogger<ServiceRegistrationService> logger)
    {
        _serviceRegistry = serviceRegistry;
        _configuration = configuration;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await RegisterService();

        // Keep the service alive
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        if (!string.IsNullOrEmpty(_serviceId))
        {
            await _serviceRegistry.DeregisterServiceAsync(_serviceId);
        }
        await base.StopAsync(cancellationToken);
    }

    private async Task RegisterService()
    {
        try
        {
            var serviceName = _configuration["Service:Name"] ?? "unknown-service";
            var serviceAddress = _configuration["Service:Address"] ?? "localhost";
            var servicePort = int.Parse(_configuration["Service:Port"] ?? "8080");

            _serviceId = $"{serviceName}-{Environment.MachineName}-{Guid.NewGuid():N}";

            var registration = new ServiceRegistration
            {
                ServiceId = _serviceId,
                ServiceName = serviceName,
                Address = serviceAddress,
                Port = servicePort,
                Tags = new List<string> { "v1", Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Unknown" }
            };

            await _serviceRegistry.RegisterServiceAsync(registration);
            _logger.LogInformation("Service registered successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to register service");
        }
    }
}
```

**Service Discovery HTTP Client:**

```csharp
// Services/ServiceDiscoveryHttpClientFactory.cs
public interface IServiceDiscoveryHttpClientFactory
{
    Task<HttpClient> CreateClientAsync(string serviceName);
}

public class ServiceDiscoveryHttpClientFactory : IServiceDiscoveryHttpClientFactory
{
    private readonly IServiceRegistry _serviceRegistry;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<ServiceDiscoveryHttpClientFactory> _logger;
    private readonly Dictionary<string, DateTime> _lastDiscovery = new();
    private readonly Dictionary<string, List<ServiceInstance>> _serviceCache = new();
    private readonly object _lock = new();

    public ServiceDiscoveryHttpClientFactory(
        IServiceRegistry serviceRegistry,
        IHttpClientFactory httpClientFactory,
        ILogger<ServiceDiscoveryHttpClientFactory> logger)
    {
        _serviceRegistry = serviceRegistry;
        _httpClientFactory = httpClientFactory;
        _logger = logger;
    }

    public async Task<HttpClient> CreateClientAsync(string serviceName)
    {
        var serviceInstance = await GetServiceInstanceAsync(serviceName);

        var client = _httpClientFactory.CreateClient();
        client.BaseAddress = new Uri($"http://{serviceInstance.Address}:{serviceInstance.Port}");
        client.DefaultRequestHeaders.Add("X-Service-Discovery", "Consul");

        return client;
    }

    private async Task<ServiceInstance> GetServiceInstanceAsync(string serviceName)
    {
        List<ServiceInstance> instances;

        lock (_lock)
        {
            if (_serviceCache.TryGetValue(serviceName, out instances) &&
                _lastDiscovery.TryGetValue(serviceName, out var lastDiscovery) &&
                DateTime.UtcNow - lastDiscovery < TimeSpan.FromMinutes(1))
            {
                // Use cached instances if they're recent
            }
            else
            {
                instances = null;
            }
        }

        if (instances == null)
        {
            instances = (await _serviceRegistry.DiscoverServicesAsync(serviceName))
                .Where(s => s.IsHealthy)
                .ToList();

            lock (_lock)
            {
                _serviceCache[serviceName] = instances;
                _lastDiscovery[serviceName] = DateTime.UtcNow;
            }
        }

        if (!instances.Any())
        {
            throw new InvalidOperationException($"No healthy instances found for service: {serviceName}");
        }

        // Simple round-robin load balancing
        var selectedInstance = instances[Random.Shared.Next(instances.Count)];

        _logger.LogDebug("Selected instance {ServiceId} for service {ServiceName}",
            selectedInstance.ServiceId, serviceName);

        return selectedInstance;
    }
}
```

### Use Cases

- **Microservices Communication**: Dynamic service-to-service communication
- **Load Balancing**: Distribute requests across multiple service instances
- **Auto-scaling**: Automatically discover new instances as they come online
- **Fault Tolerance**: Remove unhealthy instances from service discovery
- **Blue-Green Deployments**: Route traffic between different service versions
- **Multi-Environment Support**: Different service configurations per environment

## When to Use vs When Not to Use

**Use Service Discovery when:**

- Building microservices with multiple instances
- Services need to communicate dynamically
- Using container orchestration (Docker, Kubernetes)
- Implementing auto-scaling
- Need health-based routing
- Working with multiple environments

**Consider alternatives when:**

- Simple monolithic applications
- Fixed, well-known service endpoints
- Small number of services with static configuration
- Network policies restrict dynamic discovery
- Complexity outweighs benefits

### Market Alternatives & Pros/Cons

### Alternatives:

- **Kubernetes DNS**: Built-in service discovery for Kubernetes
- **Netflix Eureka**: Service registry for resilient mid-tier load balancing
- **etcd**: Distributed key-value store for service discovery
- **Apache Zookeeper**: Centralized service for configuration and naming
- **AWS Cloud Map**: Managed service discovery for AWS
- **Azure Service Fabric**: Microsoft's service discovery solution

### Pros:

- Dynamic service location
- Automatic health checking
- Load balancing capabilities
- Multi-datacenter support
- Rich querying and filtering
- Battle-tested in production

### Cons:

- Additional infrastructure dependency
- Potential single point of failure
- Network latency for discovery calls
- Complexity in service registration/deregistration
- Consistency challenges in distributed scenarios

### Complete Runnable Sample

**Complete Service Discovery Setup:**

```yaml
# docker-compose.yml
version: "3.8"
services:
  consul:
    image: consul:1.15
    ports:
      - "8500:8500"
    command: >
      consul agent -dev -client=0.0.0.0 -ui
    environment:
      - CONSUL_BIND_INTERFACE=eth0
    networks:
      - hotel-network

  booking-service:
    build: ./BookingService
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - Consul__Address=http://consul:8500
      - Service__Name=booking-service
      - Service__Address=booking-service
      - Service__Port=8080
    depends_on:
      - consul
    networks:
      - hotel-network
    deploy:
      replicas: 2

  room-service:
    build: ./RoomService
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - Consul__Address=http://consul:8500
      - Service__Name=room-service
      - Service__Address=room-service
      - Service__Port=8080
    depends_on:
      - consul
    networks:
      - hotel-network
    deploy:
      replicas: 3

networks:
  hotel-network:
    driver: bridge
```
