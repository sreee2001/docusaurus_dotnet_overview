---
slug: unit_of_work
title: Unit of Work
tags:
  [dotnet, database, uow, unit_of_work, patterns, design_pattern, architecture]
---

# Unit of Work

The Unit of Work pattern maintains a list of objects affected by a business transaction and coordinates writing out changes and resolving concurrency problems. It ensures that multiple repository operations are treated as a single transaction.

## Official Definition

The Unit of Work pattern maintains a list of objects affected by a business transaction and coordinates writing out changes. It tracks changes to objects during a business transaction and coordinates writing out changes as a single operation.

## Usage

Implement by creating a class that manages multiple repositories and provides a single commit method. Often used with Entity Framework's DbContext, which already implements the Unit of Work pattern internally.

```csharp
public interface IUnitOfWork : IDisposable
{
    Task<int> SaveChangesAsync();
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}
```

## Use Cases

- Coordinating multiple repository operations in a single transaction
- Ensuring data consistency across multiple entities
- Implementing complex business operations that span multiple aggregates
- Managing database transactions at the application level
- Batch processing multiple changes efficiently
- Implementing saga patterns in microservices

## When to Use vs When Not to Use

### Use Unit of Work when:

- Operations span multiple repositories or entities
- Need explicit transaction control
- Implementing complex business workflows
- Coordinating changes across multiple bounded contexts
- Building systems with strict consistency requirements
- Need to optimize database round trips

### Don't use Unit of Work when:

