---
slug: dependency_injection
title: Dependency Injection (DI)
tags: [dotnet, core, dependency injection, DI, IOC]
---

# Dependency Injection

## Short Introduction

Dependency Injection is a design pattern that implements Inversion of Control for resolving dependencies, making applications more modular and testable.

## Official Definition

Dependency Injection (DI) is a technique for achieving Inversion of Control (IoC) between classes and their dependencies, where dependencies are provided to a class rather than created by the class itself.

## Usage

```csharp
// Service interface
public interface IEmailService
{
    Task SendEmailAsync(string to, string subject, string body);
}

// Service implementation
public class EmailService : IEmailService
{
    private readonly ILogger<EmailService> _logger;
    private readonly IConfiguration _configuration;

    public EmailService(ILogger<EmailService> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }

    public async Task SendEmailAsync(string to, string subject, string body)
    {
        _logger.LogInformation("Sending email to {EmailAddress}", to);
        // Email sending logic
        await Task.CompletedTask;
    }
}

// Service registration
builder.Services.AddScoped<IEmailService, EmailService>();
builder.Services.AddSingleton<IMemoryCache, MemoryCache>();
builder.Services.AddTransient<IProductService, ProductService>();
```

## Service Lifetimes

- **Singleton**: Created once and shared
- **Scoped**: Created once per request
- **Transient**: Created every time requested

## Use Cases

- Loose coupling between components
- Unit testing with mock dependencies
- Configuration injection
- Cross-cutting concerns

## Sample Usage

```csharp
// Controller with DI
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly IEmailService _emailService;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(
        IOrderService orderService,
        IEmailService emailService,
        ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _emailService = emailService;
        _logger = logger;
    }

    [HttpPost]
    public async Task<ActionResult<Order>> CreateOrder(CreateOrderDto dto)
    {
        try
        {
            var order = await _orderService.CreateOrderAsync(dto);
            await _emailService.SendEmailAsync(
                dto.CustomerEmail,
                "Order Confirmation",
                $"Your order {order.Id} has been created");

            return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating order");
            return StatusCode(500, "Internal server error");
        }
    }
}
```
