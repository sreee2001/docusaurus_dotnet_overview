## 26. Caching

### Short Introduction and Official Definition

Caching is a performance optimization technique that stores frequently accessed data in high-speed storage layers to reduce latency and improve application responsiveness. In .NET Core, caching operates at multiple levels: in-memory for single-instance scenarios, distributed for multi-instance deployments, and response caching for HTTP responses.

**Official Definition**: According to Microsoft documentation, caching in ASP.NET Core provides mechanisms to store data temporarily in memory or external stores to avoid expensive operations like database queries or complex computations.

### Setup/Usage

**In-Memory Caching Setup:**

```csharp
// Program.cs (.NET 8)
var builder = WebApplication.CreateBuilder(args);

// Add memory cache
builder.Services.AddMemoryCache();

var app = builder.Build();
```

**Distributed Caching with Redis:**

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
});

// Or with connection string from appsettings
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
});
```

**Response Caching:**

```csharp
// Program.cs
builder.Services.AddResponseCaching();

var app = builder.Build();
app.UseResponseCaching();
```

### Use Cases

- **In-Memory Caching**: Single-server applications, session data, frequently accessed reference data
- **Distributed Caching**: Multi-server deployments, session state sharing, cache invalidation across instances
- **Response Caching**: Static or semi-static HTTP responses, API responses with predictable expiration
- **Database Query Results**: Expensive database operations, reporting data, lookup tables
- **Computed Values**: Complex calculations, image processing results, aggregated metrics
- **External API Responses**: Third-party service calls, weather data, exchange rates

### When to Use vs When Not to Use

**When to Use:**

- Data is expensive to generate or retrieve
- Data doesn't change frequently
- Application serves multiple concurrent users
- Database or external service latency is high
- Memory is available and cost-effective

**When Not to Use:**

- Data changes very frequently (real-time data)
- Memory constraints are severe
- Cache invalidation logic becomes too complex
- Data consistency is critical and cache adds risk
- Simple applications with minimal performance requirements

### Alternatives and Trade-offs

**Alternatives:**

- Database query optimization and indexing
- CDN for static content
- Application-level optimization
- Horizontal scaling instead of caching

**Trade-offs:**

- Memory usage vs performance gain
- Data freshness vs speed
- Implementation complexity vs benefits
- Cost of cache infrastructure vs performance improvement

### Sample Code and Commands

**In-Memory Cache Example:**

```csharp
// Service using IMemoryCache
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repository;

    public ProductService(IMemoryCache cache, IProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public async Task<Product> GetProductAsync(int id)
    {
        string cacheKey = $"product_{id}";

        if (_cache.TryGetValue(cacheKey, out Product cachedProduct))
        {
            return cachedProduct;
        }

        var product = await _repository.GetByIdAsync(id);

        var cacheOptions = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
            SlidingExpiration = TimeSpan.FromMinutes(5),
            Priority = CacheItemPriority.Normal
        };

        _cache.Set(cacheKey, product, cacheOptions);
        return product;
    }
}
```

**Distributed Cache Example:**

```csharp
// Service using IDistributedCache
public class UserService
{
    private readonly IDistributedCache _cache;
    private readonly IUserRepository _repository;

    public UserService(IDistributedCache cache, IUserRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public async Task<User> GetUserAsync(int userId)
    {
        string cacheKey = $"user_{userId}";

        var cachedUser = await _cache.GetStringAsync(cacheKey);
        if (cachedUser != null)
        {
            return JsonSerializer.Deserialize<User>(cachedUser);
        }

        var user = await _repository.GetByIdAsync(userId);

        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1),
            SlidingExpiration = TimeSpan.FromMinutes(20)
        };

        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(user), options);
        return user;
    }
}
```

**Response Caching Example:**

```csharp
// Controller with response caching
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [ResponseCache(Duration = 300, Location = ResponseCacheLocation.Any)]
    public async Task<IActionResult> GetProducts()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }

    [HttpGet("{id}")]
    [ResponseCache(Duration = 60, VaryByQueryKeys = new[] { "id" })]
    public async Task<IActionResult> GetProduct(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        return Ok(product);
    }
}
```

**Redis Docker Command:**

```bash
# Start Redis container
docker run --name redis-cache -p 6379:6379 -d redis:latest

# Connect to Redis CLI
docker exec -it redis-cache redis-cli

# Test connection
PING
# Should return PONG
```
