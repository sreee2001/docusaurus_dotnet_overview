## 29. Memory Management

### Short Introduction and Official Definition

Memory management in .NET Core involves understanding and optimizing how the runtime allocates, tracks, and frees memory. This includes managing the Large Object Heap (LOH), configuring Garbage Collector modes, and implementing patterns that minimize memory pressure and improve application performance.

**Official Definition**: Memory management in .NET is the automatic process by which the runtime allocates memory for objects, tracks their usage, and reclaims memory when objects are no longer needed, primarily through the Garbage Collector (GC).

### Setup/Usage

**GC Configuration in Program.cs:**

```csharp
// Configure GC settings
var builder = WebApplication.CreateBuilder(args);

// For server applications - configure server GC
builder.Services.Configure<GCSettings>(options =>
{
    // These are typically set via runtime config or environment
});
```

**Runtime Configuration (runtimeconfig.json):**

```json
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.Concurrent": true,
      "System.GC.RetainVM": true,
      "System.Runtime.Serialization.EnableUnsafeBinaryFormatterSerialization": false
    }
  }
}
```

### Use Cases

- **High-throughput applications** requiring minimal GC pauses
- **Memory-constrained environments** needing efficient memory usage
- **Long-running services** requiring stable memory patterns
- **Real-time applications** sensitive to GC pause times
- **Large data processing** applications working with big objects
- **Microservices** optimizing for quick startup and low memory footprint

### When to Use vs When Not to Use

**When to Use Memory Optimization:**

- Application experiences frequent OutOfMemoryExceptions
- GC pauses impact user experience or SLA requirements
- Memory usage grows continuously (memory leaks suspected)
- Application handles large objects or datasets
- Running in memory-constrained environments

**When Not to Optimize:**

- Memory usage is well within acceptable limits
- GC performance doesn't impact application requirements
- Development effort outweighs memory optimization benefits
- Application has short lifespan or minimal memory requirements

### Alternatives and Trade-offs

**Alternatives:**

- Increase available memory (scale up approach)
- Use external caching systems instead of in-memory caching
- Implement object pooling for frequently allocated objects
- Use streaming instead of loading entire datasets into memory
- Offload memory-intensive operations to separate services

**Trade-offs:**

- Memory usage vs CPU usage (compression, lazy loading)
- Performance vs memory efficiency
- Code complexity vs memory optimization
- Startup time vs runtime memory usage

### Sample Code and Commands

**LOH Management:**

```csharp
// Avoiding LOH allocations
public class EfficientDataProcessor
{
    // Pool large buffers to avoid LOH pressure
    private static readonly ObjectPool<byte[]> _bufferPool =
        new ObjectPool<byte[]>(() => new byte[100_000]); // > 85KB goes to LOH

    public async Task ProcessLargeDataAsync(Stream dataStream)
    {
        // Rent buffer from pool instead of allocating
        var buffer = _bufferPool.Get();
        try
        {
            int bytesRead;
            while ((bytesRead = await dataStream.ReadAsync(buffer)) > 0)
            {
                ProcessChunk(buffer.AsSpan(0, bytesRead));
            }
        }
        finally
        {
            // Return buffer to pool
            _bufferPool.Return(buffer);
        }
    }

    private void ProcessChunk(ReadOnlySpan<byte> data)
    {
        // Process data without additional allocations
        for (int i = 0; i < data.Length; i++)
        {
            // Process byte at data[i]
        }
    }
}

// Simple ObjectPool implementation
public class ObjectPool<T> where T : class
{
    private readonly ConcurrentQueue<T> _objects = new();
    private readonly Func<T> _objectFactory;

    public ObjectPool(Func<T> objectFactory)
    {
        _objectFactory = objectFactory;
    }

    public T Get()
    {
        if (_objects.TryDequeue(out T item))
            return item;

        return _objectFactory();
    }

    public void Return(T item)
    {
        _objects.Enqueue(item);
    }
}
```

**Memory-Efficient String Operations:**

```csharp
public class StringOptimizations
{
    // Use StringValues for HTTP headers to avoid allocations
    public void ProcessHeaders(IHeaderDictionary headers)
    {
        if (headers.TryGetValue("Authorization", out StringValues authValue))
        {
            // Process without string allocation
            ProcessAuthHeader(authValue);
        }
    }

    // Use Span<char> for string manipulation without allocations
    public bool IsValidEmail(ReadOnlySpan<char> email)
    {
        int atIndex = email.IndexOf('@');
        if (atIndex <= 0 || atIndex >= email.Length - 1)
            return false;

        var localPart = email[..atIndex];
        var domainPart = email[(atIndex + 1)..];

        return localPart.Length > 0 &&
               domainPart.Length > 0 &&
               domainPart.Contains('.');
    }

    // Use StringBuilder for multiple concatenations
    public string BuildLargeString(IEnumerable<string> parts)
    {
        var sb = new StringBuilder(capacity: 1024); // Pre-size if possible

        foreach (var part in parts)
        {
            sb.Append(part);
            sb.Append(Environment.NewLine);
        }

        return sb.ToString();
    }
}
```

**Weak References for Caches:**

```csharp
public class WeakReferenceCache<TKey, TValue>
    where TKey : notnull
    where TValue : class
{
    private readonly ConcurrentDictionary<TKey, WeakReference<TValue>> _cache = new();

    public bool TryGet(TKey key, out TValue value)
    {
        value = null;

        if (_cache.TryGetValue(key, out var weakRef) &&
            weakRef.TryGetTarget(out value))
        {
            return true;
        }

        // Clean up dead reference
        _cache.TryRemove(key, out _);
        return false;
    }

    public void Set(TKey key, TValue value)
    {
        _cache[key] = new WeakReference<TValue>(value);
    }

    public void Cleanup()
    {
        var keysToRemove = new List<TKey>();

        foreach (var kvp in _cache)
        {
            if (!kvp.Value.TryGetTarget(out _))
            {
                keysToRemove.Add(kvp.Key);
            }
        }

        foreach (var key in keysToRemove)
        {
            _cache.TryRemove(key, out _);
        }
    }
}
```

