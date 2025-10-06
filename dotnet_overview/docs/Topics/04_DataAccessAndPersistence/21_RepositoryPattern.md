---
slug: repository_pattern
title: Repository Pattern
tags:
  [dotnet, database, repository_pattern, patterns, design_pattern, architecture]
---

# Repository Pattern

The Repository Pattern encapsulates the logic needed to access data sources, centralizing common data access functionality and providing better maintainability and decoupling infrastructure or technology used to access databases from the domain model layer.

## Official Definition

The Repository Pattern is a design pattern that encapsulates the logic required to access data sources. It centralizes common data access functionality, promoting better maintainability and decoupling the infrastructure or technology used to access databases from the domain model layer.

## Usage

Implement by creating interfaces that define data access operations and concrete implementations that handle the actual data access logic. Often combined with dependency injection for better testability.

```csharp
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
}
```

## Use Cases

- Abstracting data access logic from business logic
- Supporting multiple data sources or switching between them
- Improving testability with mock repositories
- Centralizing query logic and data access patterns
- Implementing domain-driven design principles
- Creating clean architecture boundaries

## When to Use vs When Not to Use

### Use Repository Pattern when:

- Building applications with complex business logic
- Need to support multiple data sources
- Want to improve testability and maintainability
- Implementing clean architecture or DDD
- Team needs clear separation of concerns
- Planning to switch data access technologies

### Don't use Repository Pattern when:

- Building simple CRUD applications
- Using Entity Framework with direct DbContext access
- Team is small and simplicity is preferred
- Over-engineering concerns outweigh benefits
- Performance is critical and abstraction adds overhead

## Market Alternatives and Market Adoption

The Repository Pattern is widely adopted in enterprise .NET applications. Alternatives include direct ORM usage (Entity Framework DbContext), Active Record pattern, and Data Access Objects (DAO). Many modern frameworks like Entity Framework already implement repository-like patterns internally.

## Pros and Cons

### Pros:

- Clear separation of concerns
- Improved testability
- Centralized query logic
- Easy to mock for unit tests
- Supports multiple data sources
- Promotes clean architecture

### Cons:

- Can add unnecessary complexity
- Additional abstraction layer
- Potential performance overhead
- May duplicate ORM functionality
- Can become over-engineered
- Learning curve for developers

## Sample Usage

