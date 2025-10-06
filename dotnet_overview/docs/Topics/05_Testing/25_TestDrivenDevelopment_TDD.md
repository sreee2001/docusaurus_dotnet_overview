---
slug: tdd
title: Test Driven Development
tags: [dotnet, testing, tdd, quality, pattern]
---

# Test-Driven Development (TDD)

## Short Introduction

Test-Driven Development (TDD) is a software development methodology where tests are written before the actual implementation code. The process follows a Red-Green-Refactor cycle: write a failing test (Red), implement the minimum code to pass (Green), then refactor for quality while keeping tests passing.

## Official Definition

TDD is a software development process that relies on the repetition of a very short development cycle: requirements are turned into very specific test cases, then the code is improved so that the tests pass. This is opposed to software development that allows code to be added that is not proven to meet requirements.

## TDD Principles and Process

### The Red-Green-Refactor Cycle:

1. **Red**: Write a failing test that defines desired functionality
2. **Green**: Write minimal code to make the test pass
3. **Refactor**: Improve code quality while keeping tests green

### Core Principles:

- Write only enough test to fail
- Write only enough code to pass the test
- Refactor with confidence knowing tests protect against regressions
- Tests serve as living documentation
- Design emerges from test requirements

## Use Cases

- **Complex Business Logic**: Ensures business rules are correctly implemented
- **API Development**: Validates endpoints behave as expected
- **Legacy Code Refactoring**: Provides safety net when changing existing code
- **Team Collaboration**: Tests serve as living documentation
- **Regression Prevention**: Catches bugs early in development cycle
- **Design Improvement**: Forces consideration of code design and interfaces

## When to Use vs When Not to Use

### Use TDD when:

- Working on complex business logic
- Building critical system components
- Working in teams where code quality is paramount
- Refactoring legacy systems
- Building long-term maintainable software

### Consider alternatives when:

- Prototyping or proof-of-concept work
- Simple CRUD operations with minimal logic
- Working with rapidly changing requirements
- Time constraints are extremely tight
- Team lacks TDD experience and training time

## Market Alternatives & Pros/Cons

### Alternatives:

- **Behavior-Driven Development (BDD)**: SpecFlow, Cucumber
- **Test-After Development**: Traditional approach
- **Property-Based Testing**: FsCheck
- **Mutation Testing**: Stryker.NET

### Pros:

- Higher code quality and fewer bugs
- Better code design through testability focus
- Living documentation through tests
- Confidence in refactoring
- Faster debugging and development cycles

### Cons:

- Initial learning curve and slower start
- More code to write and maintain
- Can lead to over-testing trivial code
- May not suit all development contexts
- Requires discipline and team buy-in

## Benefits of TDD in .NET Development

### Code Quality Benefits:

- Higher test coverage by design
- Better code design and architecture
- Reduced debugging time
- Fewer production defects
- Self-documenting code through tests

### Development Process Benefits:

- Faster feedback loop
- Confidence in refactoring
- Clear requirements understanding
- Incremental development approach
- Better team collaboration

### Long-term Benefits:

- Maintainable codebase
- Easier feature additions
- Reduced technical debt
- Better system design
- Improved developer productivity

## TDD Best Practices for .NET

1. **Start Small**: Begin with simple, focused tests
2. **Test Behavior, Not Implementation**: Focus on what the code should do
3. **Use Descriptive Test Names**: Tests should clearly communicate intent
4. **Keep Tests Independent**: Each test should run in isolation
5. **Follow SOLID Principles**: TDD naturally leads to better design
6. **Mock External Dependencies**: Use frameworks like Moq for isolation
7. **Refactor Regularly**: Improve code quality continuously
8. **Write Tests First**: Resist the urge to write code before tests

## Common TDD Challenges and Solutions

**Challenge**: Slow test execution
**Solution**: Use unit tests for TDD, integration tests for verification

**Challenge**: Over-testing or testing implementation details
**Solution**: Focus on public interfaces and behavior

**Challenge**: Difficulty writing tests for complex scenarios
**Solution**: Break down complex problems into smaller, testable units

**Challenge**: Team resistance to TDD
**Solution**: Start with critical components, demonstrate value, provide training

