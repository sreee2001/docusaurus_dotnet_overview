## 54. Event Sourcing

### Short Introduction

Event Sourcing is an architectural pattern where state changes are stored as a sequence of events rather than storing just the current state. Instead of updating records in place, every change is appended as an event to an immutable log, allowing the system to reconstruct any past state and providing a complete audit trail.

### Official Definition

Event Sourcing ensures that all changes to application state are stored as a sequence of events. Not just can we query these events, we can also use the event log to reconstruct past states, and as a foundation to automatically adjust the state to cope with retroactive changes.

### Setup/Usage with .NET 8+ Code

**Installation:**

```bash
dotnet add package EventStore.Client.Grpc.Streams
dotnet add package MediatR
dotnet add package System.Text.Json
```

**Event Base Classes:**

```csharp
// Events/DomainEvent.cs
public abstract record DomainEvent
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public DateTime Timestamp { get; init; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
    public int Version { get; init; } = 1;
}

// Events/BookingEvents.cs
public record BookingCreatedEvent(
    Guid BookingId,
    string CustomerId,
    int RoomId,
    DateTime CheckIn,
    DateTime CheckOut,
    decimal TotalAmount) : DomainEvent;

public record BookingConfirmedEvent(
    Guid BookingId,
    DateTime ConfirmedAt) : DomainEvent;

public record BookingCancelledEvent(
    Guid BookingId,
    string Reason,
    DateTime CancelledAt) : DomainEvent;

public record PaymentProcessedEvent(
    Guid BookingId,
    decimal Amount,
    string PaymentMethod,
    string TransactionId) : DomainEvent;
```

**Event Store Implementation:**

```csharp
// Infrastructure/EventStore.cs
using EventStore.Client;
using System.Text.Json;

public interface IEventStore
{
    Task SaveEventsAsync(string streamName, IEnumerable<DomainEvent> events, long expectedVersion);
    Task<IEnumerable<DomainEvent>> GetEventsAsync(string streamName);
    Task<IEnumerable<DomainEvent>> GetEventsAsync(string streamName, long fromVersion);
}

public class EventStoreRepository : IEventStore
{
    private readonly EventStoreClient _client;
    private readonly JsonSerializerOptions _jsonOptions;

    public EventStoreRepository(EventStoreClient client)
    {
        _client = client;
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };
    }

    public async Task SaveEventsAsync(string streamName, IEnumerable<DomainEvent> events, long expectedVersion)
    {
        var eventData = events.Select(evt => new EventData(
            Uuid.NewUuid(),
            evt.EventType,
            JsonSerializer.SerializeToUtf8Bytes(evt, evt.GetType(), _jsonOptions)
        ));

        var revision = expectedVersion == -1 ? StreamRevision.None : StreamRevision.FromInt64(expectedVersion);

        await _client.AppendToStreamAsync(
            streamName,
            revision,
            eventData);
    }

    public async Task<IEnumerable<DomainEvent>> GetEventsAsync(string streamName)
    {
        return await GetEventsAsync(streamName, 0);
    }

    public async Task<IEnumerable<DomainEvent>> GetEventsAsync(string streamName, long fromVersion)
    {
        var events = new List<DomainEvent>();

        try
        {
            var result = _client.ReadStreamAsync(
                Direction.Forwards,
                streamName,
                StreamPosition.FromInt64(fromVersion));

            await foreach (var evt in result)
            {
                var eventType = Type.GetType($"HotelManagement.Events.{evt.Event.EventType}");
                if (eventType != null)
                {
                    var eventData = JsonSerializer.Deserialize(evt.Event.Data.Span, eventType, _jsonOptions);
                    if (eventData is DomainEvent domainEvent)
                    {
                        events.Add(domainEvent);
                    }
                }
            }
        }
        catch (StreamNotFoundException)
        {
            // Stream doesn't exist yet, return empty list
        }

        return events;
    }
}
```

**Aggregate Root with Event Sourcing:**