```csharp
// Package references might include Entity Framework or Dapper
using Microsoft.EntityFrameworkCore;

// Domain model
public class Booking
{
    public int Id { get; set; }
    public int GuestId { get; set; }
    public int RoomId { get; set; }
    public DateTime CheckInDate { get; set; }
    public DateTime CheckOutDate { get; set; }
    public decimal TotalAmount { get; set; }
    public BookingStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
}

public enum BookingStatus
{
    Pending,
    Confirmed,
    CheckedIn,
    CheckedOut,
    Cancelled
}

// Generic repository interface
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
    Task<bool> ExistsAsync(int id);
}

// Specific repository interface
public interface IBookingRepository : IRepository<Booking>
{
    Task<IEnumerable<Booking>> GetBookingsByGuestIdAsync(int guestId);
    Task<IEnumerable<Booking>> GetBookingsByDateRangeAsync(DateTime startDate, DateTime endDate);
    Task<IEnumerable<Booking>> GetActiveBookingsAsync();
    Task<bool> IsRoomAvailableAsync(int roomId, DateTime checkIn, DateTime checkOut);
}

// Entity Framework implementation
public class BookingRepository : IBookingRepository
{
    private readonly HotelDbContext _context;

    public BookingRepository(HotelDbContext context)
    {
        _context = context;
    }

    public async Task<Booking?> GetByIdAsync(int id)
    {
        return await _context.Bookings.FindAsync(id);
    }

    public async Task<IEnumerable<Booking>> GetAllAsync()
    {
        return await _context.Bookings.ToListAsync();
    }

    public async Task<Booking> AddAsync(Booking booking)
    {
        booking.CreatedAt = DateTime.UtcNow;
        _context.Bookings.Add(booking);
        await _context.SaveChangesAsync();
        return booking;
    }

    public async Task UpdateAsync(Booking booking)
    {
        _context.Entry(booking).State = EntityState.Modified;
        await _context.SaveChangesAsync();
    }

    public async Task DeleteAsync(int id)
    {
        var booking = await _context.Bookings.FindAsync(id);
        if (booking != null)
        {
            _context.Bookings.Remove(booking);
            await _context.SaveChangesAsync();
        }
    }

    public async Task<bool> ExistsAsync(int id)
    {
        return await _context.Bookings.AnyAsync(b => b.Id == id);
    }

    // Specific booking methods
    public async Task<IEnumerable<Booking>> GetBookingsByGuestIdAsync(int guestId)
    {
        return await _context.Bookings
            .Where(b => b.GuestId == guestId)
            .OrderByDescending(b => b.CreatedAt)
            .ToListAsync();
    }

    public async Task<IEnumerable<Booking>> GetBookingsByDateRangeAsync(DateTime startDate, DateTime endDate)
    {
        return await _context.Bookings
            .Where(b => b.CheckInDate >= startDate && b.CheckOutDate <= endDate)
            .ToListAsync();
    }

    public async Task<IEnumerable<Booking>> GetActiveBookingsAsync()
    {
        var today = DateTime.Today;
        return await _context.Bookings
            .Where(b => b.Status == BookingStatus.Confirmed || b.Status == BookingStatus.CheckedIn)
            .Where(b => b.CheckOutDate >= today)
            .ToListAsync();
    }

    public async Task<bool> IsRoomAvailableAsync(int roomId, DateTime checkIn, DateTime checkOut)
    {
        return !await _context.Bookings
            .AnyAsync(b => b.RoomId == roomId &&
                          b.Status != BookingStatus.Cancelled &&
                          ((checkIn >= b.CheckInDate && checkIn < b.CheckOutDate) ||
                           (checkOut > b.CheckInDate && checkOut <= b.CheckOutDate) ||
                           (checkIn <= b.CheckInDate && checkOut >= b.CheckOutDate)));
    }
}

// Service layer using repository
public class BookingService
{
    private readonly IBookingRepository _bookingRepository;

    public BookingService(IBookingRepository bookingRepository)
    {
        _bookingRepository = bookingRepository;
    }

    public async Task<Booking> CreateBookingAsync(int guestId, int roomId, DateTime checkIn, DateTime checkOut, decimal totalAmount)
    {
        // Business logic validation
        if (checkIn >= checkOut)
            throw new ArgumentException("Check-in date must be before check-out date");

        if (!await _bookingRepository.IsRoomAvailableAsync(roomId, checkIn, checkOut))
            throw new InvalidOperationException("Room is not available for the selected dates");

        var booking = new Booking
        {
            GuestId = guestId,
            RoomId = roomId,
            CheckInDate = checkIn,
            CheckOutDate = checkOut,
            TotalAmount = totalAmount,
            Status = BookingStatus.Pending
        };

        return await _bookingRepository.AddAsync(booking);
    }

    public async Task ConfirmBookingAsync(int bookingId)
    {
        var booking = await _bookingRepository.GetByIdAsync(bookingId);
        if (booking == null)
            throw new ArgumentException("Booking not found");

        booking.Status = BookingStatus.Confirmed;
        await _bookingRepository.UpdateAsync(booking);
    }
}

// Dependency injection setup (in Program.cs)
public class HotelDbContext : DbContext
{
    public HotelDbContext(DbContextOptions<HotelDbContext> options) : base(options) { }

    public DbSet<Booking> Bookings { get; set; }
}

// Registration in Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<HotelDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddScoped<IBookingRepository, BookingRepository>();
builder.Services.AddScoped<BookingService>();

var app = builder.Build();

// Usage in controller
[ApiController]
[Route("api/[controller]")]
public class BookingsController : ControllerBase
{
    private readonly BookingService _bookingService;

    public BookingsController(BookingService bookingService)
    {
        _bookingService = bookingService;
    }

    [HttpPost]
    public async Task<ActionResult<Booking>> CreateBooking([FromBody] CreateBookingRequest request)
    {
        try
        {
            var booking = await _bookingService.CreateBookingAsync(
                request.GuestId,
                request.RoomId,
                request.CheckIn,
                request.CheckOut,
                request.TotalAmount);

            return CreatedAtAction(nameof(GetBooking), new { id = booking.Id }, booking);
        }
        catch (Exception ex)
        {
            return BadRequest(ex.Message);
        }
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Booking>> GetBooking(int id)
    {
        var booking = await _bookingService._bookingRepository.GetByIdAsync(id);
        return booking == null ? NotFound() : Ok(booking);
    }
}

public class CreateBookingRequest
{
    public int GuestId { get; set; }
    public int RoomId { get; set; }
    public DateTime CheckIn { get; set; }
    public DateTime CheckOut { get; set; }
    public decimal TotalAmount { get; set; }
}
```
