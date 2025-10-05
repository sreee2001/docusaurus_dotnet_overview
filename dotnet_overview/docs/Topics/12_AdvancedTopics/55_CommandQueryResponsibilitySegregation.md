## 55. CQRS (Command Query Responsibility Segregation)

### Short Introduction

CQRS is an architectural pattern that separates read and write operations by using different models to update information and to read information. This separation allows for optimized data models and can improve performance, scalability, and security.

### Official Definition

Command Query Responsibility Segregation (CQRS) is a pattern that segregates the operations that read data (Queries) from the operations that update data (Commands) by using separate interfaces. CQRS can also be used with Event Sourcing to provide a complete audit trail and enable temporal queries.

### Usage

CQRS is typically implemented using the MediatR library in .NET, which provides a mediator pattern implementation. Commands represent state-changing operations, while queries represent read operations.

```bash
# Install required packages
dotnet add package MediatR
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection
dotnet add package FluentValidation
dotnet add package FluentValidation.DependencyInjectionExtensions
```

### Use Cases

- Applications with complex business logic that benefits from separate read/write models
- Systems requiring different optimization strategies for reads vs writes
- Applications needing detailed audit trails and event history
- High-performance systems where read and write workloads have different scaling requirements
- Domain-driven design implementations with rich business models
- Systems integrating with event sourcing patterns

### When to Use vs When Not to Use

**Use CQRS when:**

- Your application has complex business logic with different read/write requirements
- You need to optimize read and write operations separately
- The system benefits from eventual consistency
- You're implementing event sourcing or need comprehensive audit trails
- Different teams work on read vs write functionality
- You need to scale read and write operations independently

**Don't use CQRS when:**

- Your application has simple CRUD operations
- The added complexity isn't justified by business requirements
- You need strong consistency for all operations
- Your team lacks experience with advanced architectural patterns
- The application has straightforward data access patterns

### Market Alternatives and Adoption

**Alternatives:**

- Traditional Repository Pattern with services
- Standard MVC/API controllers with direct database access
- Domain-Driven Design without CQRS
- Event-driven architecture without command/query separation

**Market Adoption:**

- Widely adopted in enterprise applications and microservices
- Popular in Domain-Driven Design implementations
- Supported by major frameworks (.NET, Java Spring, Node.js)
- Growing adoption in cloud-native applications

### Pros and Cons

**Pros:**

- Clear separation of concerns between reads and writes
- Optimized data models for different use cases
- Better scalability through independent scaling of read/write sides
- Enhanced security through separate permissions
- Improved performance through specialized optimization
- Better testability of business logic
- Supports complex business workflows

**Cons:**

- Increased architectural complexity
- Potential for eventual consistency issues
- More code to maintain and understand
- Learning curve for development teams
- Can lead to over-engineering simple scenarios
- Debugging can be more complex

### Sample Usage

