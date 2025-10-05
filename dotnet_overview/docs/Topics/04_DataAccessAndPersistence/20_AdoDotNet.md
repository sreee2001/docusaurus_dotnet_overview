## 5.3 ADO.NET

ADO.NET is the foundational data access technology in .NET that provides low-level access to data sources. It offers maximum control and performance but requires more code and careful resource management.

### Official Definition

ADO.NET (ActiveX Data Objects for .NET) is a set of classes that expose data access services for .NET Framework and .NET Core programmers. It provides a bridge between the front-end controls and the back-end database.

### Usage

ADO.NET is included in .NET by default. For SQL Server, install the Microsoft.Data.SqlClient package:

```bash
dotnet add package Microsoft.Data.SqlClient
```

Basic components include Connection, Command, DataReader, and DataAdapter classes.

### Use Cases

- Maximum performance requirements
- Fine-grained control over database operations
- Legacy system integration
- Complex data processing scenarios
- Bulk data operations
- Custom connection pooling requirements

### When to Use vs When Not to Use

**Use ADO.NET when:**

- Absolute maximum performance is required
- Working with legacy databases or systems
- Need precise control over connection management
- Implementing custom data access patterns
- Building data access frameworks
- Processing large datasets efficiently

**Don't use ADO.NET when:**

- Development speed is prioritized
- Team lacks database expertise
- Building standard CRUD applications
- Need object-relational mapping features
- Want to avoid boilerplate code

### Market Alternatives and Market Adoption

ADO.NET remains the foundation for all .NET data access technologies. While higher-level tools like Entity Framework and Dapper are more commonly used for application development, ADO.NET is still essential for framework development and high-performance scenarios.

### Pros and Cons

**Pros:**

- Maximum performance and control
- Direct access to all database features
- Minimal memory footprint
- No abstraction overhead
- Supports all data types
- Battle-tested and stable

**Cons:**

- Verbose and repetitive code
- Manual resource management required
- No object-relational mapping
- Prone to SQL injection if not careful
- Requires extensive database knowledge
- Time-consuming development

### Sample Usage

```csharp
// Package reference: <PackageReference Include="Microsoft.Data.SqlClient" Version="5.1.1" />
using Microsoft.Data.SqlClient;
using System.Data;

// Models
public class Room
{
    public int Id { get; set; }
    public string RoomNumber { get; set; } = string.Empty;
    public string RoomType { get; set; } = string.Empty;
    public decimal PricePerNight { get; set; }
    public bool IsAvailable { get; set; }
}

public class RoomRepository
{
    private readonly string _connectionString;

    public RoomRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    // Query with DataReader
    public async Task<Room?> GetRoomByIdAsync(int roomId)
    {
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        using var command = new SqlCommand(
            "SELECT Id, RoomNumber, RoomType, PricePerNight, IsAvailable FROM Rooms WHERE Id = @Id",
            connection);

        command.Parameters.Add(new SqlParameter("@Id", SqlDbType.Int) { Value = roomId });

        using var reader = await command.ExecuteReaderAsync();

        if (await reader.ReadAsync())
        {
            return new Room
            {
                Id = reader.GetInt32("Id"),
                RoomNumber = reader.GetString("RoomNumber"),
                RoomType = reader.GetString("RoomType"),
                PricePerNight = reader.GetDecimal("PricePerNight"),
                IsAvailable = reader.GetBoolean("IsAvailable")
            };
        }

        return null;
    }

    // Query multiple records
    public async Task<List<Room>> GetAvailableRoomsAsync()
    {
        var rooms = new List<Room>();

        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        using var command = new SqlCommand(
            "SELECT Id, RoomNumber, RoomType, PricePerNight, IsAvailable FROM Rooms WHERE IsAvailable = 1",
            connection);

        using var reader = await command.ExecuteReaderAsync();

        while (await reader.ReadAsync())
        {
            rooms.Add(new Room
            {
                Id = reader.GetInt32("Id"),
                RoomNumber = reader.GetString("RoomNumber"),
                RoomType = reader.GetString("RoomType"),
                PricePerNight = reader.GetDecimal("PricePerNight"),
                IsAvailable = reader.GetBoolean("IsAvailable")
            });
        }

        return rooms;
    }

    // Insert with transaction
    public async Task<int> CreateRoomAsync(Room room)
    {
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        using var transaction = connection.BeginTransaction();

        try
        {
            using var command = new SqlCommand(
                @"INSERT INTO Rooms (RoomNumber, RoomType, PricePerNight, IsAvailable)
                  VALUES (@RoomNumber, @RoomType, @PricePerNight, @IsAvailable);
                  SELECT CAST(SCOPE_IDENTITY() as int);",
                connection, transaction);

            command.Parameters.AddRange(new[]
            {
                new SqlParameter("@RoomNumber", SqlDbType.VarChar, 10) { Value = room.RoomNumber },
                new SqlParameter("@RoomType", SqlDbType.VarChar, 50) { Value = room.RoomType },
                new SqlParameter("@PricePerNight", SqlDbType.Decimal) { Value = room.PricePerNight },
                new SqlParameter("@IsAvailable", SqlDbType.Bit) { Value = room.IsAvailable }
            });

            var newId = (int)await command.ExecuteScalarAsync();

            await transaction.CommitAsync();
            return newId;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }

    // Bulk operations with DataAdapter
    public async Task<List<Room>> GetRoomsWithDataSetAsync()
    {
        using var connection = new SqlConnection(_connectionString);
        var adapter = new SqlDataAdapter("SELECT * FROM Rooms", connection);
        var dataSet = new DataSet();

        adapter.Fill(dataSet, "Rooms");

        var rooms = new List<Room>();
        foreach (DataRow row in dataSet.Tables["Rooms"].Rows)
        {
            rooms.Add(new Room
            {
                Id = (int)row["Id"],
                RoomNumber = row["RoomNumber"].ToString()!,
                RoomType = row["RoomType"].ToString()!,
                PricePerNight = (decimal)row["PricePerNight"],
                IsAvailable = (bool)row["IsAvailable"]
            });
        }

        return rooms;
    }

    // Stored procedure execution
    public async Task UpdateRoomAvailabilityAsync(int roomId, bool isAvailable)
    {
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        using var command = new SqlCommand("sp_UpdateRoomAvailability", connection)
        {
            CommandType = CommandType.StoredProcedure
        };

        command.Parameters.AddRange(new[]
        {
            new SqlParameter("@RoomId", SqlDbType.Int) { Value = roomId },
            new SqlParameter("@IsAvailable", SqlDbType.Bit) { Value = isAvailable }
        });

        await command.ExecuteNonQueryAsync();
    }
}

// Usage example
var connectionString = "Server=localhost;Database=HotelDB;Integrated Security=true;";
var roomRepository = new RoomRepository(connectionString);

// Create and retrieve a room
var newRoom = new Room
{
    RoomNumber = "101",
    RoomType = "Standard",
    PricePerNight = 99.99m,
    IsAvailable = true
};

var roomId = await roomRepository.CreateRoomAsync(newRoom);
var retrievedRoom = await roomRepository.GetRoomByIdAsync(roomId);
var availableRooms = await roomRepository.GetAvailableRoomsAsync();
```
