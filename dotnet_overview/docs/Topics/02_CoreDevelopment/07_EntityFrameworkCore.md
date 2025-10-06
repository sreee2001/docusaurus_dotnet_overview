---
slug: entity_framework_core
title: Entity Framework Core
tags: [dotnet, entity framework, core]
---

# Entity Framework Core

## Short Introduction

Entity Framework Core is a modern object-database mapper for .NET that supports LINQ queries, change tracking, updates, and schema migrations.

## Official Definition

Entity Framework Core (EF Core) is a lightweight, extensible, open source and cross-platform version of the popular Entity Framework data access technology.

## Usage

```csharp
// DbContext definition
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options) { }

    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Name).IsRequired().HasMaxLength(100);
            entity.Property(e => e.Price).HasColumnType("decimal(18,2)");
            entity.HasOne(e => e.Category)
                  .WithMany(c => c.Products)
                  .HasForeignKey(e => e.CategoryId);
        });
    }
}

// Service registration
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));
```

## Use Cases

- Data access layer
- Object-relational mapping
- Database migrations
- Complex queries with LINQ

## When to Use / When Not to Use

### Use EF Core when:

- Working with relational databases
- Need object-relational mapping
- Want type-safe queries
- Require change tracking

### Consider alternatives when:

- Simple data access requirements
- Performance is critical
- Working with non-relational data
- Team prefers SQL-first approach

## Market Alternatives

- Dapper (.NET)
- Hibernate (Java)
- Sequelize (Node.js)
- Django ORM (Python)
- ActiveRecord (Ruby)

## Sample Usage

```csharp
// Repository pattern with EF Core
public class ProductRepository : IProductRepository
{
    private readonly ApplicationDbContext _context;

    public ProductRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        return await _context.Products
            .Include(p => p.Category)
            .ToListAsync();
    }

    public async Task<Product?> GetByIdAsync(int id)
    {
        return await _context.Products
            .Include(p => p.Category)
            .FirstOrDefaultAsync(p => p.Id == id);
    }

    public async Task<Product> CreateAsync(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        return product;
    }

    public async Task UpdateAsync(Product product)
    {
        _context.Entry(product).State = EntityState.Modified;
        await _context.SaveChangesAsync();
    }

    public async Task DeleteAsync(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product != null)
        {
            _context.Products.Remove(product);
            await _context.SaveChangesAsync();
        }
    }
}
```
