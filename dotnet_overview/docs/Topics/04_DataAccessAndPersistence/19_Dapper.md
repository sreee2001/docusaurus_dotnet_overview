### Dapper

#### Short Introduction

Dapper is a lightweight, fast, and efficient micro-ORM (Object-Relational Mapping) for .NET applications. Created by Stack Overflow team, it bridges the gap between raw ADO.NET and full-featured ORMs like Entity Framework Core.

Dapper is a simple object mapper for .NET that provides a fast, lightweight alternative to Entity Framework. It bridges the gap between raw ADO.NET and full ORMs by offering simple object mapping capabilities without the complexity of change tracking or lazy loading.

### Official Definition

Dapper is a simple object mapper for .NET that extends IDbConnection with high-performance extension methods for querying and manipulating databases. It's designed to be fast, lightweight, and close to the metal while still providing object mapping capabilities.

Dapper is a micro-ORM that extends `IDbConnection` with methods to execute SQL commands and map results to strongly-typed objects. It focuses on speed and simplicity rather than feature completeness.

### Usage

```bash
# Install Dapper
dotnet add package Dapper
dotnet add package Microsoft.Data.SqlClient
```

```csharp
// Program.cs
using Dapper;
using Microsoft.Data.SqlClient;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IDbConnection>(_ =>
    new SqlConnection(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddScoped<IUserRepository, UserRepository>();

var app = builder.Build();
```

Install via NuGet package manager:

```bash
dotnet add package Dapper
```

Basic setup requires a database connection and SQL commands:

```csharp
using System.Data.SqlClient;
using Dapper;

// Connection string configuration
var connectionString = "Server=localhost;Database=HotelDB;Integrated Security=true;";
using var connection = new SqlConnection(connectionString);
```

### Use Cases

- High-performance data access scenarios (database)
- Legacy database integration with complex stored procedures
- Microservices requiring lightweight data access and minimal dependencies
- Applications needing fine-grained SQL control
- Read-heavy operations with custom queries
- Data migration and ETL processes
- Bulk data operations

### When to Use vs When Not to Use

**Use Dapper when:**

- Performance is critical and you need speed close to raw ADO.NET
- Minimal ORM overhead required
- Working with existing databases with complex schemas
- Team has strong SQL expertise
- Need direct SQL control
- Need to execute stored procedures or complex queries that are hard to express in LINQ
- Building read-heavy applications

**Don't use Dapper when:**

- Rapid development is prioritized over performance
- Team lacks SQL expertise
- Need advanced ORM features (change tracking, lazy loading)
- Prefer strongly-typed LINQ queries
- Building CRUD-heavy applications
- Want automatic database schema migrations
- Complex domain models with relationships

### Market Alternatives and Market Adoption

**Market Position:** Widely adopted in performance-critical applications, especially at Stack Overflow, and popular in enterprise environments.
Dapper strikes a balance between EF Core's convenience and ADO.NET's performance.

**Alternatives:**

- Entity Framework Core (full-featured ORM)
- NHibernate (mature ORM)
- PetaPoco (micro-ORM)
- Massive (dynamic micro-ORM)
- Raw ADO.NET (maximum control)

### Pros and Cons

**Pros:**

- Excellent performance (close to raw ADO.NET)
- Simple learning curve
- Works with any database, Supports multiple database providers
- Great for existing SQL code, No configuration required
- Supports stored procedures well
- Flexible parameter binding
- Lightweight footprint

**Cons:**

- No change tracking
- Manual SQL writing required
- No lazy loading
- Limited LINQ support
- No automatic migrations
- Requires SQL expertise
- No automatic relationship handling
- Less productivity for complex domains

### Sample Usage 1

```csharp
// Package reference: <PackageReference Include="Dapper" Version="2.1.24" />
using System.Data.SqlClient;
using Dapper;

// Models
public class Guest
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
}

public class GuestService
{
    private readonly string _connectionString;

    public GuestService(string connectionString)
    {
        _connectionString = connectionString;
    }

    // Query single record
    public async Task<Guest?> GetGuestByIdAsync(int id)
    {
        using var connection = new SqlConnection(_connectionString);
        var sql = "SELECT Id, FirstName, LastName, Email, CreatedAt FROM Guests WHERE Id = @Id";
        return await connection.QuerySingleOrDefaultAsync<Guest>(sql, new { Id = id });
    }

    // Query multiple records
    public async Task<IEnumerable<Guest>> GetAllGuestsAsync()
    {
        using var connection = new SqlConnection(_connectionString);
        var sql = "SELECT Id, FirstName, LastName, Email, CreatedAt FROM Guests ORDER BY LastName";
        return await connection.QueryAsync<Guest>(sql);
    }

    // Insert record
    public async Task<int> CreateGuestAsync(Guest guest)
    {
        using var connection = new SqlConnection(_connectionString);
        var sql = @"INSERT INTO Guests (FirstName, LastName, Email, CreatedAt)
                   VALUES (@FirstName, @LastName, @Email, @CreatedAt);
                   SELECT CAST(SCOPE_IDENTITY() as int);";

        return await connection.QuerySingleAsync<int>(sql, guest);
    }

    // Update record
    public async Task<bool> UpdateGuestAsync(Guest guest)
    {
        using var connection = new SqlConnection(_connectionString);
        var sql = @"UPDATE Guests
                   SET FirstName = @FirstName, LastName = @LastName, Email = @Email
                   WHERE Id = @Id";

        var rowsAffected = await connection.ExecuteAsync(sql, guest);
        return rowsAffected > 0;
    }

    // Delete record
    public async Task<bool> DeleteGuestAsync(int id)
    {
        using var connection = new SqlConnection(_connectionString);
        var sql = "DELETE FROM Guests WHERE Id = @Id";
        var rowsAffected = await connection.ExecuteAsync(sql, new { Id = id });
        return rowsAffected > 0;
    }

    // Stored procedure example
    public async Task<IEnumerable<Guest>> GetGuestsByDateRangeAsync(DateTime startDate, DateTime endDate)
    {
        using var connection = new SqlConnection(_connectionString);
        return await connection.QueryAsync<Guest>(
            "sp_GetGuestsByDateRange",
            new { StartDate = startDate, EndDate = endDate },
            commandType: System.Data.CommandType.StoredProcedure);
    }
}

// Usage in Program.cs or controller
var connectionString = "Server=localhost;Database=HotelDB;Integrated Security=true;";
var guestService = new GuestService(connectionString);

// Create a new guest
var newGuest = new Guest
{
    FirstName = "John",
    LastName = "Doe",
    Email = "john.doe@email.com",
    CreatedAt = DateTime.UtcNow
};

var guestId = await guestService.CreateGuestAsync(newGuest);
var retrievedGuest = await guestService.GetGuestByIdAsync(guestId);
```

