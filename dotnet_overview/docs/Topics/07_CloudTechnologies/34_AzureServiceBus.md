## 35. Azure Service Bus

### Short Introduction + Official Definition

Azure Service Bus is a fully managed enterprise message broker with message queues and publish-subscribe topics. It provides reliable cloud messaging between applications and services, supporting advanced messaging patterns like sessions, transactions, and dead lettering.

**Official Definition**: "Azure Service Bus is a fully managed enterprise integration message broker. Service Bus can decouple applications and services. Service Bus offers a reliable and secure platform for asynchronous transfer of data and state."

### Setup and Deployment Steps

**Azure CLI Setup**:

```bash
# Create Service Bus namespace
az servicebus namespace create --resource-group myResourceGroup --name myservicebus --location eastus --sku Standard

# Create queue
az servicebus queue create --resource-group myResourceGroup --namespace-name myservicebus --name myqueue --max-size 1024

# Create topic
az servicebus topic create --resource-group myResourceGroup --namespace-name myservicebus --name mytopic

# Create subscription
az servicebus topic subscription create --resource-group myResourceGroup --namespace-name myservicebus --topic-name mytopic --name mysubscription
```

**Bicep Template**:

```bicep
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'myservicebus'
  location: resourceGroup().location
  sku: {
    name: 'Standard'
    tier: 'Standard'
  }
}

resource serviceBusQueue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: serviceBusNamespace
  name: 'orders'
  properties: {
    maxSizeInMegabytes: 1024
    defaultMessageTimeToLive: 'P14D'
    deadLetteringOnMessageExpiration: true
  }
}
```

### Typical Usage and Integration with .NET Apps

**NuGet Package**:

```xml
<PackageReference Include="Azure.Messaging.ServiceBus" Version="7.17.0" />
```

**Service Registration and Configuration**:

```csharp
// appsettings.json
{
  "ServiceBus": {
    "ConnectionString": "Endpoint=sb://myservicebus.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=your-key"
  }
}

// Program.cs
using Azure.Messaging.ServiceBus;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<ServiceBusClient>(provider =>
{
    var connectionString = builder.Configuration.GetConnectionString("ServiceBus");
    return new ServiceBusClient(connectionString);
});

builder.Services.AddScoped<IMessageService, MessageService>();
```

**Message Service Implementation**:

```csharp
public interface IMessageService
{
    Task SendMessageAsync<T>(string queueName, T message);
    Task<T> ReceiveMessageAsync<T>(string queueName);
    Task PublishMessageAsync<T>(string topicName, T message);
    Task StartProcessingAsync<T>(string queueName, Func<T, Task> messageHandler);
}

public class MessageService : IMessageService
{
    private readonly ServiceBusClient _client;
    private readonly ILogger<MessageService> _logger;

    public MessageService(ServiceBusClient client, ILogger<MessageService> logger)
    {
        _client = client;
        _logger = logger;
    }

    public async Task SendMessageAsync<T>(string queueName, T message)
    {
        var sender = _client.CreateSender(queueName);
        var json = JsonSerializer.Serialize(message);
        var serviceBusMessage = new ServiceBusMessage(json)
        {
            ContentType = "application/json",
            MessageId = Guid.NewGuid().ToString()
        };

        await sender.SendMessageAsync(serviceBusMessage);
        _logger.LogInformation($"Message sent to queue {queueName}: {serviceBusMessage.MessageId}");
    }

    public async Task PublishMessageAsync<T>(string topicName, T message)
    {
        var sender = _client.CreateSender(topicName);
        var json = JsonSerializer.Serialize(message);
        var serviceBusMessage = new ServiceBusMessage(json)
        {
            ContentType = "application/json",
            Subject = typeof(T).Name
        };

        await sender.SendMessageAsync(serviceBusMessage);
        _logger.LogInformation($"Message published to topic {topicName}");
    }

    public async Task StartProcessingAsync<T>(string queueName, Func<T, Task> messageHandler)
    {
        var processor = _client.CreateProcessor(queueName);

        processor.ProcessMessageAsync += async args =>
        {
            try
            {
                var json = args.Message.Body.ToString();
                var message = JsonSerializer.Deserialize<T>(json);
                await messageHandler(message);
                await args.CompleteMessageAsync(args.Message);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing message");
                await args.DeadLetterMessageAsync(args.Message);
            }
        };

        processor.ProcessErrorAsync += args =>
        {
            _logger.LogError(args.Exception, "Error occurred while processing messages");
            return Task.CompletedTask;
        };

        await processor.StartProcessingAsync();
    }
}

// Message Models
public class OrderCreatedEvent
{
    public int OrderId { get; set; }
    public string CustomerEmail { get; set; }
    public decimal Amount { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

**Background Service for Message Processing**:

```csharp
public class OrderProcessingService : BackgroundService
{
    private readonly IMessageService _messageService;
    private readonly ILogger<OrderProcessingService> _logger;

    public OrderProcessingService(IMessageService messageService, ILogger<OrderProcessingService> logger)
    {
        _messageService = messageService;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await _messageService.StartProcessingAsync<OrderCreatedEvent>("orders", async order =>
        {
            _logger.LogInformation($"Processing order {order.OrderId} for {order.CustomerEmail}");

            // Process the order
            await Task.Delay(1000); // Simulate processing

            _logger.LogInformation($"Order {order.OrderId} processed successfully");
        });
    }
}
```

### Use Cases

- Decoupling microservices communication
- Reliable message delivery with retry mechanisms
- Event-driven architectures
- Load leveling between applications
- Integration with legacy systems
- Workflow orchestration and saga patterns

### When to Use vs Alternatives

**Use Azure Service Bus when**:

- Advanced messaging features needed (sessions, transactions, duplicate detection)
- Enterprise-grade reliability and security required
- Integration with Azure ecosystem is important
- Complex routing and filtering capabilities needed
- FIFO (First-In-First-Out) ordering is critical

**Don't use when**:

- Simple queue requirements (Azure Storage Queues sufficient)
- Cost optimization is primary concern
- Ultra-high throughput needed (consider Event Hubs)
- Real-time communication required (use SignalR)

**Alternatives**:

- **Azure**: Storage Queues (simpler), Event Hubs (high throughput), Event Grid (event routing)
- **AWS**: SQS, SNS, Amazon MQ
- **GCP**: Cloud Tasks, Pub/Sub
- **Open Source**: RabbitMQ, Apache Kafka

### Market Pros/Cons and Cost Considerations

**Pros**:

- Enterprise messaging features (sessions, transactions, duplicate detection)
- Dead letter queues for failed messages
- Auto-scaling and high availability
- Advanced security with Azure AD integration
- Message scheduling and deferral

**Cons**:

- More expensive than simple queue solutions
- Complexity overhead for simple scenarios
- Learning curve for advanced features
- Potential vendor lock-in

**Cost Considerations**:

- Basic tier: ~$0.05 per million operations
- Standard tier: ~$10/month base + $0.80 per million operations
- Premium tier: ~$677/month for dedicated capacity
- Additional charges for message storage and throughput units
