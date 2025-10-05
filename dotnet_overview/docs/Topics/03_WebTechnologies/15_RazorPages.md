### Razor Pages

#### Short Introduction

Razor Pages is a page-focused framework that makes coding page-focused scenarios easier and more productive than using controllers and views.

#### Official Definition

Razor Pages is a page-based programming model that makes building web UI easier and more productive. It's built on top of ASP.NET Core MVC but provides a simpler, more focused approach to page-based scenarios.

#### Usage

```csharp
// Pages/Products/Index.cshtml.cs
public class IndexModel : PageModel
{
    private readonly IProductService _productService;

    public IndexModel(IProductService productService)
    {
        _productService = productService;
    }

    public IList<Product> Products { get; set; } = new List<Product>();

    public async Task OnGetAsync()
    {
        Products = (await _productService.GetAllProductsAsync()).ToList();
    }
}

// Pages/Products/Create.cshtml.cs
public class CreateModel : PageModel
{
    private readonly IProductService _productService;

    public CreateModel(IProductService productService)
    {
        _productService = productService;
    }

    [BindProperty]
    public CreateProductDto Product { get; set; } = new();

    public IActionResult OnGet()
    {
        return Page();
    }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid)
        {
            return Page();
        }

        await _productService.CreateProductAsync(Product);
        return RedirectToPage("./Index");
    }
}
```

#### Razor Page View

```html
@page @model IndexModel @{ ViewData["Title"] = "Products"; }

<h1>Products</h1>

<p>
  <a asp-page="Create">Create New</a>
</p>

<table class="table">
  <thead>
    <tr>
      <th>Name</th>
      <th>Price</th>
      <th>Category</th>
      <th>Actions</th>
    </tr>
  </thead>
  <tbody>
    @foreach (var item in Model.Products) {
    <tr>
      <td>@item.Name</td>
      <td>@item.Price.ToString("C")</td>
      <td>@item.Category?.Name</td>
      <td>
        <a asp-page="./Edit" asp-route-id="@item.Id">Edit</a> |
        <a asp-page="./Details" asp-route-id="@item.Id">Details</a> |
        <a asp-page="./Delete" asp-route-id="@item.Id">Delete</a>
      </td>
    </tr>
    }
  </tbody>
</table>
```

#### Use Cases

- Simple web applications
- Admin panels
- Content management systems
- Form-heavy applications
- Rapid prototyping

#### When to Use / When Not to Use

**Use Razor Pages when:**

- Building page-focused applications
- Simple CRUD operations
- Prototyping
- Team prefers page-based model

**Use MVC when:**

- Complex routing requirements
- Heavy use of filters
- Complex view logic
- RESTful APIs

---