**IDisposable and Memory Cleanup:**

```csharp
public class ResourceManager : IDisposable, IAsyncDisposable
{
    private readonly FileStream _fileStream;
    private readonly HttpClient _httpClient;
    private bool _disposed = false;

    public ResourceManager(string filePath)
    {
        _fileStream = new FileStream(filePath, FileMode.Create);
        _httpClient = new HttpClient();
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }

    public async ValueTask DisposeAsync()
    {
        await DisposeAsyncCore();
        Dispose(disposing: false);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                _fileStream?.Dispose();
                _httpClient?.Dispose();
            }
            _disposed = true;
        }
    }

    protected virtual async ValueTask DisposeAsyncCore()
    {
        if (_fileStream != null)
        {
            await _fileStream.DisposeAsync();
        }

        _httpClient?.Dispose();
    }
}
```

**Memory Diagnostic Commands:**

```bash
# Monitor GC activity
dotnet-counters monitor --process-id <PID> --counters System.Runtime[gen-0-gc-count,gen-1-gc-count,gen-2-gc-count,time-in-gc]

# Collect memory dump
dotnet-dump collect --process-id <PID>

# Analyze memory usage patterns
dotnet-trace collect --process-id <PID> --providers Microsoft-Windows-DotNETRuntime:0x1:4

# Force garbage collection (for testing)
dotnet-gcdump collect --process-id <PID>
```

**Environment Variables for GC Tuning:**

```bash
# Server GC (for multi-core servers)
set DOTNET_gcServer=1

# Concurrent GC
set DOTNET_gcConcurrent=1

# Large pages for LOH
set DOTNET_GCLargePages=1

# Heap count (for server GC)
set DOTNET_GCHeapCount=4
```

—continued—

````

**GC Tuning Configuration File:**
```json
// runtimeconfig.template.json
{
  "configProperties": {
    "System.GC.Server": true,
    "System.GC.Concurrent": true,
    "System.GC.RetainVM": true,
    "System.GC.LOHThreshold": 85000,
    "System.GC.HeapCount": 4,
    "System.GC.NoAffinitize": false,
    "System.GC.HeapHardLimit": 2147483648
  }
}
````

**Advanced Memory Profiling:**

```csharp
// Custom memory pressure monitoring
public class MemoryMonitoringService : BackgroundService
{
    private readonly ILogger<MemoryMonitoringService> _logger;

    public MemoryMonitoringService(ILogger<MemoryMonitoringService> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var gcMemoryInfo = GC.GetGCMemoryInfo();
            var totalMemory = GC.GetTotalMemory(false);
            var gen0Collections = GC.CollectionCount(0);
            var gen1Collections = GC.CollectionCount(1);
            var gen2Collections = GC.CollectionCount(2);

            _logger.LogInformation(
                "Memory Stats - Total: {TotalMemory:N0} bytes, " +
                "Heap Size: {HeapSize:N0} bytes, " +
                "GC Collections - Gen0: {Gen0}, Gen1: {Gen1}, Gen2: {Gen2}",
                totalMemory,
                gcMemoryInfo.HeapSizeBytes,
                gen0Collections,
                gen1Collections,
                gen2Collections);

            // Alert if memory pressure is high
            if (gcMemoryInfo.MemoryLoadBytes > gcMemoryInfo.HighMemoryLoadThresholdBytes * 0.8)
            {
                _logger.LogWarning("High memory pressure detected. Consider memory optimization.");

                // Force garbage collection if critical
                if (gcMemoryInfo.MemoryLoadBytes > gcMemoryInfo.HighMemoryLoadThresholdBytes * 0.95)
                {
                    GC.Collect();
                    GC.WaitForPendingFinalizers();
                    GC.Collect();
                }
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

**PowerShell GC Tuning Commands:**

```powershell
# Set environment variables for the current session
$env:DOTNET_gcServer = "1"
$env:DOTNET_gcConcurrent = "1"
$env:DOTNET_GCLargePages = "1"
$env:DOTNET_GCHeapCount = "4"

# Set permanently for user
[Environment]::SetEnvironmentVariable("DOTNET_gcServer", "1", "User")
[Environment]::SetEnvironmentVariable("DOTNET_gcConcurrent", "1", "User")
[Environment]::SetEnvironmentVariable("DOTNET_GCLargePages", "1", "User")
[Environment]::SetEnvironmentVariable("DOTNET_GCHeapCount", "4", "User")

# View current GC settings
dotnet --info
```

**Memory-Optimized appsettings.json:**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "MemoryCache": {
    "SizeLimit": 104857600,
    "CompactionPercentage": 0.1,
    "ExpirationScanFrequency": "00:05:00"
  },
  "GarbageCollection": {
    "MonitoringEnabled": true,
    "MonitoringInterval": "00:05:00",
    "HighMemoryThreshold": 0.8,
    "CriticalMemoryThreshold": 0.95
  }
}
```

This completes the **VI. Performance & Scalability** section covering caching, background services, performance optimization, and memory management. Each topic includes comprehensive setup instructions, use cases, alternatives, trade-offs, and practical code examples that can be implemented in .NET 8 applications.

The section provides developers with actionable guidance for optimizing their applications' performance through proper caching strategies, efficient background processing, systematic performance measurement and optimization, and effective memory management techniques.