- Using Entity Framework DbContext directly (it's already a UoW)
- Operations are simple and single-entity focused
- Over-engineering simple CRUD operations
- Team lacks understanding of transaction concepts
- Performance overhead is not justified

## Market Alternatives and Market Adoption

Entity Framework Core's DbContext implements the Unit of Work pattern internally. Alternatives include manual transaction management, distributed transaction coordinators, and message-based eventual consistency patterns. Many modern applications use DbContext directly rather than implementing additional UoW abstractions.

## Pros and Cons

### Pros:

- Ensures transactional consistency
- Optimizes database round trips
- Clear transaction boundaries
- Supports complex business operations
- Enables rollback of multiple changes
- Coordinates multiple repositories

### Cons:

- Adds complexity to simple operations
- Can create tight coupling between entities
- Memory overhead for tracking changes
- Potential for long-running transactions
- May duplicate EF Core functionality
- Requires careful lifecycle management

## Sample Usage

```csharp
// Package reference: Microsoft.EntityFrameworkCore
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage;

// Domain models
public class Guest
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
}

public class Booking
{
    public int Id { get; set; }
    public int GuestId { get; set; }
    public int RoomId { get; set; }
    public DateTime CheckInDate { get; set; }
    public DateTime CheckOutDate { get; set; }
    public decimal TotalAmount { get; set; }
    public Guest Guest { get; set; } = null!;
}

public class Payment
{
    public int Id { get; set; }
    public int BookingId { get; set; }
    public decimal Amount { get; set; }
    public DateTime PaymentDate { get; set; }
    public string PaymentMethod { get; set; } = string.Empty;
    public Booking Booking { get; set; } = null!;
}

// Repository interfaces
public interface IGuestRepository
{
    Task<Guest> AddAsync(Guest guest);
    Task<Guest?> GetByEmailAsync(string email);
}

public interface IBookingRepository
{
    Task<Booking> AddAsync(Booking booking);
    Task<Booking?> GetByIdAsync(int id);
}

public interface IPaymentRepository
{
    Task<Payment> AddAsync(Payment payment);
    Task<IEnumerable<Payment>> GetByBookingIdAsync(int bookingId);
}

// Unit of Work interface
public interface IUnitOfWork : IDisposable
{
    IGuestRepository Guests { get; }
    IBookingRepository Bookings { get; }
    IPaymentRepository Payments { get; }

    Task<int> SaveChangesAsync();
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}

// DbContext
public class HotelDbContext : DbContext
{
    public HotelDbContext(DbContextOptions<HotelDbContext> options) : base(options) { }

    public DbSet<Guest> Guests { get; set; }
    public DbSet<Booking> Bookings { get; set; }
    public DbSet<Payment> Payments { get; set; }
}

// Repository implementations
public class GuestRepository : IGuestRepository
{
    private readonly HotelDbContext _context;

    public GuestRepository(HotelDbContext context)
    {
        _context = context;
    }

    public async Task<Guest> AddAsync(Guest guest)
    {
        _context.Guests.Add(guest);
        return guest;
    }

    public async Task<Guest?> GetByEmailAsync(string email)
    {
        return await _context.Guests.FirstOrDefaultAsync(g => g.Email == email);
    }
}

public class BookingRepository : IBookingRepository
{
    private readonly HotelDbContext _context;

    public BookingRepository(HotelDbContext context)
    {
        _context = context;
    }

    public async Task<Booking> AddAsync(Booking booking)
    {
        _context.Bookings.Add(booking);
        return booking;
    }

    public async Task<Booking?> GetByIdAsync(int id)
    {
        return await _context.Bookings
            .Include(b => b.Guest)
            .FirstOrDefaultAsync(b => b.Id == id);
    }
}

public class PaymentRepository : IPaymentRepository
{
    private readonly HotelDbContext _context;

    public PaymentRepository(HotelDbContext context)
    {
        _context = context;
    }

    public async Task<Payment> AddAsync(Payment payment)
    {
        _context.Payments.Add(payment);
        return payment;
    }

    public async Task<IEnumerable<Payment>> GetByBookingIdAsync(int bookingId)
    {
        return await _context.Payments
            .Where(p => p.BookingId == bookingId)
            .ToListAsync();
    }
}

// Unit of Work implementation
public class UnitOfWork : IUnitOfWork
{
    private readonly HotelDbContext _context;
    private IDbContextTransaction? _transaction;

    public UnitOfWork(HotelDbContext context)
    {
        _context = context;
        Guests = new GuestRepository(_context);
        Bookings = new BookingRepository(_context);
        Payments = new PaymentRepository(_context);
    }

    public IGuestRepository Guests { get; }
    public IBookingRepository Bookings { get; }
    public IPaymentRepository Payments { get; }

    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }

    public async Task BeginTransactionAsync()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }

    public async Task CommitTransactionAsync()
    {
        if (_transaction != null)
        {
            await _transaction.CommitAsync();
            await _transaction.DisposeAsync();
            _transaction = null;
        }
    }

    public async Task RollbackTransactionAsync()
    {
        if (_transaction != null)
        {
            await _transaction.RollbackAsync();
            await _transaction.DisposeAsync();
            _transaction = null;
        }
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
    }
}

// Business service using Unit of Work
public class BookingService
{
    private readonly IUnitOfWork _unitOfWork;

    public BookingService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task<Booking> CreateBookingWithPaymentAsync(
        string guestEmail,
        string firstName,
        string lastName,
        int roomId,
        DateTime checkIn,
        DateTime checkOut,
        decimal totalAmount,
        decimal paymentAmount,
        string paymentMethod)
    {
        await _unitOfWork.BeginTransactionAsync();

        try
        {
            // Find or create guest
            var guest = await _unitOfWork.Guests.GetByEmailAsync(guestEmail);
            if (guest == null)
            {
                guest = new Guest
                {
                    FirstName = firstName,
                    LastName = lastName,
                    Email = guestEmail
                };
                await _unitOfWork.Guests.AddAsync(guest);
                await _unitOfWork.SaveChangesAsync(); // Save to get ID
            }

            // Create booking
            var booking = new Booking
            {
                GuestId = guest.Id,
                RoomId = roomId,
                CheckInDate = checkIn,
                CheckOutDate = checkOut,
                TotalAmount = totalAmount
            };

            await _unitOfWork.Bookings.AddAsync(booking);
            await _unitOfWork.SaveChangesAsync(); // Save to get booking ID

            // Create payment
            var payment = new Payment
            {
                BookingId = booking.Id,
                Amount = paymentAmount,
                PaymentDate = DateTime.UtcNow,
                PaymentMethod = paymentMethod
            };

            await _unitOfWork.Payments.AddAsync(payment);
            await _unitOfWork.SaveChangesAsync();

            await _unitOfWork.CommitTransactionAsync();
            return booking;
        }
        catch
        {
            await _unitOfWork.RollbackTransactionAsync();
            throw;
        }
    }
}

// Dependency injection setup (Program.cs)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<HotelDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddScoped<BookingService>();

var app = builder.Build();

// Usage example
public class BookingsController : ControllerBase
{
    private readonly BookingService _bookingService;

    public BookingsController(BookingService bookingService)
    {
        _bookingService = bookingService;
    }

    [HttpPost("with-payment")]
    public async Task<ActionResult<Booking>> CreateBookingWithPayment([FromBody] CreateBookingWithPaymentRequest request)
    {
        try
        {
            var booking = await _bookingService.CreateBookingWithPaymentAsync(
                request.GuestEmail,
                request.FirstName,
                request.LastName,
                request.RoomId,
                request.CheckIn,
                request.CheckOut,
                request.TotalAmount,
                request.PaymentAmount,
                request.PaymentMethod);

            return CreatedAtAction(nameof(GetBooking), new { id = booking.Id }, booking);
        }
        catch (Exception ex)
        {
            return BadRequest(ex.Message);
        }
    }
}

public class CreateBookingWithPaymentRequest
{
    public string GuestEmail { get; set; } = string.Empty;
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public int RoomId { get; set; }
    public DateTime CheckIn { get; set; }
    public DateTime CheckOut { get; set; }
    public decimal TotalAmount { get; set; }
    public decimal PaymentAmount { get; set; }
    public string PaymentMethod { get; set; } = string.Empty;
}
```
