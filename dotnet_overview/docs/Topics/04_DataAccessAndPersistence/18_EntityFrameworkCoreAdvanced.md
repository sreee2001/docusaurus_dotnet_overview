### Entity Framework Core Advanced

#### Short Introduction

Advanced Entity Framework Core topics including performance optimization, advanced queries, migrations, and database design patterns.

#### Advanced Querying

```csharp
// Complex queries with LINQ
public class ProductService : IProductService
{
    private readonly ApplicationDbContext _context;

    public ProductService(ApplicationDbContext context)
    {
        _context = context;
    }

    // Projection to avoid loading full entities
    public async Task<IEnumerable<ProductSummaryDto>> GetProductSummariesAsync()
    {
        return await _context.Products
            .Select(p => new ProductSummaryDto
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price,
                CategoryName = p.Category!.Name
            })
            .ToListAsync();
    }

    // Filtered and paged results
    public async Task<PagedResult<Product>> GetProductsPagedAsync(
        int page, int pageSize, string? searchTerm = null)
    {
        var query = _context.Products.Include(p => p.Category).AsQueryable();

        if (!string.IsNullOrEmpty(searchTerm))
        {
            query = query.Where(p => p.Name.Contains(searchTerm) ||
                                   p.Description.Contains(searchTerm));
        }

        var totalCount = await query.CountAsync();
        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();

        return new PagedResult<Product>
        {
            Items = items,
            TotalCount = totalCount,
            Page = page,
            PageSize = pageSize
        };
    }

    // Raw SQL queries
    public async Task<IEnumerable<Product>> GetExpensiveProductsAsync(decimal minPrice)
    {
        return await _context.Products
            .FromSqlRaw("SELECT * FROM Products WHERE Price >= {0}", minPrice)
            .ToListAsync();
    }

    // Stored procedure execution
    public async Task<IEnumerable<ProductSalesReport>> GetSalesReportAsync(
        DateTime startDate, DateTime endDate)
    {
        return await _context.Set<ProductSalesReport>()
            .FromSqlRaw("EXEC GetProductSalesReport @StartDate = {0}, @EndDate = {1}",
                       startDate, endDate)
            .ToListAsync();
    }
}
```

#### Advanced Configurations

```csharp
public class ApplicationDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Indexes
        modelBuilder.Entity<Product>()
            .HasIndex(p => p.Name)
            .IsUnique();

        modelBuilder.Entity<Product>()
            .HasIndex(p => new { p.CategoryId, p.Price });

        // Value conversions
        modelBuilder.Entity<Order>()
            .Property(e => e.Status)
            .HasConversion<string>();

        // Shadow properties
        modelBuilder.Entity<Product>()
            .Property<DateTime>("LastModified");

        // Global query filters
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => !p.IsDeleted);

        // Table splitting
        modelBuilder.Entity<Customer>()
            .ToTable("Customers");

        modelBuilder.Entity<CustomerDetails>()
            .ToTable("Customers");

        modelBuilder.Entity<Customer>()
            .HasOne(c => c.Details)
            .WithOne(d => d.Customer)
            .HasForeignKey<CustomerDetails>(d => d.CustomerId);

        // Owned types
        modelBuilder.Entity<Order>()
            .OwnsOne(o => o.ShippingAddress, sa =>
            {
                sa.Property(a => a.Street).HasColumnName("ShippingStreet");
                sa.Property(a => a.City).HasColumnName("ShippingCity");
                sa.Property(a => a.ZipCode).HasColumnName("ShippingZipCode");
            });
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Automatically set LastModified for all entities
        var entries = ChangeTracker.Entries()
            .Where(e => e.State == EntityState.Added || e.State == EntityState.Modified);

        foreach (var entry in entries)
        {
            entry.Property("LastModified").CurrentValue = DateTime.UtcNow;
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

#### Performance Optimization

```csharp
public class OptimizedProductService : IProductService
{
    private readonly ApplicationDbContext _context;

    public OptimizedProductService(ApplicationDbContext context)
    {
        _context = context;
    }

    // Split queries for multiple includes
    public async Task<IEnumerable<Order>> GetOrdersWithItemsAsync()
    {
        return await _context.Orders
            .AsSplitQuery()
            .Include(o => o.Items)
                .ThenInclude(i => i.Product)
            .Include(o => o.Customer)
            .ToListAsync();
    }

    // No-tracking queries for read-only scenarios
    public async Task<IEnumerable<Product>> GetProductsReadOnlyAsync()
    {
        return await _context.Products
            .AsNoTracking()
            .Include(p => p.Category)
            .ToListAsync();
    }

    // Compiled queries for frequently executed queries
    private static readonly Func<ApplicationDbContext, int, Task<Product?>> GetProductByIdCompiled =
        EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
            context.Products.FirstOrDefault(p => p.Id == id));

    public async Task<Product?> GetProductByIdAsync(int id)
    {
        return await GetProductByIdCompiled(_context, id);
    }

    // Bulk operations
    public async Task UpdateProductPricesAsync(decimal multiplier)
    {
        await _context.Database.ExecuteSqlRawAsync(
            "UPDATE Products SET Price = Price * {0}", multiplier);
    }
}
```

#### Migrations and Database Management

```csharp
// Custom migration
public partial class AddProductIndexes : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateIndex(
            name: "IX_Products_Name",
            table: "Products",
            column: "Name",
            unique: true);

        migrationBuilder.CreateIndex(
            name: "IX_Products_CategoryId_Price",
            table: "Products",
            columns: new[] { "CategoryId", "Price" });

        // Custom SQL
        migrationBuilder.Sql(@"
            CREATE VIEW ProductSummary AS
            SELECT p.Id, p.Name, p.Price, c.Name as CategoryName
            FROM Products p
            INNER JOIN Categories c ON p.CategoryId = c.Id
        ");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropIndex(
            name: "IX_Products_Name",
            table: "Products");

        migrationBuilder.DropIndex(
            name: "IX_Products_CategoryId_Price",
            table: "Products");

        migrationBuilder.Sql("DROP VIEW ProductSummary");
    }
}
```

---
