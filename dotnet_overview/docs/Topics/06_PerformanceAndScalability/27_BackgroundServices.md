## 27. Background Services

### Short Introduction and Official Definition

Background services in .NET Core provide a way to run long-running tasks, scheduled jobs, and background processing outside the main request-response cycle. They enable applications to perform work continuously or on schedule without blocking user requests.

**Official Definition**: Background services are hosted services that implement `IHostedService` or inherit from `BackgroundService` class, allowing applications to run background tasks that start when the application starts and stop gracefully when the application shuts down.

### Setup/Usage

**Basic BackgroundService:**

```csharp
// Program.cs
builder.Services.AddHostedService<DataProcessingService>();
```

**With Hangfire:**

```csharp
// Program.cs
builder.Services.AddHangfire(config =>
{
    config.UseSqlServerStorage(builder.Configuration.GetConnectionString("DefaultConnection"));
});

builder.Services.AddHangfireServer();

var app = builder.Build();
app.UseHangfireDashboard("/hangfire");
```

### Use Cases

- **Data Processing**: ETL operations, file processing, data transformation
- **Scheduled Tasks**: Daily reports, cleanup operations, backup processes
- **Message Queue Processing**: Email sending, notification delivery, event handling
- **Monitoring and Health Checks**: System monitoring, log cleanup, performance metrics
- **Cache Warming**: Preloading frequently accessed data
- **External API Synchronization**: Periodic data sync with third-party services

### When to Use vs When Not to Use

**When to Use:**

- Tasks don't require immediate user feedback
- Long-running operations that shouldn't block requests
- Scheduled or recurring operations
- Resource-intensive operations
- Tasks that can tolerate some delay

**When Not to Use:**

- Real-time operations requiring immediate response
- Simple, fast operations better handled in request pipeline
- Operations requiring user interaction
- Tasks requiring guaranteed execution order (consider message queues instead)
- Memory-intensive operations on resource-constrained systems

### Alternatives and Trade-offs

**Alternatives:**

- Azure Functions or AWS Lambda for serverless processing
- Message queues (Azure Service Bus, RabbitMQ) for reliable processing
- Windows Services or Linux daemons for system-level services
- Cron jobs for simple scheduled tasks

**Trade-offs:**

- Application resources vs dedicated processing services
- Immediate vs delayed processing
- Simple implementation vs advanced scheduling features
- In-process vs out-of-process execution

### Sample Code and Commands

**Basic BackgroundService:**

```csharp
public class DataProcessingService : BackgroundService
{
    private readonly ILogger<DataProcessingService> _logger;
    private readonly IServiceScopeFactory _scopeFactory;

    public DataProcessingService(
        ILogger<DataProcessingService> logger,
        IServiceScopeFactory scopeFactory)
    {
        _logger = logger;
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var dataService = scope.ServiceProvider.GetRequiredService<IDataService>();

                await ProcessDataAsync(dataService);

                _logger.LogInformation("Data processing completed at {Time}", DateTimeOffset.Now);

                // Wait 30 minutes before next execution
                await Task.Delay(TimeSpan.FromMinutes(30), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error occurred in data processing");
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken); // Wait before retry
            }
        }
    }

    private async Task ProcessDataAsync(IDataService dataService)
    {
        var unprocessedData = await dataService.GetUnprocessedDataAsync();

        foreach (var item in unprocessedData)
        {
            await dataService.ProcessItemAsync(item);
        }
    }
}
```

**IHostedService Implementation:**

```csharp
public class TimedHostedService : IHostedService, IDisposable
{
    private readonly ILogger<TimedHostedService> _logger;
    private readonly IServiceScopeFactory _scopeFactory;
    private Timer _timer;

    public TimedHostedService(ILogger<TimedHostedService> logger, IServiceScopeFactory scopeFactory)
    {
        _logger = logger;
        _scopeFactory = scopeFactory;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Timed Hosted Service running");

        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromHours(1));

        return Task.CompletedTask;
    }

    private async void DoWork(object state)
    {
        using var scope = _scopeFactory.CreateScope();
        var reportService = scope.ServiceProvider.GetRequiredService<IReportService>();

        try
        {
            await reportService.GenerateHourlyReportAsync();
            _logger.LogInformation("Hourly report generated successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error generating hourly report");
        }
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Timed Hosted Service is stopping");
        _timer?.Change(Timeout.Infinite, 0);
        return Task.CompletedTask;
    }

    public void Dispose()
    {
        _timer?.Dispose();
    }
}
```

**Hangfire Example:**

```csharp
// Job registration
public class JobService
{
    public void ConfigureJobs()
    {
        // Fire-and-forget job
        BackgroundJob.Enqueue(() => ProcessOrderAsync(123));

        // Delayed job
        BackgroundJob.Schedule(() => SendReminderEmail(456), TimeSpan.FromHours(24));

        // Recurring job
        RecurringJob.AddOrUpdate("daily-cleanup", () => CleanupTempFiles(), Cron.Daily);
    }

    public async Task ProcessOrderAsync(int orderId)
    {
        // Process order logic
        await Task.Delay(1000); // Simulate work
    }

    public async Task SendReminderEmail(int userId)
    {
        // Send email logic
        await Task.Delay(500);
    }

    public async Task CleanupTempFiles()
    {
        // Cleanup logic
        await Task.Delay(2000);
    }
}
```
