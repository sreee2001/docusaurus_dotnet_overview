## 47. Circuit Breaker Pattern (Polly)

## Short Introduction

The Circuit Breaker pattern prevents an application from repeatedly trying to execute an operation that's likely to fail, allowing it to continue without waiting for the fault to be fixed or wasting CPU cycles. Polly is a .NET resilience and transient-fault-handling library that implements circuit breaker along with retry, timeout, bulkhead isolation, and fallback patterns.

## Official Definition

The Circuit Breaker pattern is a design pattern used in software development to detect failures and encapsulates the logic of preventing a failure from constantly recurring during maintenance, temporary external system failure, or unexpected system difficulties.

## Setup/Usage with .NET 8+ Code

**Install Polly:**

```bash
dotnet add package Polly
dotnet add package Polly.Extensions.Http
dotnet add package Microsoft.Extensions.Http.Polly
```

**Basic Circuit Breaker Setup:**

```csharp
// Program.cs
using Polly;
using Polly.Extensions.Http;

var builder = WebApplication.CreateBuilder(args);

// Add Polly policies
builder.Services.AddHttpClient<IRoomService, RoomServiceClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Services:RoomService:BaseUrl"]);
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy())
.AddPolicyHandler(GetTimeoutPolicy());

// Add resilience service
builder.Services.AddSingleton<IResilienceService, ResilienceService>();

var app = builder.Build();

// Polly policies
static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
            onRetry: (outcome, timespan, retryCount, context) =>
            {
                var logger = context.GetLogger();
                logger?.LogWarning("Retry {RetryCount} after {Delay}ms", retryCount, timespan.TotalMilliseconds);
            });
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 3,
            durationOfBreak: TimeSpan.FromSeconds(30),
            onBreak: (result, timespan) =>
            {
                Console.WriteLine($"Circuit breaker opened for {timespan}");
            },
            onReset: () =>
            {
                Console.WriteLine("Circuit breaker closed");
            },
            onHalfOpen: () =>
            {
                Console.WriteLine("Circuit breaker half-open");
            });
}

static IAsyncPolicy<HttpResponseMessage> GetTimeoutPolicy()
{
    return Policy.TimeoutAsync<HttpResponseMessage>(10);
}
```

**Advanced Resilience Service:**

