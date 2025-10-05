## 28. Performance Optimization

### Short Introduction and Official Definition

Performance optimization in .NET Core involves analyzing, measuring, and improving application speed, throughput, and resource utilization. It encompasses profiling, JIT/AOT compilation strategies, threading optimizations, and runtime tuning.

**Official Definition**: Performance optimization is the systematic process of making .NET applications run faster and use resources more efficiently through code improvements, configuration tuning, and runtime optimizations.

### Setup/Usage

**Profiling Tools Setup:**

```xml
<!-- Add to .csproj for profiling -->
<PackageReference Include="Microsoft.Diagnostics.Tracing.TraceEvent" Version="3.0.2" />
<PackageReference Include="BenchmarkDotNet" Version="0.13.8" />
```

**AOT Publishing:**

```xml
<!-- Enable AOT in .csproj -->
<PropertyGroup>
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>
</PropertyGroup>
```

### Use Cases

- **High-traffic web applications** requiring low latency
- **Microservices** with strict SLA requirements
- **Real-time systems** needing predictable response times
- **Resource-constrained environments** (IoT, edge computing)
- **Cost optimization** in cloud deployments
- **Batch processing** applications handling large datasets

### When to Use vs When Not to Use

**When to Use:**

- Performance bottlenecks are identified through profiling
- Application doesn't meet performance requirements
- Resource costs are high due to inefficiency
- User experience is impacted by slow response times
- Scaling requirements demand optimization

**When Not to Use:**

- Premature optimization without measurement
- Development time outweighs performance benefits
- Simple applications with minimal load
- Temporary or prototype applications
- When optimization adds significant complexity

### Alternatives and Trade-offs

**Alternatives:**

- Horizontal scaling instead of optimization
- Caching layers to mask performance issues
- CDN for content delivery optimization
- Database optimization and indexing
- Infrastructure upgrades (faster CPUs, more RAM)

**Trade-offs:**

- Development time vs performance gains
- Code complexity vs execution speed
- Memory usage vs CPU usage
- Startup time vs runtime performance (AOT vs JIT)

### Sample Code and Commands

**BenchmarkDotNet Example:**

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[SimpleJob(BenchmarkDotNet.Jobs.RuntimeMoniker.Net80)]
public class StringConcatenationBenchmark
{
    private readonly string[] _strings = Enumerable.Range(0, 1000)
        .Select(i => $"String_{i}")
        .ToArray();

    [Benchmark]
    public string UsingStringConcatenation()
    {
        string result = "";
        foreach (var str in _strings)
        {
            result += str;
        }
        return result;
    }

    [Benchmark]
    public string UsingStringBuilder()
    {
        var sb = new StringBuilder();
        foreach (var str in _strings)
        {
            sb.Append(str);
        }
        return sb.ToString();
    }

    [Benchmark]
    public string UsingStringJoin()
    {
        return string.Join("", _strings);
    }
}

// Program.cs
class Program
{
    static void Main(string[] args)
    {
        BenchmarkRunner.Run<StringConcatenationBenchmark>();
    }
}
```

**ThreadPool Tuning:**

```csharp
// Program.cs - Configure ThreadPool
var builder = WebApplication.CreateBuilder(args);

// Configure ThreadPool for high-throughput scenarios
ThreadPool.SetMinThreads(
    workerThreads: Environment.ProcessorCount * 4,
    completionPortThreads: Environment.ProcessorCount * 4
);

// Configure Kestrel for performance
builder.Services.Configure<KestrelServerOptions>(options =>
{
    options.Limits.MaxConcurrentConnections = 1000;
    options.Limits.MaxConcurrentUpgradedConnections = 1000;
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10MB
    options.Limits.MinRequestBodyDataRate = new MinDataRate(100, TimeSpan.FromSeconds(10));
});
```

**High-Performance HTTP Client:**

```csharp
// Efficient HttpClient usage
public class OptimizedHttpService
{
    private static readonly HttpClient _httpClient = new HttpClient(new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(15),
        MaxConnectionsPerServer = 10
    });

    static OptimizedHttpService()
    {
        _httpClient.DefaultRequestHeaders.Connection.Add("keep-alive");
    }

    public async Task<T> GetAsync<T>(string url)
    {
        using var response = await _httpClient.GetAsync(url);

        // Use System.Text.Json for better performance
        var jsonStream = await response.Content.ReadAsStreamAsync();
        return await JsonSerializer.DeserializeAsync<T>(jsonStream);
    }
}
```

**Memory-Efficient Collections:**

```csharp
// Use Span<T> and Memory<T> for high-performance scenarios
public class HighPerformanceProcessor
{
    public void ProcessData(ReadOnlySpan<byte> data)
    {
        // Process data without allocations
        for (int i = 0; i < data.Length; i++)
        {
            var value = data[i];
            // Process value
        }
    }

    public async Task<int> ProcessFileAsync(string filePath)
    {
        // Use Memory<T> for async operations
        using var fileStream = new FileStream(filePath, FileMode.Open, FileAccess.Read);
        var buffer = new byte[4096];
        var memory = buffer.AsMemory();

        int totalBytes = 0;
        int bytesRead;

        while ((bytesRead = await fileStream.ReadAsync(memory)) > 0)
        {
            totalBytes += bytesRead;
            ProcessData(memory.Span[..bytesRead]);
        }

        return totalBytes;
    }
}
```

**Performance Commands:**

```bash
# Build with AOT
dotnet publish -c Release -r win-x64 --self-contained

# Profile with dotnet-trace
dotnet tool install --global dotnet-trace
dotnet trace collect --process-id <PID> --providers Microsoft-DotNETCore-SampleProfiler

# Analyze memory usage
dotnet tool install --global dotnet-dump
dotnet dump collect --process-id <PID>

# Monitor performance counters
dotnet tool install --global dotnet-counters
dotnet counters monitor --process-id <PID> System.Runtime
```
