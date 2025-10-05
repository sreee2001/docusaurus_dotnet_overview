## 32. Azure SQL Database

### Short Introduction + Official Definition

Azure SQL Database is a fully managed Platform-as-a-Service (PaaS) database engine that provides SQL Server capabilities in the cloud with built-in intelligence, security, and high availability.

**Official Definition**: "Azure SQL Database is a fully managed relational database with auto-scale, integral intelligence, and robust security. It's the intelligent, scalable, cloud database service that provides the broadest SQL Server engine compatibility."

### Setup and Deployment Steps

**Azure CLI Setup**:

```bash
# Create SQL Server
az sql server create --name myserver --resource-group myResourceGroup --location eastus --admin-user myadmin --admin-password MyPassword123!

# Configure firewall
az sql server firewall-rule create --resource-group myResourceGroup --server myserver --name AllowAzureIps --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0

# Create database
az sql db create --resource-group myResourceGroup --server myserver --name mydatabase --service-objective S0
```

**Terraform Configuration**:

```hcl
resource "azurerm_mssql_server" "example" {
  name                         = "example-sqlserver"
  resource_group_name          = azurerm_resource_group.example.name
  location                     = azurerm_resource_group.example.location
  version                      = "12.0"
  administrator_login          = "4dm1n157r470r"
  administrator_login_password = "4-v3ry-53cr37-p455w0rd"
}

resource "azurerm_mssql_database" "example" {
  name           = "example-db"
  server_id      = azurerm_mssql_server.example.id
  collation      = "SQL_Latin1_General_CP1_CI_AS"
  license_type   = "LicenseIncluded"
  max_size_gb    = 2
  sku_name       = "S0"
}
```

### Typical Usage and Integration with .NET Apps

**Entity Framework Configuration**:

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=tcp:myserver.database.windows.net,1433;Database=mydatabase;User ID=myadmin;Password=MyPassword123!;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
  }
}

// Program.cs
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

// DbContext
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }

    public DbSet<Product> Products { get; set; }
    public DbSet<Customer> Customers { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Name).IsRequired().HasMaxLength(100);
            entity.Property(e => e.Price).HasColumnType("decimal(18,2)");
        });
    }
}
```

**Direct ADO.NET Usage**:

```csharp
using Microsoft.Data.SqlClient;

public class ProductService
{
    private readonly string _connectionString;

    public ProductService(IConfiguration configuration)
    {
        _connectionString = configuration.GetConnectionString("DefaultConnection");
    }

    public async Task<List<Product>> GetProductsAsync()
    {
        var products = new List<Product>();

        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();

        using var command = new SqlCommand("SELECT Id, Name, Price FROM Products", connection);
        using var reader = await command.ExecuteReaderAsync();

        while (await reader.ReadAsync())
        {
            products.Add(new Product
            {
                Id = reader.GetInt32("Id"),
                Name = reader.GetString("Name"),
                Price = reader.GetDecimal("Price")
            });
        }

        return products;
    }
}
```

### Use Cases

- Traditional relational database applications
- OLTP (Online Transaction Processing) workloads
- Data warehousing with columnstore indexes
- Applications requiring ACID transactions
- Migration from on-premises SQL Server
- Multi-tenant SaaS applications

### When to Use vs Alternatives

**Use Azure SQL Database when**:

- You need full SQL Server compatibility
- ACID transactions are critical
- Complex relational queries are common
- Existing SQL Server expertise exists
- Integration with Microsoft ecosystem is important

**Don't use when**:

- Document-based data models are more suitable
- Extreme horizontal scaling is required
- Cost optimization for simple data scenarios
- NoSQL flexibility is needed

**Alternatives**:

- **Azure**: Cosmos DB (NoSQL), PostgreSQL, MySQL
- **AWS**: RDS (SQL Server, PostgreSQL, MySQL), Aurora
- **GCP**: Cloud SQL, Cloud Spanner

### Market Pros/Cons and Cost Considerations

**Pros**:

- Fully managed with automatic backups and updates
- Built-in intelligent performance optimization
- Advanced security features (encryption, threat detection)
- High availability with 99.99% SLA
- Elastic scaling capabilities

**Cons**:

- Can be expensive for large databases
- Some SQL Server features not available
- Vendor lock-in to Azure
- Connection limits based on service tier

**Cost Considerations**:

- DTU model: S0 (10 DTU) ~$15/month, S1 (20 DTU) ~$30/month
- vCore model: 2 vCore General Purpose ~$350/month
- Serverless option available for intermittent workloads
- Storage charged separately (~$0.115/GB/month)
