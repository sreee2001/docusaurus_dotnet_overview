## 44. Microservices Architecture

### Short Introduction

Microservices architecture is a design approach that structures an application as a collection of loosely coupled, independently deployable services. Each service is responsible for a specific business capability and communicates with other services through well-defined APIs, enabling teams to develop, deploy, and scale services independently.

### Official Definition

Microservices architecture is an approach to developing a single application as a suite of small services, each running in its own process and communicating with lightweight mechanisms, often an HTTP resource API. These services are built around business capabilities and independently deployable by fully automated deployment machinery.

### Setup/Usage with .NET 8+ Code

**Basic Microservice Structure:**

```csharp
// BookingService/Program.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddDbContext<BookingDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
    });

builder.Services.AddAuthorization();
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Add service-specific dependencies
builder.Services.AddScoped<IBookingRepository, BookingRepository>();
builder.Services.AddScoped<IBookingService, BookingService>();
builder.Services.AddHttpClient<IRoomService, RoomServiceClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Services:RoomService:BaseUrl"]);
});

// Add health checks
builder.Services.AddHealthChecks()
    .AddDbContext<BookingDbContext>()
    .AddUrlGroup(new Uri($"{builder.Configuration["Services:RoomService:BaseUrl"]}/health"), "room-service");

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health");

app.Run();

// Models/Booking.cs
public class Booking
{
    public int Id { get; set; }
    public string CustomerId { get; set; } = string.Empty;
    public int RoomId { get; set; }
    public DateTime CheckIn { get; set; }
    public DateTime CheckOut { get; set; }
    public decimal TotalAmount { get; set; }
    public BookingStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
}

public enum BookingStatus
{
    Pending,
    Confirmed,
    Cancelled,
    CheckedIn,
    CheckedOut
}

// Controllers/BookingsController.cs
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class BookingsController : ControllerBase
{
    private readonly IBookingService _bookingService;
    private readonly IRoomService _roomService;
    private readonly ILogger<BookingsController> _logger;

    public BookingsController(
        IBookingService bookingService,
        IRoomService roomService,
        ILogger<BookingsController> logger)
    {
        _bookingService = bookingService;
        _roomService = roomService;
        _logger = logger;
    }

    [HttpPost]
    public async Task<ActionResult<BookingDto>> CreateBooking([FromBody] CreateBookingRequest request)
    {
        try
        {
            // Check room availability with Room Service
            var roomAvailability = await _roomService.CheckAvailabilityAsync(
                request.RoomId, request.CheckIn, request.CheckOut);

            if (!roomAvailability.IsAvailable)
            {
                return BadRequest("Room is not available for the selected dates");
            }

            var booking = await _bookingService.CreateBookingAsync(request);
            return Ok(booking);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating booking");
            return StatusCode(500, "An error occurred while creating the booking");
        }
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<BookingDto>> GetBooking(int id)
    {
        var booking = await _bookingService.GetBookingByIdAsync(id);
        if (booking == null)
        {
            return NotFound();
        }

        return Ok(booking);
    }

    [HttpGet("customer/{customerId}")]
    public async Task<ActionResult<IEnumerable<BookingDto>>> GetCustomerBookings(string customerId)
    {
        var bookings = await _bookingService.GetBookingsByCustomerAsync(customerId);
        return Ok(bookings);
    }
}
```

**Service Communication Pattern:**

```csharp
// Services/RoomServiceClient.cs
public interface IRoomService
{
    Task<RoomAvailabilityResponse> CheckAvailabilityAsync(int roomId, DateTime checkIn, DateTime checkOut);
    Task<RoomDetailsResponse> GetRoomDetailsAsync(int roomId);
}

public class RoomServiceClient : IRoomService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<RoomServiceClient> _logger;

    public RoomServiceClient(HttpClient httpClient, ILogger<RoomServiceClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<RoomAvailabilityResponse> CheckAvailabilityAsync(int roomId, DateTime checkIn, DateTime checkOut)
    {
        try
        {
            var request = new
            {
                RoomId = roomId,
                CheckIn = checkIn,
                CheckOut = checkOut
            };

            var response = await _httpClient.PostAsJsonAsync("/api/rooms/check-availability", request);
            response.EnsureSuccessStatusCode();

            var result = await response.Content.ReadFromJsonAsync<RoomAvailabilityResponse>();
            return result ?? new RoomAvailabilityResponse { IsAvailable = false };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error checking room availability for room {RoomId}", roomId);
            throw;
        }
    }

    public async Task<RoomDetailsResponse> GetRoomDetailsAsync(int roomId)
    {
        try
        {
            var response = await _httpClient.GetAsync($"/api/rooms/{roomId}");
            response.EnsureSuccessStatusCode();

            var result = await response.Content.ReadFromJsonAsync<RoomDetailsResponse>();
            return result ?? throw new InvalidOperationException("Room not found");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting room details for room {RoomId}", roomId);
            throw;
        }
    }
}

// DTOs/ServiceDtos.cs
public class RoomAvailabilityResponse
{
    public bool IsAvailable { get; set; }
    public decimal PricePerNight { get; set; }
    public string Message { get; set; } = string.Empty;
}

public class RoomDetailsResponse
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Type { get; set; } = string.Empty;
    public decimal PricePerNight { get; set; }
    public int MaxOccupancy { get; set; }
    public List<string> Amenities { get; set; } = new();
}
```

