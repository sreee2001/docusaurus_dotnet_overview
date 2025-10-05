### MVC (Model-View-Controller)

#### Short Introduction

MVC is an architectural pattern that separates an application into three main components: Model (data), View (presentation), and Controller (logic).

#### Official Definition

Model-View-Controller (MVC) is an architectural pattern that separates an application into three main logical components: the model, the view, and the controller, each with distinct responsibilities.

#### Usage

```csharp
// Model
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string Description { get; set; } = string.Empty;
    public int CategoryId { get; set; }
    public Category? Category { get; set; }
}

public class ProductViewModel
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string FormattedPrice => Price.ToString("C");
    public string CategoryName { get; set; } = string.Empty;
}

// Controller
public class ProductsController : Controller
{
    private readonly IProductService _productService;

    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }

    public async Task<IActionResult> Index()
    {
        var products = await _productService.GetAllProductsAsync();
        var viewModels = products.Select(p => new ProductViewModel
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            CategoryName = p.Category?.Name ?? "Unknown"
        });
        return View(viewModels);
    }

    public async Task<IActionResult> Details(int id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        if (product == null)
            return NotFound();

        var viewModel = new ProductViewModel
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            CategoryName = product.Category?.Name ?? "Unknown"
        };

        return View(viewModel);
    }

    [HttpPost]
    public async Task<IActionResult> Create(CreateProductDto dto)
    {
        if (!ModelState.IsValid)
            return View(dto);

        await _productService.CreateProductAsync(dto);
        return RedirectToAction(nameof(Index));
    }
}
```

#### View (Razor)

```html
@model IEnumerable<ProductViewModel>
@{
    ViewData["Title"] = "Products";
}

<h1>Products</h1>

<div class="row">
    @foreach (var product in Model)
    {
        <div class="col-md-4 mb-3">
            <div class="card">
                <div class="card-body">
                    <h5 class="card-title">@product.Name</h5>
                    <p class="card-text">@product.FormattedPrice</p>
                    <p class="card-text"><small class="text-muted">@product.CategoryName</small></p>
                    <a href="@Url.Action("Details", new { id = product.Id })" class="btn btn-primary">View Details</a>
                </div>
            </div>
        </div>
    }
</div>
```

#### Use Cases

- Traditional web applications
- Server-rendered applications
- Applications with complex UI logic
- Multi-page applications

#### When to Use / When Not to Use

**Use MVC when:**

- Building traditional web applications
- Need server-side rendering
- SEO is important
- Team familiar with MVC pattern

**Consider alternatives when:**

- Building SPAs
- Need real-time updates
- API-only applications
- Simple static sites

---