```csharp
// Package references:
// MediatR v12.2.0
// MediatR.Extensions.Microsoft.DependencyInjection v11.1.0
// FluentValidation v11.8.0
// FluentValidation.DependencyInjectionExtensions v11.8.0

using MediatR;
using FluentValidation;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

// Domain Model
public class Hotel
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Address { get; set; } = string.Empty;
    public int TotalRooms { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}

// Commands (Write Side)
public record CreateHotelCommand(string Name, string Address, int TotalRooms) : IRequest<int>;

public record UpdateHotelCommand(int Id, string Name, string Address, int TotalRooms) : IRequest<bool>;

public record DeleteHotelCommand(int Id) : IRequest<bool>;

// Queries (Read Side)
public record GetHotelByIdQuery(int Id) : IRequest<HotelDto?>;

public record GetAllHotelsQuery(int Page, int PageSize) : IRequest<PaginatedResult<HotelDto>>;

public record SearchHotelsQuery(string SearchTerm, string? City) : IRequest<List<HotelDto>>;

// DTOs for Read Side
public record HotelDto(int Id, string Name, string Address, int TotalRooms, DateTime CreatedAt);

public record PaginatedResult<T>(List<T> Data, int TotalCount, int Page, int PageSize);

// Command Handlers
public class CreateHotelCommandHandler : IRequestHandler<CreateHotelCommand, int>
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<CreateHotelCommandHandler> _logger;

    public CreateHotelCommandHandler(ApplicationDbContext context, ILogger<CreateHotelCommandHandler> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task<int> Handle(CreateHotelCommand request, CancellationToken cancellationToken)
    {
        var hotel = new Hotel
        {
            Name = request.Name,
            Address = request.Address,
            TotalRooms = request.TotalRooms,
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };

        _context.Hotels.Add(hotel);
        await _context.SaveChangesAsync(cancellationToken);

        _logger.LogInformation("Created hotel {HotelName} with ID {HotelId}", hotel.Name, hotel.Id);
        return hotel.Id;
    }
}

public class UpdateHotelCommandHandler : IRequestHandler<UpdateHotelCommand, bool>
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<UpdateHotelCommandHandler> _logger;

    public UpdateHotelCommandHandler(ApplicationDbContext context, ILogger<UpdateHotelCommandHandler> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task<bool> Handle(UpdateHotelCommand request, CancellationToken cancellationToken)
    {
        var hotel = await _context.Hotels.FindAsync(new object[] { request.Id }, cancellationToken);
        if (hotel == null)
            return false;

        hotel.Name = request.Name;
        hotel.Address = request.Address;
        hotel.TotalRooms = request.TotalRooms;
        hotel.UpdatedAt = DateTime.UtcNow;

        await _context.SaveChangesAsync(cancellationToken);
        _logger.LogInformation("Updated hotel {HotelId}", request.Id);
        return true;
    }
}

// Query Handlers
public class GetHotelByIdQueryHandler : IRequestHandler<GetHotelByIdQuery, HotelDto?>
{
    private readonly ApplicationDbContext _context;

    public GetHotelByIdQueryHandler(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<HotelDto?> Handle(GetHotelByIdQuery request, CancellationToken cancellationToken)
    {
        var hotel = await _context.Hotels
            .AsNoTracking()
            .FirstOrDefaultAsync(h => h.Id == request.Id, cancellationToken);

        return hotel == null ? null : new HotelDto(hotel.Id, hotel.Name, hotel.Address, hotel.TotalRooms, hotel.CreatedAt);
    }
}

public class GetAllHotelsQueryHandler : IRequestHandler<GetAllHotelsQuery, PaginatedResult<HotelDto>>
{
    private readonly ApplicationDbContext _context;

    public GetAllHotelsQueryHandler(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<PaginatedResult<HotelDto>> Handle(GetAllHotelsQuery request, CancellationToken cancellationToken)
    {
        var query = _context.Hotels.AsNoTracking();

        var totalCount = await query.CountAsync(cancellationToken);

        var hotels = await query
            .OrderBy(h => h.Name)
            .Skip((request.Page - 1) * request.PageSize)
            .Take(request.PageSize)
            .Select(h => new HotelDto(h.Id, h.Name, h.Address, h.TotalRooms, h.CreatedAt))
            .ToListAsync(cancellationToken);

        return new PaginatedResult<HotelDto>(hotels, totalCount, request.Page, request.PageSize);
    }
}

// Validators
public class CreateHotelCommandValidator : AbstractValidator<CreateHotelCommand>
{
    public CreateHotelCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Hotel name is required")
            .MaximumLength(200).WithMessage("Hotel name cannot exceed 200 characters");

        RuleFor(x => x.Address)
            .NotEmpty().WithMessage("Address is required")
            .MaximumLength(500).WithMessage("Address cannot exceed 500 characters");

        RuleFor(x => x.TotalRooms)
            .GreaterThan(0).WithMessage("Total rooms must be greater than 0")
            .LessThanOrEqualTo(10000).WithMessage("Total rooms cannot exceed 10,000");
    }
}

// Behaviors (Cross-cutting concerns)
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        if (_validators.Any())
        {
            var context = new ValidationContext<TRequest>(request);
            var validationResults = await Task.WhenAll(_validators.Select(v => v.ValidateAsync(context, cancellationToken)));
            var failures = validationResults.SelectMany(r => r.Errors).Where(f => f != null).ToList();

            if (failures.Any())
                throw new ValidationException(failures);
        }

        return await next();
    }
}

public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("Handling {RequestName}", requestName);

        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            var response = await next();
            stopwatch.Stop();
            _logger.LogInformation("Handled {RequestName} in {ElapsedMs}ms", requestName, stopwatch.ElapsedMilliseconds);
            return response;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger.LogError(ex, "Error handling {RequestName} after {ElapsedMs}ms", requestName, stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}

// Controller
[ApiController]
[Route("api/[controller]")]
public class HotelsController : ControllerBase
{
    private readonly IMediator _mediator;

    public HotelsController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    public async Task<ActionResult<int>> CreateHotel([FromBody] CreateHotelCommand command)
    {
        try
        {
            var hotelId = await _mediator.Send(command);
            return CreatedAtAction(nameof(GetHotel), new { id = hotelId }, hotelId);
        }
        catch (ValidationException ex)
        {
            return BadRequest(ex.Errors.Select(e => e.ErrorMessage));
        }
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<HotelDto>> GetHotel(int id)
    {
        var hotel = await _mediator.Send(new GetHotelByIdQuery(id));
        return hotel == null ? NotFound() : Ok(hotel);
    }

    [HttpGet]
    public async Task<ActionResult<PaginatedResult<HotelDto>>> GetHotels([FromQuery] int page = 1, [FromQuery] int pageSize = 10)
    {
        var result = await _mediator.Send(new GetAllHotelsQuery(page, pageSize));
        return Ok(result);
    }

    [HttpPut("{id}")]
    public async Task<ActionResult> UpdateHotel(int id, [FromBody] UpdateHotelRequest request)
    {
        try
        {
            var command = new UpdateHotelCommand(id, request.Name, request.Address, request.TotalRooms);
            var success = await _mediator.Send(command);
            return success ? NoContent() : NotFound();
        }
        catch (ValidationException ex)
        {
            return BadRequest(ex.Errors.Select(e => e.ErrorMessage));
        }
    }

    [HttpDelete("{id}")]
    public async Task<ActionResult> DeleteHotel(int id)
    {
        var success = await _mediator.Send(new DeleteHotelCommand(id));
        return success ? NoContent() : NotFound();
    }
}

public record UpdateHotelRequest(string Name, string Address, int TotalRooms);

// DbContext
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }

    public DbSet<Hotel> Hotels { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Hotel>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Name).IsRequired().HasMaxLength(200);
            entity.Property(e => e.Address).IsRequired().HasMaxLength(500);
            entity.Property(e => e.TotalRooms).IsRequired();
            entity.Property(e => e.CreatedAt).IsRequired();
            entity.Property(e => e.UpdatedAt).IsRequired();
        });
    }
}

// Program.cs Configuration
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Add DbContext
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Add MediatR
builder.Services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);
});

// Add FluentValidation
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);

// Add behaviors
builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```