```csharp
// Domain/BookingAggregate.cs
public class BookingAggregate
{
    private readonly List<DomainEvent> _uncommittedEvents = new();

    public Guid Id { get; private set; }
    public string CustomerId { get; private set; } = string.Empty;
    public int RoomId { get; private set; }
    public DateTime CheckIn { get; private set; }
    public DateTime CheckOut { get; private set; }
    public decimal TotalAmount { get; private set; }
    public BookingStatus Status { get; private set; }
    public long Version { get; private set; } = -1;

    // For reconstruction from events
    public BookingAggregate() { }

    // For creating new booking
    public BookingAggregate(Guid id, string customerId, int roomId,
        DateTime checkIn, DateTime checkOut, decimal totalAmount)
    {
        var @event = new BookingCreatedEvent(id, customerId, roomId, checkIn, checkOut, totalAmount);
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }

    public void ConfirmBooking()
    {
        if (Status != BookingStatus.Pending)
            throw new InvalidOperationException("Booking can only be confirmed when pending");

        var @event = new BookingConfirmedEvent(Id, DateTime.UtcNow);
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }

    public void CancelBooking(string reason)
    {
        if (Status == BookingStatus.Cancelled)
            throw new InvalidOperationException("Booking is already cancelled");

        var @event = new BookingCancelledEvent(Id, reason, DateTime.UtcNow);
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }

    public void ProcessPayment(decimal amount, string paymentMethod, string transactionId)
    {
        if (Status != BookingStatus.Confirmed)
            throw new InvalidOperationException("Payment can only be processed for confirmed bookings");

        var @event = new PaymentProcessedEvent(Id, amount, paymentMethod, transactionId);
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }

    // Apply events to rebuild state
    public void Apply(DomainEvent @event)
    {
        switch (@event)
        {
            case BookingCreatedEvent created:
                Id = created.BookingId;
                CustomerId = created.CustomerId;
                RoomId = created.RoomId;
                CheckIn = created.CheckIn;
                CheckOut = created.CheckOut;
                TotalAmount = created.TotalAmount;
                Status = BookingStatus.Pending;
                break;

            case BookingConfirmedEvent confirmed:
                Status = BookingStatus.Confirmed;
                break;

            case BookingCancelledEvent cancelled:
                Status = BookingStatus.Cancelled;
                break;

            case PaymentProcessedEvent payment:
                Status = BookingStatus.Paid;
                break;
        }

        Version++;
    }

    public void LoadFromHistory(IEnumerable<DomainEvent> events)
    {
        foreach (var @event in events)
        {
            Apply(@event);
        }
    }

    public IEnumerable<DomainEvent> GetUncommittedEvents() => _uncommittedEvents.AsReadOnly();

    public void MarkEventsAsCommitted() => _uncommittedEvents.Clear();
}

public enum BookingStatus
{
    Pending,
    Confirmed,
    Cancelled,
    Paid
}
```

**Repository Pattern with Event Sourcing:**

```csharp
// Infrastructure/BookingRepository.cs
public interface IBookingRepository
{
    Task<BookingAggregate?> GetByIdAsync(Guid id);
    Task SaveAsync(BookingAggregate aggregate);
}

public class EventSourcedBookingRepository : IBookingRepository
{
    private readonly IEventStore _eventStore;

    public EventSourcedBookingRepository(IEventStore eventStore)
    {
        _eventStore = eventStore;
    }

    public async Task<BookingAggregate?> GetByIdAsync(Guid id)
    {
        var streamName = $"booking-{id}";
        var events = await _eventStore.GetEventsAsync(streamName);

        if (!events.Any())
            return null;

        var aggregate = new BookingAggregate();
        aggregate.LoadFromHistory(events);
        return aggregate;
    }

    public async Task SaveAsync(BookingAggregate aggregate)
    {
        var streamName = $"booking-{aggregate.Id}";
        var uncommittedEvents = aggregate.GetUncommittedEvents();

        if (uncommittedEvents.Any())
        {
            await _eventStore.SaveEventsAsync(streamName, uncommittedEvents, aggregate.Version - uncommittedEvents.Count());
            aggregate.MarkEventsAsCommitted();
        }
    }
}
```

**Projection for Read Models:**

```csharp
// Projections/BookingProjection.cs
public class BookingProjection
{
    public Guid Id { get; set; }
    public string CustomerId { get; set; } = string.Empty;
    public int RoomId { get; set; }
    public DateTime CheckIn { get; set; }
    public DateTime CheckOut { get; set; }
    public decimal TotalAmount { get; set; }
    public string Status { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? ConfirmedAt { get; set; }
    public DateTime? CancelledAt { get; set; }
    public string? CancellationReason { get; set; }
}

// Projections/BookingProjectionHandler.cs
public class BookingProjectionHandler :
    INotificationHandler<BookingCreatedEvent>,
    INotificationHandler<BookingConfirmedEvent>,
    INotificationHandler<BookingCancelledEvent>
{
    private readonly HotelDbContext _context;

    public BookingProjectionHandler(HotelDbContext context)
    {
        _context = context;
    }

    public async Task Handle(BookingCreatedEvent notification, CancellationToken cancellationToken)
    {
        var projection = new BookingProjection
        {
            Id = notification.BookingId,
            CustomerId = notification.CustomerId,
            RoomId = notification.RoomId,
            CheckIn = notification.CheckIn,
            CheckOut = notification.CheckOut,
            TotalAmount = notification.TotalAmount,
            Status = "Pending",
            CreatedAt = notification.Timestamp
        };

        _context.BookingProjections.Add(projection);
        await _context.SaveChangesAsync(cancellationToken);
    }

    public async Task Handle(BookingConfirmedEvent notification, CancellationToken cancellationToken)
    {
        var projection = await _context.BookingProjections
            .FindAsync(notification.BookingId, cancellationToken);

        if (projection != null)
        {
            projection.Status = "Confirmed";
            projection.ConfirmedAt = notification.ConfirmedAt;
            await _context.SaveChangesAsync(cancellationToken);
        }
    }

    public async Task Handle(BookingCancelledEvent notification, CancellationToken cancellationToken)
    {
        var projection = await _context.BookingProjections
            .FindAsync(notification.BookingId, cancellationToken);

        if (projection != null)
        {
            projection.Status = "Cancelled";
            projection.CancelledAt = notification.CancelledAt;
            projection.CancellationReason = notification.Reason;
            await _context.SaveChangesAsync(cancellationToken);
        }
    }
}
```

