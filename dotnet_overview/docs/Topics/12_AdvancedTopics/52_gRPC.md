## 52. gRPC

### Short Introduction

gRPC is a modern, open-source, high-performance Remote Procedure Call (RPC) framework that can run anywhere. It uses HTTP/2 for transport, Protocol Buffers as the interface description language, and provides features like authentication, bidirectional streaming, flow control, blocking or nonblocking bindings, and cancellation and timeouts.

### Official Definition

gRPC is a language-agnostic, high-performance Remote Procedure Call (RPC) framework. gRPC uses HTTP/2 as its transport protocol and Protocol Buffers (protobuf) as its message format, enabling efficient communication between services in distributed systems.

### Setup/Usage with .NET 8+ Code

**Installation:**

```bash
dotnet add package Grpc.AspNetCore
dotnet add package Grpc.Tools
dotnet add package Google.Protobuf
```

**Proto Definition:**

```protobuf
// Protos/hotel.proto
syntax = "proto3";

option csharp_namespace = "HotelManagement.Grpc";

package hotel;

// Hotel service definition
service HotelService {
  rpc GetRoom (GetRoomRequest) returns (RoomResponse);
  rpc CreateBooking (CreateBookingRequest) returns (BookingResponse);
  rpc GetBookings (GetBookingsRequest) returns (stream BookingResponse);
  rpc BookingStream (stream CreateBookingRequest) returns (stream BookingResponse);
}

// Messages
message GetRoomRequest {
  int32 room_id = 1;
}

message RoomResponse {
  int32 id = 1;
  string name = 2;
  string type = 3;
  double price_per_night = 4;
  int32 max_occupancy = 5;
  repeated string amenities = 6;
  bool is_available = 7;
}

message CreateBookingRequest {
  string customer_id = 1;
  int32 room_id = 2;
  string check_in = 3;  // ISO 8601 date string
  string check_out = 4; // ISO 8601 date string
  int32 guests = 5;
}

message BookingResponse {
  int32 id = 1;
  string customer_id = 2;
  int32 room_id = 3;
  string check_in = 4;
  string check_out = 5;
  double total_amount = 6;
  string status = 7;
  string created_at = 8;
}

message GetBookingsRequest {
  string customer_id = 1;
  int32 page = 2;
  int32 page_size = 3;
}
```

**gRPC Server Implementation:**

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add gRPC services
builder.Services.AddGrpc(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.MaxReceiveMessageSize = 4 * 1024 * 1024; // 4MB
    options.MaxSendMessageSize = 4 * 1024 * 1024; // 4MB
});

// Add application services
builder.Services.AddScoped<IRoomRepository, RoomRepository>();
builder.Services.AddScoped<IBookingRepository, BookingRepository>();

var app = builder.Build();

// Configure gRPC endpoints
app.MapGrpcService<HotelGrpcService>();

// Enable gRPC-Web for browser clients
app.UseGrpcWeb(new GrpcWebOptions { DefaultEnabled = true });

// Add gRPC reflection for development
if (app.Environment.IsDevelopment())
{
    app.MapGrpcReflectionService();
}

app.Run();

// Services/HotelGrpcService.cs
using Grpc.Core;
using HotelManagement.Grpc;

public class HotelGrpcService : HotelService.HotelServiceBase
{
    private readonly IRoomRepository _roomRepository;
    private readonly IBookingRepository _bookingRepository;
    private readonly ILogger<HotelGrpcService> _logger;

    public HotelGrpcService(
        IRoomRepository roomRepository,
        IBookingRepository bookingRepository,
        ILogger<HotelGrpcService> logger)
    {
        _roomRepository = roomRepository;
        _bookingRepository = bookingRepository;
        _logger = logger;
    }