**Challenge**: Legacy code without tests
**Solution**: Use characterization tests, gradually introduce TDD for new features

TDD is particularly effective in .NET development when combined with:

- Dependency injection for testable designs
- SOLID principles for maintainable code
- Clean architecture patterns
- Continuous integration for automated testing
- Code review practices that emphasize test quality

The investment in TDD pays dividends through reduced bugs, improved design, and increased developer confidence in making changes to the codebase.

## Setup/Usage with .NET 8+ Code

### Installation:

```bash
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package Moq
dotnet add package FluentAssertions
```

### Basic TDD Workflow:

```csharp
// 1. RED: Write failing test first
[Fact]
public void CalculateTotal_ShouldReturnCorrectSum_WhenValidItemsProvided()
{
    // Arrange
    var calculator = new OrderCalculator();
    var items = new List<OrderItem>
    {
        new() { Price = 10.00m, Quantity = 2 },
        new() { Price = 5.50m, Quantity = 1 }
    };

    // Act
    var result = calculator.CalculateTotal(items);

    // Assert
    result.Should().Be(25.50m);
}

// 2. GREEN: Implement minimum code to pass
public class OrderCalculator
{
    public decimal CalculateTotal(List<OrderItem> items)
    {
        return items.Sum(i => i.Price * i.Quantity);
    }
}

// 3. REFACTOR: Improve code quality
public class OrderCalculator
{
    public decimal CalculateTotal(IEnumerable<OrderItem> items)
    {
        if (items == null) throw new ArgumentNullException(nameof(items));

        return items
            .Where(item => item != null)
            .Sum(item => item.Price * item.Quantity);
    }
}
```

## TDD Practices for .NET

```csharp
// Example: TDD for Hotel Room Availability Service

// Step 1: RED - Write failing test
[Test]
public void IsRoomAvailable_WhenRoomExists_ReturnsTrue()
{
    // Arrange
    var roomService = new RoomService();

    // Act
    var result = roomService.IsRoomAvailable(101);

    // Assert
    Assert.That(result, Is.True);
}

// This test fails because RoomService doesn't exist yet

// Step 2: GREEN - Write minimal code to pass
public class RoomService
{
    public bool IsRoomAvailable(int roomId)
    {
        return true; // Simplest implementation to pass the test
    }
}

// Step 3: REFACTOR - Add more tests and improve implementation
[Test]
public void IsRoomAvailable_WhenRoomDoesNotExist_ReturnsFalse()
{
    // Arrange
    var roomService = new RoomService();

    // Act
    var result = roomService.IsRoomAvailable(999);

    // Assert
    Assert.That(result, Is.False);
}

// Now we need to improve the implementation
public class RoomService
{
    private readonly List<int> _availableRooms = new() { 101, 102, 103 };

    public bool IsRoomAvailable(int roomId)
    {
        return _availableRooms.Contains(roomId);
    }
}

// Continue with more tests and refinements
[Test]
public void IsRoomAvailable_WhenRoomIsBooked_ReturnsFalse()
{
    // Arrange
    var roomService = new RoomService();
    roomService.BookRoom(101);

    // Act
    var result = roomService.IsRoomAvailable(101);

    // Assert
    Assert.That(result, Is.False);
}
```

