### Logging

#### Short Introduction

Logging in .NET Core provides a built-in, extensible logging framework that supports multiple logging providers and structured logging.

#### Official Definition

The .NET logging system provides a logging API that works with a variety of built-in and third-party logging providers, enabling developers to log messages with different severity levels.

#### Usage

```csharp
// Program.cs - Configure logging
builder.Logging.ClearProviders();
builder.Logging.AddConsole();
builder.Logging.AddDebug();
builder.Logging.AddEventSourceLogger();

// Add Serilog (popular third-party logger)
builder.Host.UseSerilog((context, configuration) =>
    configuration.ReadFrom.Configuration(context.Configuration));
```

#### Log Levels

- **Trace**: Very detailed logs
- **Debug**: Debug information
- **Information**: General information
- **Warning**: Warning messages
- **Error**: Error messages
- **Critical**: Critical errors

#### Use Cases

- Application monitoring
- Debugging and troubleshooting
- Audit trails
- Performance monitoring
- Security monitoring

#### Sample Usage

```csharp
public class OrderService : IOrderService
{
    private readonly ILogger<OrderService> _logger;
    private readonly IOrderRepository _orderRepository;

    public OrderService(ILogger<OrderService> logger, IOrderRepository orderRepository)
    {
        _logger = logger;
        _orderRepository = orderRepository;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderDto dto)
    {
        _logger.LogInformation("Creating order for customer {CustomerId}", dto.CustomerId);

        try
        {
            var order = new Order
            {
                CustomerId = dto.CustomerId,
                OrderDate = DateTime.UtcNow,
                Items = dto.Items.Select(i => new OrderItem
                {
                    ProductId = i.ProductId,
                    Quantity = i.Quantity
                }).ToList()
            };

            var createdOrder = await _orderRepository.CreateAsync(order);

            _logger.LogInformation("Order {OrderId} created successfully for customer {CustomerId}",
                createdOrder.Id, dto.CustomerId);

            return createdOrder;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create order for customer {CustomerId}", dto.CustomerId);
            throw;
        }
    }
}

// Structured logging with Serilog
Log.Information("User {UserId} placed order {OrderId} for {Amount:C}",
    userId, orderId, totalAmount);
```

---