    public override async Task<RoomResponse> GetRoom(GetRoomRequest request, ServerCallContext context)
    {
        try
        {
            var room = await _roomRepository.GetByIdAsync(request.RoomId);
            if (room == null)
            {
                throw new RpcException(new Status(StatusCode.NotFound, $"Room {request.RoomId} not found"));
            }

            return new RoomResponse
            {
                Id = room.Id,
                Name = room.Name,
                Type = room.Type,
                PricePerNight = (double)room.PricePerNight,
                MaxOccupancy = room.MaxOccupancy,
                IsAvailable = room.IsAvailable
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting room {RoomId}", request.RoomId);
            throw new RpcException(new Status(StatusCode.Internal, "Internal server error"));
        }
    }

    public override async Task<BookingResponse> CreateBooking(CreateBookingRequest request, ServerCallContext context)
    {
        try
        {
            // Validate dates
            if (!DateTime.TryParse(request.CheckIn, out var checkIn) ||
                !DateTime.TryParse(request.CheckOut, out var checkOut))
            {
                throw new RpcException(new Status(StatusCode.InvalidArgument, "Invalid date format"));
            }

            if (checkOut <= checkIn)
            {
                throw new RpcException(new Status(StatusCode.InvalidArgument, "Check-out must be after check-in"));
            }

            // Check room availability
            var room = await _roomRepository.GetByIdAsync(request.RoomId);
            if (room == null || !room.IsAvailable)
            {
                throw new RpcException(new Status(StatusCode.FailedPrecondition, "Room not available"));
            }

            // Create booking
            var booking = new Booking
            {
                CustomerId = request.CustomerId,
                RoomId = request.RoomId,
                CheckIn = checkIn,
                CheckOut = checkOut,
                Guests = request.Guests,
                TotalAmount = (decimal)room.PricePerNight * (checkOut - checkIn).Days,
                Status = "Confirmed",
                CreatedAt = DateTime.UtcNow
            };

            await _bookingRepository.AddAsync(booking);

            return new BookingResponse
            {
                Id = booking.Id,
                CustomerId = booking.CustomerId,
                RoomId = booking.RoomId,
                CheckIn = booking.CheckIn.ToString("O"),
                CheckOut = booking.CheckOut.ToString("O"),
                TotalAmount = (double)booking.TotalAmount,
                Status = booking.Status,
                CreatedAt = booking.CreatedAt.ToString("O")
            };
        }
        catch (RpcException)
        {
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating booking");
            throw new RpcException(new Status(StatusCode.Internal, "Internal server error"));
        }
    }

    public override async Task GetBookings(GetBookingsRequest request,
        IServerStreamWriter<BookingResponse> responseStream, ServerCallContext context)
    {
        try
        {
            var bookings = await _bookingRepository.GetByCustomerIdAsync(request.CustomerId,
                request.Page, request.PageSize);

            foreach (var booking in bookings)
            {
                if (context.CancellationToken.IsCancellationRequested)
                    break;

                var response = new BookingResponse
                {
                    Id = booking.Id,
                    CustomerId = booking.CustomerId,
                    RoomId = booking.RoomId,
                    CheckIn = booking.CheckIn.ToString("O"),
                    CheckOut = booking.CheckOut.ToString("O"),
                    TotalAmount = (double)booking.TotalAmount,
                    Status = booking.Status,
                    CreatedAt = booking.CreatedAt.ToString("O")
                };

                await responseStream.WriteAsync(response);
                await Task.Delay(100, context.CancellationToken); // Simulate processing time
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error streaming bookings for customer {CustomerId}", request.CustomerId);
            throw new RpcException(new Status(StatusCode.Internal, "Internal server error"));
        }
    }

    public override async Task<BookingResponse> BookingStream(
        IAsyncStreamReader<CreateBookingRequest> requestStream,
        ServerCallContext context)
    {
        var bookingCount = 0;
        var totalAmount = 0.0;

        await foreach (var request in requestStream.ReadAllAsync())
        {
            try
            {
                var booking = await CreateBooking(request, context);
                bookingCount++;
                totalAmount += booking.TotalAmount;
            }
            catch (RpcException ex)
            {
                _logger.LogWarning("Failed to create booking in stream: {Error}", ex.Status.Detail);
            }
        }

        return new BookingResponse
        {
            Id = bookingCount,
            TotalAmount = totalAmount,
            Status = $"Processed {bookingCount} bookings"
        };
    }
}
```

**gRPC Client:**

```csharp
// Client/Program.cs
using Grpc.Net.Client;
using HotelManagement.Grpc;

// Create gRPC channel
using var channel = GrpcChannel.ForAddress("https://localhost:7001");
var client = new HotelService.HotelServiceClient(channel);

// Unary call
var roomRequest = new GetRoomRequest { RoomId = 1 };
var roomResponse = await client.GetRoomAsync(roomRequest);
Console.WriteLine($"Room: {roomResponse.Name} - ${roomResponse.PricePerNight}/night");

// Server streaming
var bookingsRequest = new GetBookingsRequest
{
    CustomerId = "CUST001",
    Page = 1,
    PageSize = 10
};

using var streamingCall = client.GetBookings(bookingsRequest);
await foreach (var booking in streamingCall.ResponseStream.ReadAllAsync())
{
    Console.WriteLine($"Booking {booking.Id}: {booking.Status}");
}

// Client streaming example would go here...
```

### Use Cases

- **High-Performance APIs**: Low-latency, high-throughput service communication
- **Microservices Communication**: Efficient inter-service communication
- **Real-time Systems**: Streaming data and bidirectional communication
- **Mobile Applications**: Efficient mobile-to-server communication
- **IoT Applications**: Device-to-cloud communication with binary protocols
- **Multi-language Environments**: Language-agnostic service interfaces

### When to Use vs When Not to Use

**Use gRPC when:**

- Need high-performance, low-latency communication
- Building microservices with type-safe contracts
- Require streaming capabilities
- Working in polyglot environments
- Need efficient binary serialization
- Building internal service APIs

**Consider alternatives when:**

- Building public web APIs (REST is more accessible)
- Working with web browsers (limited gRPC support)
- Need human-readable messages for debugging
- Working with legacy systems
- Team lacks Protocol Buffers expertise

### Market Alternatives & Pros/Cons

**Alternatives:**

- **REST APIs**: HTTP-based, human-readable, widely supported
- **Apache Thrift**: Cross-language RPC framework
- **Apache Avro**: Data serialization system
- **MessagePack**: Efficient binary serialization
- **JSON-RPC**: Lightweight remote procedure call protocol
- **WebSockets**: Real-time bidirectional communication

**Pros:**

- High performance and efficiency
- Strong typing with Protocol Buffers
- Built-in streaming support
- Language agnostic
- HTTP/2 benefits (multiplexing, compression)
- Excellent tooling and code generation

**Cons:**

- Limited browser support
- Learning curve for Protocol Buffers
- Binary format makes debugging harder
- Requires HTTP/2 infrastructure
- Less human-readable than JSON/XML

### Complete Runnable Sample

**Complete gRPC Solution:**

```xml
<!-- HotelManagement.Grpc.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Grpc.AspNetCore" Version="2.57.0" />
    <PackageReference Include="Grpc.AspNetCore.Server.Reflection" Version="2.57.0" />
  </ItemGroup>

  <ItemGroup>
    <Protobuf Include="Protos\hotel.proto" GrpcServices="Server" />
  </ItemGroup>
</Project>
```