```csharp
// Models/OrderItem.cs
public class OrderItem
{
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Quantity { get; set; }
    public decimal Discount { get; set; }
}

// Services/IOrderService.cs
public interface IOrderService
{
    Task<Order> CreateOrderAsync(List<OrderItem> items, string customerId);
    decimal CalculateDiscount(decimal subtotal, string customerTier);
}

// Tests/OrderServiceTests.cs
using Xunit;
using Moq;
using FluentAssertions;

public class OrderServiceTests
{
    private readonly Mock<ICustomerRepository> _customerRepositoryMock;
    private readonly Mock<IOrderRepository> _orderRepositoryMock;
    private readonly OrderService _orderService;

    public OrderServiceTests()
    {
        _customerRepositoryMock = new Mock<ICustomerRepository>();
        _orderRepositoryMock = new Mock<IOrderRepository>();
        _orderService = new OrderService(_customerRepositoryMock.Object, _orderRepositoryMock.Object);
    }

    [Fact]
    public async Task CreateOrderAsync_ShouldCreateOrder_WhenValidItemsProvided()
    {
        // Arrange
        var customerId = "CUST001";
        var items = new List<OrderItem>
        {
            new() { Name = "Product A", Price = 100m, Quantity = 2, Discount = 0 },
            new() { Name = "Product B", Price = 50m, Quantity = 1, Discount = 10m }
        };

        _customerRepositoryMock
            .Setup(x => x.GetCustomerTierAsync(customerId))
            .ReturnsAsync("Gold");

        _orderRepositoryMock
            .Setup(x => x.SaveOrderAsync(It.IsAny<Order>()))
            .ReturnsAsync((Order order) => { order.Id = "ORDER001"; return order; });

        // Act
        var result = await _orderService.CreateOrderAsync(items, customerId);

        // Assert
        result.Should().NotBeNull();
        result.CustomerId.Should().Be(customerId);
        result.Items.Should().HaveCount(2);
        result.Subtotal.Should().Be(240m); // (100*2) + (50*1) = 250, minus 10 discount
        result.Total.Should().BeLessThan(result.Subtotal); // Gold tier discount applied
    }

    [Theory]
    [InlineData(100, "Bronze", 100)]
    [InlineData(100, "Silver", 95)]
    [InlineData(100, "Gold", 90)]
    [InlineData(100, "Platinum", 85)]
    public void CalculateDiscount_ShouldReturnCorrectAmount_ForDifferentCustomerTiers(
        decimal subtotal, string tier, decimal expected)
    {
        // Act
        var result = _orderService.CalculateDiscount(subtotal, tier);

        // Assert
        result.Should().Be(expected);
    }

    [Fact]
    public async Task CreateOrderAsync_ShouldThrowException_WhenCustomerNotFound()
    {
        // Arrange
        var customerId = "INVALID";
        var items = new List<OrderItem> { new() { Name = "Test", Price = 10, Quantity = 1 } };

        _customerRepositoryMock
            .Setup(x => x.GetCustomerTierAsync(customerId))
            .ThrowsAsync(new CustomerNotFoundException(customerId));

        // Act & Assert
        await _orderService.Invoking(x => x.CreateOrderAsync(items, customerId))
            .Should().ThrowAsync<CustomerNotFoundException>()
            .WithMessage($"Customer {customerId} not found");
    }
}

// Services/OrderService.cs - Implementation
public class OrderService : IOrderService
{
    private readonly ICustomerRepository _customerRepository;
    private readonly IOrderRepository _orderRepository;

    public OrderService(ICustomerRepository customerRepository, IOrderRepository orderRepository)
    {
        _customerRepository = customerRepository;
        _orderRepository = orderRepository;
    }

    public async Task<Order> CreateOrderAsync(List<OrderItem> items, string customerId)
    {
        if (items == null || !items.Any())
            throw new ArgumentException("Order must contain at least one item");

        var customerTier = await _customerRepository.GetCustomerTierAsync(customerId);

        var subtotal = items.Sum(i => (i.Price * i.Quantity) - i.Discount);
        var total = CalculateDiscount(subtotal, customerTier);

        var order = new Order
        {
            CustomerId = customerId,
            Items = items,
            Subtotal = subtotal,
            Total = total,
            CreatedAt = DateTime.UtcNow
        };

        return await _orderRepository.SaveOrderAsync(order);
    }

    public decimal CalculateDiscount(decimal subtotal, string customerTier)
    {
        var discountPercentage = customerTier switch
        {
            "Bronze" => 0m,
            "Silver" => 0.05m,
            "Gold" => 0.10m,
            "Platinum" => 0.15m,
            _ => 0m
        };

        return subtotal * (1 - discountPercentage);
    }
}

// Project file structure for TDD
// HotelManagement.Tests.csproj
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xunit" Version="2.4.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.4.5" />
    <PackageReference Include="Moq" Version="4.20.69" />
    <PackageReference Include="FluentAssertions" Version="6.12.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\HotelManagement.Core\HotelManagement.Core.csproj" />
  </ItemGroup>
</Project>
```
