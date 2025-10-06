## 53. GraphQL (HotChocolate)

## Short Introduction

GraphQL is a query language and runtime for APIs that allows clients to request exactly the data they need. HotChocolate is a powerful .NET GraphQL server that provides a code-first approach to building GraphQL APIs, with features like real-time subscriptions, DataLoader for efficient data fetching, and comprehensive tooling.

## Official Definition

GraphQL is a query language for APIs and a runtime for fulfilling those queries with your existing data. GraphQL provides a complete and understandable description of the data in your API, gives clients the power to ask for exactly what they need and nothing more.

## Setup/Usage with .NET 8+ Code

**Installation:**

```bash
dotnet add package HotChocolate.AspNetCore
dotnet add package HotChocolate.Data.EntityFramework
dotnet add package HotChocolate.Subscriptions.InMemory
```

**GraphQL Schema Setup:**

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add Entity Framework
builder.Services.AddDbContext<HotelDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Add GraphQL
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>()
    .AddType<RoomType>()
    .AddType<BookingType>()
    .AddFiltering()
    .AddSorting()
    .AddProjections()
    .AddInMemorySubscriptions();

var app = builder.Build();

// Configure GraphQL endpoint
app.MapGraphQL("/graphql");

// Add GraphQL IDE (Banana Cake Pop)
if (app.Environment.IsDevelopment())
{
    app.MapGraphQLVoyager("/graphql-voyager");
    app.MapBananaCakePop("/graphql-ui");
}

app.Run();

// GraphQL/Types/RoomType.cs
public class RoomType : ObjectType<Room>
{
    protected override void Configure(IObjectTypeDescriptor<Room> descriptor)
    {
        descriptor.Description("Represents a hotel room");

        descriptor
            .Field(r => r.Id)
            .Description("The unique identifier of the room");

        descriptor
            .Field(r => r.Bookings)
            .Description("All bookings for this room")
            .UseFiltering()
            .UseSorting();

        descriptor
            .Field("availability")
            .Type<NonNullType<BooleanType>>()
            .Description("Current availability status")
            .Resolve(context =>
            {
                var room = context.Parent<Room>();
                var now = DateTime.UtcNow;
                return !room.Bookings.Any(b =>
                    b.CheckIn <= now && b.CheckOut >= now &&
                    b.Status == "Confirmed");
            });
    }
}

// GraphQL/Types/BookingType.cs
public class BookingType : ObjectType<Booking>
{
    protected override void Configure(IObjectTypeDescriptor<Booking> descriptor)
    {
        descriptor.Description("Represents a hotel booking");

        descriptor
            .Field(b => b.Room)
            .Description("The room associated with this booking");

        descriptor
            .Field("nights")
            .Type<NonNullType<IntType>>()
            .Description("Number of nights for this booking")
            .Resolve(context =>
            {
                var booking = context.Parent<Booking>();
                return (booking.CheckOut - booking.CheckIn).Days;
            });
    }
}
```

**Query Implementation:**

```csharp
// GraphQL/Query.cs
[UseProjection]
[UseFiltering]
[UseSorting]
public class Query
{
    public IQueryable<Room> GetRooms([Service] HotelDbContext context) =>
        context.Rooms.AsQueryable();

    public IQueryable<Booking> GetBookings([Service] HotelDbContext context) =>
        context.Bookings.AsQueryable();

    public async Task<Room?> GetRoomById(
        int id,
        [Service] HotelDbContext context,
        CancellationToken cancellationToken) =>
        await context.Rooms
            .Include(r => r.Bookings)
            .FirstOrDefaultAsync(r => r.Id == id, cancellationToken);

    [UseFirstOrDefault]
    [UseProjection]
    public IQueryable<Booking> GetBookingById(
        int id,
        [Service] HotelDbContext context) =>
        context.Bookings.Where(b => b.Id == id);