### Use Cases

- **Audit Requirements**: Complete audit trail of all changes
- **Financial Systems**: Immutable transaction history
- **Temporal Queries**: "What was the state at time X?"
- **Event-Driven Architectures**: Natural fit for event-driven systems
- **Debugging and Analytics**: Rich historical data for analysis
- **Regulatory Compliance**: Immutable record of business events

### When to Use vs When Not to Use

**Use Event Sourcing when:**

- Need complete audit trail
- Temporal queries are important
- Building event-driven systems
- Regulatory compliance requirements
- Complex business processes with state transitions
- Need to rebuild state from history

**Consider alternatives when:**

- Simple CRUD operations
- Limited development team experience
- Real-time read performance is critical
- Storage costs are a major concern
- Simple domain logic
- Need for simplicity outweighs benefits

### Market Alternatives & Pros/Cons

**Alternatives:**

- **Change Data Capture (CDC)**: Database-level change tracking
- **Audit Tables**: Traditional audit logging
- **Event Streaming**: Kafka, Azure Event Hubs
- **CQRS without Event Sourcing**: Separate read/write models
- **Traditional State Storage**: Standard CRUD operations

**Pros:**

- Complete audit trail
- Natural fit for event-driven architecture
- Ability to reconstruct any past state
- Excellent for debugging and analysis
- Scalable event processing
- Strong consistency guarantees

**Cons:**

- Increased complexity
- Event schema evolution challenges
- Storage requirements grow over time
- Query complexity for current state
- Learning curve for development teams
- Eventual consistency in projections

### Complete Runnable Sample

**Program.cs Integration:**

```csharp
// Program.cs
using EventStore.Client;

var builder = WebApplication.CreateBuilder(args);

// Add EventStore client
var eventStoreSettings = EventStoreClientSettings.Create("esdb://localhost:2113?tls=false");
builder.Services.AddSingleton(new EventStoreClient(eventStoreSettings));

// Add Event Sourcing services
builder.Services.AddScoped<IEventStore, EventStoreRepository>();
builder.Services.AddScoped<IBookingRepository, EventSourcedBookingRepository>();

// Add MediatR for projections
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));

// Add Entity Framework for projections
builder.Services.AddDbContext<HotelDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

// Example usage
app.MapPost("/bookings", async (CreateBookingCommand command, IBookingRepository repository) =>
{
    var booking = new BookingAggregate(
        Guid.NewGuid(),
        command.CustomerId,
        command.RoomId,
        command.CheckIn,
        command.CheckOut,
        command.TotalAmount);

    await repository.SaveAsync(booking);
    return Results.Ok(booking.Id);
});

app.MapPut("/bookings/{id}/confirm", async (Guid id, IBookingRepository repository) =>
{
    var booking = await repository.GetByIdAsync(id);
    if (booking == null) return Results.NotFound();

    booking.ConfirmBooking();
    await repository.SaveAsync(booking);
    return Results.Ok();
});

// Docker Compose for EventStore
/*
version: '3.8'
services:
  eventstore:
    image: eventstore/eventstore:21.10.0-buster-slim
    environment:
      - EVENTSTORE_CLUSTER_SIZE=1
      - EVENTSTORE_RUN_PROJECTIONS=All
      - EVENTSTORE_START_STANDARD_PROJECTIONS=true
      - EVENTSTORE_EXT_TCP_PORT=1113
      - EVENTSTORE_HTTP_PORT=2113
      - EVENTSTORE_INSECURE=true
      - EVENTSTORE_ENABLE_EXTERNAL_TCP=true
      - EVENTSTORE_ENABLE_ATOM_PUB_OVER_HTTP=true
    ports:
      - "1113:1113"
      - "2113:2113"
    volumes:
      - eventstore-volume-data:/var/lib/eventstore
      - eventstore-volume-logs:/var/log/eventstore

volumes:
  eventstore-volume-data:
  eventstore-volume-logs:
*/

app.Run();

public record CreateBookingCommand(
    string CustomerId,
    int RoomId,
    DateTime CheckIn,
    DateTime CheckOut,
    decimal TotalAmount);
```
