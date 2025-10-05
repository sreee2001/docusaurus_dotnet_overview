### ASP.NET Core

#### Short Introduction

ASP.NET Core is a cross-platform, high-performance framework for building modern, cloud-based, Internet-connected applications.

#### Official Definition

ASP.NET Core is a free, open-source, and cross-platform framework for building modern cloud-based web applications on Windows, macOS, and Linux.

#### Usage

```csharp
// Program.cs - ASP.NET Core 8
var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllersWithViews();
builder.Services.AddRazorPages();

var app = builder.Build();

// Configure the HTTP request pipeline
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.MapRazorPages();
app.Run();
```

#### Use Cases

- Web applications
- RESTful APIs
- Real-time applications
- Microservices
- Backend services

#### When to Use / When Not to Use

**Use ASP.NET Core when:**

- Building web applications or APIs
- Need high performance
- Require cross-platform deployment
- Want modern development patterns

**Consider alternatives when:**

- Simple static websites
- Specific language requirements
- Team expertise in other frameworks

#### Market Alternatives

- Spring Boot (Java)
- Express.js (Node.js)
- Django/Flask (Python)
- Ruby on Rails
- Laravel (PHP)

#### Sample Usage

```csharp
// Controller example
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;

    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        var products = await _productService.GetAllProductsAsync();
        return Ok(products);
    }

    [HttpPost]
    public async Task<ActionResult<Product>> CreateProduct(CreateProductDto dto)
    {
        var product = await _productService.CreateProductAsync(dto);
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }
}
```

---