```csharp
// Services/ResilienceService.cs
using Polly;
using Polly.CircuitBreaker;
using Polly.Fallback;
using Polly.Retry;
using Polly.Timeout;

public interface IResilienceService
{
    Task<T> ExecuteAsync<T>(Func<Task<T>> operation, string operationName);
    Task<T> ExecuteWithFallbackAsync<T>(Func<Task<T>> operation, Func<Task<T>> fallback, string operationName);
    CircuitBreakerState GetCircuitBreakerState(string operationName);
}

public class ResilienceService : IResilienceService
{
    private readonly ILogger<ResilienceService> _logger;
    private readonly Dictionary<string, IAsyncPolicy> _policies = new();
    private readonly Dictionary<string, CircuitBreakerPolicy> _circuitBreakers = new();

    public ResilienceService(ILogger<ResilienceService> logger)
    {
        _logger = logger;
        InitializePolicies();
    }

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation, string operationName)
    {
        var policy = GetOrCreatePolicy<T>(operationName);

        try
        {
            return await policy.ExecuteAsync(async () =>
            {
                _logger.LogDebug("Executing operation: {OperationName}", operationName);
                return await operation();
            });
        }
        catch (CircuitBreakerOpenException)
        {
            _logger.LogWarning("Circuit breaker is open for operation: {OperationName}", operationName);
            throw new ServiceUnavailableException($"Service temporarily unavailable: {operationName}");
        }
        catch (TimeoutRejectedException)
        {
            _logger.LogWarning("Timeout occurred for operation: {OperationName}", operationName);
            throw new TimeoutException($"Operation timed out: {operationName}");
        }
    }

    public async Task<T> ExecuteWithFallbackAsync<T>(Func<Task<T>> operation, Func<Task<T>> fallback, string operationName)
    {
        var policy = GetOrCreatePolicyWithFallback<T>(operationName, fallback);

        return await policy.ExecuteAsync(async () =>
        {
            _logger.LogDebug("Executing operation with fallback: {OperationName}", operationName);
            return await operation();
        });
    }

    public CircuitBreakerState GetCircuitBreakerState(string operationName)
    {
        if (_circuitBreakers.TryGetValue(operationName, out var circuitBreaker))
        {
            return circuitBreaker.CircuitState;
        }
        return CircuitBreakerState.Closed;
    }

    private void InitializePolicies()
    {
        // Initialize common policies for different operation types
        CreatePolicyForOperation("database",
            retryCount: 3,
            circuitBreakerThreshold: 5,
            circuitBreakerDuration: TimeSpan.FromMinutes(1),
            timeoutDuration: TimeSpan.FromSeconds(30));

        CreatePolicyForOperation("external-api",
            retryCount: 2,
            circuitBreakerThreshold: 3,
            circuitBreakerDuration: TimeSpan.FromSeconds(30),
            timeoutDuration: TimeSpan.FromSeconds(10));

        CreatePolicyForOperation("internal-service",
            retryCount: 2,
            circuitBreakerThreshold: 4,
            circuitBreakerDuration: TimeSpan.FromSeconds(45),
            timeoutDuration: TimeSpan.FromSeconds(15));
    }

    private void CreatePolicyForOperation(string operationName, int retryCount, int circuitBreakerThreshold,
        TimeSpan circuitBreakerDuration, TimeSpan timeoutDuration)
    {
        // Retry policy
        var retryPolicy = Policy
            .Handle<HttpRequestException>()
            .Or<TaskCanceledException>()
            .Or<TimeoutException>()
            .WaitAndRetryAsync(
                retryCount: retryCount,
                sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (outcome, timespan, retryCount, context) =>
                {
                    _logger.LogWarning("Retry {RetryCount} for {OperationName} after {Delay}ms",
                        retryCount, operationName, timespan.TotalMilliseconds);
                });

        // Circuit breaker policy
        var circuitBreakerPolicy = Policy
            .Handle<HttpRequestException>()
            .Or<TaskCanceledException>()
            .Or<TimeoutException>()
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: circuitBreakerThreshold,
                durationOfBreak: circuitBreakerDuration,
                onBreak: (exception, timespan) =>
                {
                    _logger.LogWarning("Circuit breaker opened for {OperationName} for {Duration}ms",
                        operationName, timespan.TotalMilliseconds);
                },
                onReset: () =>
                {
                    _logger.LogInformation("Circuit breaker reset for {OperationName}", operationName);
                },
                onHalfOpen: () =>
                {
                    _logger.LogInformation("Circuit breaker half-open for {OperationName}", operationName);
                });

        // Timeout policy
        var timeoutPolicy = Policy.TimeoutAsync(timeoutDuration);

        // Combine policies
        var combinedPolicy = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy, timeoutPolicy);

        _policies[operationName] = combinedPolicy;
        _circuitBreakers[operationName] = circuitBreakerPolicy;
    }

    private IAsyncPolicy<T> GetOrCreatePolicy<T>(string operationName)
    {
        if (_policies.TryGetValue(operationName, out var policy))
        {
            return (IAsyncPolicy<T>)policy;
        }

        // Create default policy if not exists
        CreatePolicyForOperation(operationName, 2, 3, TimeSpan.FromSeconds(30), TimeSpan.FromSeconds(10));
        return (IAsyncPolicy<T>)_policies[operationName];
    }

    private IAsyncPolicy<T> GetOrCreatePolicyWithFallback<T>(string operationName, Func<Task<T>> fallback)
    {
        var basePolicy = GetOrCreatePolicy<T>(operationName);

        var fallbackPolicy = Policy<T>
            .Handle<Exception>()
            .FallbackAsync(
                fallbackAction: async (cancellationToken) => await fallback(),
                onFallbackAsync: async (result) =>
                {
                    _logger.LogWarning("Fallback executed for operation: {OperationName}", operationName);
                    await Task.CompletedTask;
                });

        return Policy.WrapAsync(fallbackPolicy, basePolicy);
    }
}

// Exceptions/ServiceUnavailableException.cs
public class ServiceUnavailableException : Exception
{
    public ServiceUnavailableException(string message) : base(message) { }
    public ServiceUnavailableException(string message, Exception innerException) : base(message, innerException) { }
}
```

**Usage in Services:**