### Use Cases

- **Large-Scale Applications**: Complex systems with multiple business domains
- **Team Autonomy**: Different teams working on different parts of the system
- **Technology Diversity**: Different services using different technologies or databases
- **Independent Scaling**: Services with different performance requirements
- **Fault Isolation**: Preventing failures in one service from affecting others
- **Continuous Deployment**: Independent release cycles for different services

### When to Use vs When Not to Use

**Use Microservices when:**

- Application has clear business domain boundaries
- Team is large enough to support multiple services
- Need independent scaling and deployment
- Different parts of system have different technology requirements
- High availability and fault tolerance are critical
- Organization can handle operational complexity

**Consider alternatives when:**

- Small team or simple application
- Unclear domain boundaries
- Limited operational expertise
- Network latency is critical
- Strong consistency requirements across domains
- Development speed is more important than scalability

### Market Alternatives & Pros/Cons

**Alternatives:**

- **Monolithic Architecture**: Single deployable unit
- **Modular Monolith**: Modular structure within single deployment
- **Service-Oriented Architecture (SOA)**: Enterprise service approach
- **Serverless Functions**: Function-as-a-Service approach
- **Event-Driven Architecture**: Async communication patterns

**Pros:**

- Independent development and deployment
- Technology diversity and flexibility
- Better fault isolation
- Easier scaling of individual components
- Team autonomy and ownership
- Better alignment with business domains

**Cons:**

- Increased operational complexity
- Network latency and communication overhead
- Data consistency challenges
- Testing complexity
- Service discovery and configuration management
- Higher infrastructure costs

### Complete Runnable Sample

**Complete Microservices Implementation:**