    public async Task<IEnumerable<Room>> GetAvailableRooms(
        DateTime checkIn,
        DateTime checkOut,
        [Service] HotelDbContext context,
        CancellationToken cancellationToken)
    {
        var conflictingBookings = await context.Bookings
            .Where(b => b.CheckIn < checkOut && b.CheckOut > checkIn && b.Status == "Confirmed")
            .Select(b => b.RoomId)
            .ToListAsync(cancellationToken);

        return await context.Rooms
            .Where(r => !conflictingBookings.Contains(r.Id))
            .ToListAsync(cancellationToken);
    }

    // DataLoader example for efficient N+1 problem resolution
    public async Task<IEnumerable<Booking>> GetBookingsByCustomerId(
        string customerId,
        BookingsByCustomerDataLoader dataLoader,
        CancellationToken cancellationToken) =>
        await dataLoader.LoadAsync(customerId, cancellationToken);
}

// GraphQL/Mutation.cs
public class Mutation
{
    public async Task<CreateBookingPayload> CreateBooking(
        CreateBookingInput input,
        [Service] HotelDbContext context,
        [Service] ITopicEventSender eventSender,
        CancellationToken cancellationToken)
    {
        // Validate input
        if (input.CheckOut <= input.CheckIn)
        {
            return new CreateBookingPayload(
                null,
                new UserError("Check-out date must be after check-in date", "INVALID_DATES"));
        }

        // Check room availability
        var room = await context.Rooms.FindAsync(input.RoomId, cancellationToken);
        if (room == null)
        {
            return new CreateBookingPayload(
                null,
                new UserError("Room not found", "ROOM_NOT_FOUND"));
        }

        var conflictingBooking = await context.Bookings
            .AnyAsync(b => b.RoomId == input.RoomId &&
                          b.CheckIn < input.CheckOut &&
                          b.CheckOut > input.CheckIn &&
                          b.Status == "Confirmed", cancellationToken);

        if (conflictingBooking)
        {
            return new CreateBookingPayload(
                null,
                new UserError("Room is not available for the selected dates", "ROOM_UNAVAILABLE"));
        }

        // Create booking
        var booking = new Booking
        {
            CustomerId = input.CustomerId,
            RoomId = input.RoomId,
            CheckIn = input.CheckIn,
            CheckOut = input.CheckOut,
            Guests = input.Guests,
            TotalAmount = room.PricePerNight * (input.CheckOut - input.CheckIn).Days,
            Status = "Confirmed",
            CreatedAt = DateTime.UtcNow
        };

        context.Bookings.Add(booking);
        await context.SaveChangesAsync(cancellationToken);

        // Send subscription notification
        await eventSender.SendAsync(nameof(Subscription.OnBookingCreated), booking, cancellationToken);

        return new CreateBookingPayload(booking, null);
    }

    public async Task<UpdateBookingPayload> UpdateBooking(
        UpdateBookingInput input,
        [Service] HotelDbContext context,
        CancellationToken cancellationToken)
    {
        var booking = await context.Bookings.FindAsync(input.Id, cancellationToken);
        if (booking == null)
        {
            return new UpdateBookingPayload(
                null,
                new UserError("Booking not found", "BOOKING_NOT_FOUND"));
        }

        if (input.Status != null)
        {
            booking.Status = input.Status;
        }

        await context.SaveChangesAsync(cancellationToken);

        return new UpdateBookingPayload(booking, null);
    }
}

// GraphQL/Subscription.cs
public class Subscription
{
    [Subscribe]
    [Topic(nameof(OnBookingCreated))]
    public Booking OnBookingCreated([EventMessage] Booking booking) => booking;

    [Subscribe]
    public async IAsyncEnumerable<string> OnRoomStatusChange(
        int roomId,
        [Service] HotelDbContext context,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        // Simulate real-time room status updates
        while (!cancellationToken.IsCancellationRequested)
        {
            var room = await context.Rooms.FindAsync(roomId, cancellationToken);
            if (room != null)
            {
                yield return $"Room {roomId} status: {(room.IsAvailable ? "Available" : "Occupied")}";
            }

            await Task.Delay(TimeSpan.FromSeconds(30), cancellationToken);
        }
    }
}
```

**Input/Output Types:**

```csharp
// GraphQL/Inputs/CreateBookingInput.cs
public record CreateBookingInput(
    string CustomerId,
    int RoomId,
    DateTime CheckIn,
    DateTime CheckOut,
    int Guests);