### Sample Usage 2

```csharp
// Models/User.cs
public class User
{
    public int Id { get; set; }
    public string Email { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsActive { get; set; }
}

// Repositories/IUserRepository.cs
public interface IUserRepository
{
    Task<IEnumerable<User>> GetAllUsersAsync();
    Task<User?> GetUserByIdAsync(int id);
    Task<User?> GetUserByEmailAsync(string email);
    Task<int> CreateUserAsync(User user);
    Task<bool> UpdateUserAsync(User user);
    Task<bool> DeleteUserAsync(int id);
    Task<IEnumerable<User>> SearchUsersAsync(string searchTerm);
}

// Repositories/UserRepository.cs
using Dapper;
using System.Data;

public class UserRepository : IUserRepository
{
    private readonly IDbConnection _connection;

    public UserRepository(IDbConnection connection)
    {
        _connection = connection;
    }

    public async Task<IEnumerable<User>> GetAllUsersAsync()
    {
        const string sql = @"
            SELECT Id, Email, FirstName, LastName, CreatedAt, IsActive
            FROM Users
            WHERE IsActive = 1
            ORDER BY LastName, FirstName";

        return await _connection.QueryAsync<User>(sql);
    }

    public async Task<User?> GetUserByIdAsync(int id)
    {
        const string sql = @"
            SELECT Id, Email, FirstName, LastName, CreatedAt, IsActive
            FROM Users
            WHERE Id = @Id";

        return await _connection.QueryFirstOrDefaultAsync<User>(sql, new { Id = id });
    }

    public async Task<User?> GetUserByEmailAsync(string email)
    {
        const string sql = @"
            SELECT Id, Email, FirstName, LastName, CreatedAt, IsActive
            FROM Users
            WHERE Email = @Email AND IsActive = 1";

        return await _connection.QueryFirstOrDefaultAsync<User>(sql, new { Email = email });
    }

    public async Task<int> CreateUserAsync(User user)
    {
        const string sql = @"
            INSERT INTO Users (Email, FirstName, LastName, CreatedAt, IsActive)
            VALUES (@Email, @FirstName, @LastName, @CreatedAt, @IsActive);
            SELECT CAST(SCOPE_IDENTITY() as int);";

        return await _connection.QuerySingleAsync<int>(sql, user);
    }

    public async Task<bool> UpdateUserAsync(User user)
    {
        const string sql = @"
            UPDATE Users
            SET Email = @Email,
                FirstName = @FirstName,
                LastName = @LastName,
                IsActive = @IsActive
            WHERE Id = @Id";

        var rowsAffected = await _connection.ExecuteAsync(sql, user);
        return rowsAffected > 0;
    }

    public async Task<bool> DeleteUserAsync(int id)
    {
        const string sql = "UPDATE Users SET IsActive = 0 WHERE Id = @Id";
        var rowsAffected = await _connection.ExecuteAsync(sql, new { Id = id });
        return rowsAffected > 0;
    }

    public async Task<IEnumerable<User>> SearchUsersAsync(string searchTerm)
    {
        const string sql = @"
            SELECT Id, Email, FirstName, LastName, CreatedAt, IsActive
            FROM Users
            WHERE IsActive = 1
                AND (FirstName LIKE @SearchTerm
                     OR LastName LIKE @SearchTerm
                     OR Email LIKE @SearchTerm)
            ORDER BY LastName, FirstName";

        return await _connection.QueryAsync<User>(sql,
            new { SearchTerm = $"%{searchTerm}%" });
    }
}

// Controllers/UsersController.cs
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserRepository _userRepository;

    public UsersController(IUserRepository userRepository)
    {
        _userRepository = userRepository;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<User>>> GetUsers()
    {
        var users = await _userRepository.GetAllUsersAsync();
        return Ok(users);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<User>> GetUser(int id)
    {
        var user = await _userRepository.GetUserByIdAsync(id);
        if (user == null)
            return NotFound();

        return Ok(user);
    }

    [HttpPost]
    public async Task<ActionResult<User>> CreateUser(User user)
    {
        user.CreatedAt = DateTime.UtcNow;
        user.IsActive = true;

        var id = await _userRepository.CreateUserAsync(user);
        user.Id = id;

        return CreatedAtAction(nameof(GetUser), new { id }, user);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateUser(int id, User user)
    {
        if (id != user.Id)
            return BadRequest();

        var success = await _userRepository.UpdateUserAsync(user);
        if (!success)
            return NotFound();

        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteUser(int id)
    {
        var success = await _userRepository.DeleteUserAsync(id);
        if (!success)
            return NotFound();

        return NoContent();
    }
}
```

---