```csharp
// Shared/Events/DomainEvents.cs
public abstract class DomainEvent
{
    public DateTime OccurredOn { get; protected set; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
}

public class BookingCreatedEvent : DomainEvent
{
    public int BookingId { get; set; }
    public string CustomerId { get; set; } = string.Empty;
    public int RoomId { get; set; }
    public DateTime CheckIn { get; set; }
    public DateTime CheckOut { get; set; }
    public decimal TotalAmount { get; set; }
}

public class BookingCancelledEvent : DomainEvent
{
    public int BookingId { get; set; }
    public string CustomerId { get; set; } = string.Empty;
    public string Reason { get; set; } = string.Empty;
}

// Shared/Infrastructure/EventBus.cs
public interface IEventBus
{
    Task PublishAsync<T>(T @event) where T : DomainEvent;
    void Subscribe<T>(Func<T, Task> handler) where T : DomainEvent;
}

public class InMemoryEventBus : IEventBus
{
    private readonly Dictionary<Type, List<Func<object, Task>>> _handlers = new();
    private readonly ILogger<InMemoryEventBus> _logger;

    public InMemoryEventBus(ILogger<InMemoryEventBus> logger)
    {
        _logger = logger;
    }

    public async Task PublishAsync<T>(T @event) where T : DomainEvent
    {
        var eventType = typeof(T);
        if (_handlers.TryGetValue(eventType, out var handlers))
        {
            var tasks = handlers.Select(handler => handler(@event));
            await Task.WhenAll(tasks);
            _logger.LogInformation("Published event {EventType}", eventType.Name);
        }
    }

    public void Subscribe<T>(Func<T, Task> handler) where T : DomainEvent
    {
        var eventType = typeof(T);
        if (!_handlers.ContainsKey(eventType))
        {
            _handlers[eventType] = new List<Func<object, Task>>();
        }

        _handlers[eventType].Add(@event => handler((T)@event));
        _logger.LogInformation("Subscribed to event {EventType}", eventType.Name);
    }
}

// BookingService/Services/BookingService.cs
public interface IBookingService
{
    Task<BookingDto> CreateBookingAsync(CreateBookingRequest request);
    Task<BookingDto?> GetBookingByIdAsync(int id);
    Task<IEnumerable<BookingDto>> GetBookingsByCustomerAsync(string customerId);
    Task<BookingDto> CancelBookingAsync(int id, string reason);
}

public class BookingService : IBookingService
{
    private readonly IBookingRepository _repository;
    private readonly IEventBus _eventBus;
    private readonly IRoomService _roomService;
    private readonly ILogger<BookingService> _logger;

    public BookingService(
        IBookingRepository repository,
        IEventBus eventBus,
        IRoomService roomService,
        ILogger<BookingService> logger)
    {
        _repository = repository;
        _eventBus = eventBus;
        _roomService = roomService;
        _logger = logger;
    }

    public async Task<BookingDto> CreateBookingAsync(CreateBookingRequest request)
    {
        try
        {
            // Validate room availability
            var availability = await _roomService.CheckAvailabilityAsync(
                request.RoomId, request.CheckIn, request.CheckOut);

            if (!availability.IsAvailable)
            {
                throw new InvalidOperationException("Room is not available for the selected dates");
            }

            // Calculate total amount
            var nights = (request.CheckOut - request.CheckIn).Days;
            var totalAmount = nights * availability.PricePerNight;

            // Create booking
            var booking = new Booking
            {
                CustomerId = request.CustomerId,
                RoomId = request.RoomId,
                CheckIn = request.CheckIn,
                CheckOut = request.CheckOut,
                TotalAmount = totalAmount,
                Status = BookingStatus.Pending,
                CreatedAt = DateTime.UtcNow
            };

            await _repository.AddAsync(booking);
            await _repository.SaveChangesAsync();

            // Publish domain event
            await _eventBus.PublishAsync(new BookingCreatedEvent
            {
                BookingId = booking.Id,
                CustomerId = booking.CustomerId,
                RoomId = booking.RoomId,
                CheckIn = booking.CheckIn,
                CheckOut = booking.CheckOut,
                TotalAmount = booking.TotalAmount
            });

            _logger.LogInformation("Booking created successfully with ID {BookingId}", booking.Id);

            return MapToDto(booking);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating booking for customer {CustomerId}", request.CustomerId);
            throw;
        }
    }

    public async Task<BookingDto?> GetBookingByIdAsync(int id)
    {
        var booking = await _repository.GetByIdAsync(id);
        return booking != null ? MapToDto(booking) : null;
    }

    public async Task<IEnumerable<BookingDto>> GetBookingsByCustomerAsync(string customerId)
    {
        var bookings = await _repository.GetByCustomerIdAsync(customerId);
        return bookings.Select(MapToDto);
    }

    public async Task<BookingDto> CancelBookingAsync(int id, string reason)
    {
        var booking = await _repository.GetByIdAsync(id);
        if (booking == null)
        {
            throw new InvalidOperationException("Booking not found");
        }

        if (booking.Status != BookingStatus.Pending && booking.Status != BookingStatus.Confirmed)
        {
            throw new InvalidOperationException("Cannot cancel booking in current status");
        }

        booking.Status = BookingStatus.Cancelled;
        await _repository.SaveChangesAsync();

        // Publish cancellation event
        await _eventBus.PublishAsync(new BookingCancelledEvent
        {
            BookingId = booking.Id,
            CustomerId = booking.CustomerId,
            Reason = reason
        });

        _logger.LogInformation("Booking {BookingId} cancelled", booking.Id);

        return MapToDto(booking);
    }

    private static BookingDto MapToDto(Booking booking)
    {
        return new BookingDto
        {
            Id = booking.Id,
            CustomerId = booking.CustomerId,
            RoomId = booking.RoomId,
            CheckIn = booking.CheckIn,
            CheckOut = booking.CheckOut,
            TotalAmount = booking.TotalAmount,
            Status = booking.Status.ToString(),
            CreatedAt = booking.CreatedAt
        };
    }
}

// docker-compose.yml for multi-service setup
version: '3.8'
services:
  booking-service:
    build:
      context: ./BookingService
      dockerfile: Dockerfile
    ports:
      - "5001:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=booking-db;Database=BookingDB;User Id=sa;Password=YourPassword123!;TrustServerCertificate=true
      - Services__RoomService__BaseUrl=http://room-service:8080
    depends_on:
      - booking-db
      - room-service

  room-service:
    build:
      context: ./RoomService
      dockerfile: Dockerfile
    ports:
      - "5002:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=room-db;Database=RoomDB;User Id=sa;Password=YourPassword123!;TrustServerCertificate=true
    depends_on:
      - room-db

  customer-service:
    build:
      context: ./CustomerService
      dockerfile: Dockerfile
    ports:
      - "5003:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=customer-db;Database=CustomerDB;User Id=sa;Password=YourPassword123!;TrustServerCertificate=true
    depends_on:
      - customer-db

  booking-db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!
    ports:
      - "1433:1433"

  room-db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!
    ports:
      - "1434:1433"

  customer-db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!
    ports:
      - "1435:1433"

networks:
  default:
    name: hotel-management-network
```