public record UpdateBookingInput(
    int Id,
    string? Status);

// GraphQL/Payloads/BookingPayloads.cs
public record CreateBookingPayload(Booking? Booking, UserError? Error);
public record UpdateBookingPayload(Booking? Booking, UserError? Error);

public record UserError(string Message, string Code);

// GraphQL/DataLoaders/BookingsByCustomerDataLoader.cs
public class BookingsByCustomerDataLoader : BatchDataLoader<string, Booking[]>
{
    private readonly IDbContextFactory<HotelDbContext> _dbContextFactory;

    public BookingsByCustomerDataLoader(
        IDbContextFactory<HotelDbContext> dbContextFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _dbContextFactory = dbContextFactory;
    }

    protected override async Task<IReadOnlyDictionary<string, Booking[]>> LoadBatchAsync(
        IReadOnlyList<string> keys,
        CancellationToken cancellationToken)
    {
        await using var context = _dbContextFactory.CreateDbContext();

        var bookings = await context.Bookings
            .Where(b => keys.Contains(b.CustomerId))
            .ToListAsync(cancellationToken);

        return bookings
            .GroupBy(b => b.CustomerId)
            .ToDictionary(g => g.Key, g => g.ToArray());
    }
}
```

### Use Cases

- **Flexible APIs**: Clients request exactly the data they need
- **Mobile Applications**: Reduce bandwidth usage with precise queries
- **Microservices Aggregation**: Single endpoint aggregating multiple services
- **Real-time Applications**: Subscriptions for live data updates
- **Developer Experience**: Strong typing and excellent tooling
- **API Evolution**: Add fields without versioning

## When to Use vs When Not to Use

**Use GraphQL when:**

- Clients have diverse data requirements
- Need to aggregate data from multiple sources
- Building mobile or bandwidth-constrained applications
- Want strong typing and schema validation
- Need real-time subscriptions
- Have complex relational data

**Consider alternatives when:**

- Simple CRUD operations
- Caching is critical (REST caching is more mature)
- File uploads are primary use case
- Team lacks GraphQL expertise
- Need simple HTTP status codes for errors

### Market Alternatives & Pros/Cons

### Alternatives:

- **Apollo Server**: Popular GraphQL server for Node.js
- **Hasura**: Auto-generated GraphQL APIs from databases
- **AWS AppSync**: Managed GraphQL service
- **Relay**: Facebook's GraphQL client
- **GraphQL Yoga**: Fully-featured GraphQL server
- **Strawberry Shake**: GraphQL client for .NET

### Pros:

- Single endpoint for all data needs
- Strong typing and schema validation
- Eliminates over-fetching and under-fetching
- Excellent developer experience
- Real-time subscriptions
- Introspection and tooling

### Cons:

- Complexity in caching
- Potential for expensive queries
- Learning curve for teams
- N+1 query problems if not handled properly
- Less mature ecosystem than REST

### Complete Runnable Sample

**Sample GraphQL Queries:**

```graphql
# Query available rooms with specific fields
query GetAvailableRooms($checkIn: DateTime!, $checkOut: DateTime!) {
  availableRooms(checkIn: $checkIn, checkOut: $checkOut) {
    id
    name
    type
    pricePerNight
    maxOccupancy
    amenities
    availability
  }
}

# Query rooms with filtering and sorting
query GetRoomsFiltered {
  rooms(where: { pricePerNight: { lte: 200 } }, order: { pricePerNight: ASC }) {
    id
    name
    pricePerNight
    bookings(where: { status: { eq: "Confirmed" } }) {
      id
      checkIn
      checkOut
      nights
    }
  }
}

# Create booking mutation
mutation CreateBooking($input: CreateBookingInput!) {
  createBooking(input: $input) {
    booking {
      id
      customerId
      room {
        name
        type
      }
      checkIn
      checkOut
      totalAmount
      status
    }
    error {
      message
      code
    }
  }
}

# Subscribe to booking events
subscription BookingCreated {
  onBookingCreated {
    id
    customerId
    room {
      name
    }
    totalAmount
    createdAt
  }
}
```