```csharp
// Services/BookingService.cs with Circuit Breaker
public class BookingService : IBookingService
{
    private readonly IResilienceService _resilienceService;
    private readonly HttpClient _roomServiceClient;
    private readonly ILogger<BookingService> _logger;

    public BookingService(
        IResilienceService resilienceService,
        HttpClient roomServiceClient,
        ILogger<BookingService> logger)
    {
        _resilienceService = resilienceService;
        _roomServiceClient = roomServiceClient;
        _logger = logger;
    }

    public async Task<BookingResult> CreateBookingAsync(CreateBookingRequest request)
    {
        try
        {
            // Check room availability with circuit breaker protection
            var roomAvailability = await _resilienceService.ExecuteWithFallbackAsync(
                operation: async () => await CheckRoomAvailabilityAsync(request.RoomId, request.CheckIn, request.CheckOut),
                fallback: async () => await GetCachedRoomAvailabilityAsync(request.RoomId),
                operationName: "room-availability-check"
            );

            if (!roomAvailability.IsAvailable)
            {
                return BookingResult.Failed("Room not available");
            }

            // Create booking with database resilience
            var booking = await _resilienceService.ExecuteAsync(
                operation: async () => await SaveBookingToDatabaseAsync(request, roomAvailability.Price),
                operationName: "database"
            );

            // Send confirmation email with fallback
            await _resilienceService.ExecuteWithFallbackAsync(
                operation: async () => await SendConfirmationEmailAsync(booking),
                fallback: async () => await QueueEmailForLaterAsync(booking),
                operationName: "email-service"
            );

            return BookingResult.Success(booking);
        }
        catch (ServiceUnavailableException ex)
        {
            _logger.LogError(ex, "Service unavailable during booking creation");
            return BookingResult.Failed("Service temporarily unavailable. Please try again later.");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error during booking creation");
            return BookingResult.Failed("An unexpected error occurred");
        }
    }

    private async Task<RoomAvailability> CheckRoomAvailabilityAsync(int roomId, DateTime checkIn, DateTime checkOut)
    {
        var response = await _roomServiceClient.PostAsJsonAsync("/api/rooms/check-availability", new
        {
            RoomId = roomId,
            CheckIn = checkIn,
            CheckOut = checkOut
        });

        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<RoomAvailability>()
               ?? throw new InvalidOperationException("Invalid response from room service");
    }

    private async Task<RoomAvailability> GetCachedRoomAvailabilityAsync(int roomId)
    {
        // Fallback: return cached or default availability
        _logger.LogInformation("Using fallback room availability for room {RoomId}", roomId);
        return new RoomAvailability { IsAvailable = false, Price = 0 };
    }

    // Additional methods...
}

// Health check for circuit breaker status
public class CircuitBreakerHealthCheck : IHealthCheck
{
    private readonly IResilienceService _resilienceService;

    public CircuitBreakerHealthCheck(IResilienceService resilienceService)
    {
        _resilienceService = resilienceService;
    }

    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        var circuitBreakerStates = new Dictionary<string, object>
        {
            ["database"] = _resilienceService.GetCircuitBreakerState("database").ToString(),
            ["external-api"] = _resilienceService.GetCircuitBreakerState("external-api").ToString(),
            ["internal-service"] = _resilienceService.GetCircuitBreakerState("internal-service").ToString()
        };

        var hasOpenCircuitBreakers = circuitBreakerStates.Values
            .Any(state => state.ToString() == CircuitBreakerState.Open.ToString());

        return Task.FromResult(hasOpenCircuitBreakers
            ? HealthCheckResult.Degraded("Some circuit breakers are open", null, circuitBreakerStates)
            : HealthCheckResult.Healthy("All circuit breakers are closed", circuitBreakerStates));
    }
}
```

### Use Cases

- **External Service Calls**: Protect against failing external APIs
- **Database Operations**: Prevent database overload during outages
- **Microservice Communication**: Resilient service-to-service calls
- **Resource Protection**: Prevent cascade failures
- **Performance Degradation**: Handle slow or unresponsive services
- **Network Instability**: Cope with intermittent connectivity issues

## When to Use vs When Not to Use

**Use Circuit Breaker when:**

- Calling external services or APIs
- Operations can fail transiently
- Failures can cascade through the system
- Need automatic recovery mechanisms
- Want to prevent resource exhaustion
- Building distributed systems

**Consider alternatives when:**

- Operations must always succeed
- Failures are rare and not cascading
- Simple applications with few dependencies
- Performance overhead is unacceptable
- Users expect immediate error feedback

### Market Alternatives & Pros/Cons

### Alternatives:

- **Hystrix**: Netflix's circuit breaker library (Java)
- **Resilience4j**: Lightweight fault tolerance library (Java)
- **Istio**: Service mesh with circuit breaker capabilities
- **AWS App Mesh**: Managed service mesh with resilience features
- **Spring Cloud Circuit Breaker**: Spring framework integration
- **Custom implementations**: Hand-rolled circuit breaker logic

### Pros:

- Prevents cascade failures
- Automatic failure detection and recovery
- Configurable thresholds and timeouts
- Rich monitoring and metrics
- Excellent .NET integration
- Production-proven reliability

### Cons:

- Additional complexity
- Potential false positives
- Configuration tuning required
- Memory overhead for state tracking
- May mask underlying issues
- Learning curve for proper usage

### Complete Runnable Sample

**Production-Ready Circuit Breaker Configuration:**

```json
{
  "CircuitBreaker": {
    "Policies": {
      "database": {
        "RetryCount": 3,
        "CircuitBreakerThreshold": 5,
        "CircuitBreakerDurationSeconds": 60,
        "TimeoutSeconds": 30
      },
      "external-api": {
        "RetryCount": 2,
        "CircuitBreakerThreshold": 3,
        "CircuitBreakerDurationSeconds": 30,
        "TimeoutSeconds": 10
      },
      "email-service": {
        "RetryCount": 1,
        "CircuitBreakerThreshold": 2,
        "CircuitBreakerDurationSeconds": 120,
        "TimeoutSeconds": 5
      }
    }
  }
}
```
