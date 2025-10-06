---
slug: blazor_pages
title: Blazor
tags: [dotnet, core, blazor, pages, architecture, web]
---

# Blazor

## Short Introduction

Blazor is a framework for building interactive web UIs using C# instead of JavaScript, supporting both server-side and client-side execution models.

## Official Definition

Blazor is a free and open-source web framework that enables developers to create web apps using C# and HTML being developed by Microsoft.

## Blazor Server vs Blazor WebAssembly

- **Blazor Server**: Runs on the server, UI updates sent via SignalR
- **Blazor WebAssembly**: Runs in the browser using WebAssembly

## Usage

```csharp
// Components/ProductCard.razor
<div class="card" style="width: 18rem;">
    <div class="card-body">
        <h5 class="card-title">@Product.Name</h5>
        <p class="card-text">@Product.Description</p>
        <p class="card-text"><strong>@Product.Price.ToString("C")</strong></p>
        <button class="btn btn-primary" @onclick="AddToCart">Add to Cart</button>
    </div>
</div>

@code {
    [Parameter] public Product Product { get; set; } = new();
    [Parameter] public EventCallback<Product> OnAddToCart { get; set; }

    private async Task AddToCart()
    {
        await OnAddToCart.InvokeAsync(Product);
    }
}

// Pages/Products.razor
@page "/products"
@inject IProductService ProductService
@inject IJSRuntime JSRuntime

<h3>Products</h3>

@if (products == null)
{
    <p><em>Loading...</em></p>
}
else
{
    <div class="row">
        @foreach (var product in products)
        {
            <div class="col-md-4 mb-3">
                <ProductCard Product="product" OnAddToCart="HandleAddToCart" />
            </div>
        }
    </div>
}

@code {
    private List<Product>? products;

    protected override async Task OnInitializedAsync()
    {
        products = (await ProductService.GetAllProductsAsync()).ToList();
    }

    private async Task HandleAddToCart(Product product)
    {
        // Add to cart logic
        await JSRuntime.InvokeVoidAsync("alert", $"Added {product.Name} to cart!");
    }
}
```

## Interactive Components

```csharp
// Components/Counter.razor
<h3>Counter</h3>

<p>Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;

    [Parameter] public int IncrementBy { get; set; } = 1;

    private void IncrementCount()
    {
        currentCount += IncrementBy;
    }
}

// Usage
<Counter IncrementBy="5" />
```

## Use Cases

- Interactive web applications
- Real-time dashboards
- Progressive Web Apps (PWAs)
- Internal business applications
- Rapid application development

## When to Use / When Not to Use

### Use Blazor when:

- Team expertise in C#
- Building interactive UIs
- Want to avoid JavaScript
- Real-time applications
- .NET ecosystem integration

### Consider alternatives when:

- Need maximum performance
- Large existing JavaScript codebase
- Third-party JavaScript library requirements
- Public-facing websites (SEO concerns with WebAssembly)

## Market Alternatives

- React
- Angular
- Vue.js
- Svelte

## Pros:

- Single language (C#)
- Strong typing
- .NET ecosystem integration
- Shared code between client/server

## Cons:

- Limited browser support (WebAssembly)
- Larger payload size
- Less mature ecosystem
- SEO challenges with client-side rendering
