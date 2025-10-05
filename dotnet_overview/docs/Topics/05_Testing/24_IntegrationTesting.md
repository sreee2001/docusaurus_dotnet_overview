## 9.2 Integration Testing

Integration testing verifies that different components or systems work correctly together. In .NET, this typically involves testing controllers, databases, external services, and the complete request-response pipeline.

### Official Definition/Standards

Integration testing is a software testing technique where individual software modules are combined and tested as a group. In ASP.NET Core, integration tests use the TestHost and WebApplicationFactory to test the entire HTTP request pipeline.

### Setup and Usage (Tools, Packages, Test Runners)

**Essential Packages:**

```bash
dotnet add package Microsoft.AspNetCore.Mvc.Testing
dotnet add package Microsoft.EntityFrameworkCore.InMemory
dotnet add package Microsoft.Extensions.DependencyInjection
dotnet add package Testcontainers  # For real database testing
```

**Test Infrastructure:**

- **WebApplicationFactory**: Creates a test server for ASP.NET Core applications
- **TestHost**: Hosts the application in-memory for testing
- **HttpClient**: Makes HTTP requests to the test server
- **In-Memory Database**: EF Core provider for testing without real database

### Typical Test Architecture and Patterns

**Common Patterns:**

- **Test Fixtures**: Shared test server setup
- **Custom WebApplicationFactory**: Override services for testing
- **Database Seeding**: Prepare test data
- **Test Containers**: Use real databases in Docker
- **API Client Testing**: Test HTTP endpoints end-to-end

### Example Integration Test Code

```csharp
// Package references:
// <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
// <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="8.0.0" />

using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
using System.Net.Http.Json;
using Xunit;
using FluentAssertions;

// Test startup configuration
public class CustomWebApplicationFactory<TStartup> : WebApplicationFactory<TStartup> where TStartup : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove the real database
            services.RemoveAll(typeof(DbContextOptions<HotelDbContext>));

            // Add in-memory database
            services.AddDbContext<HotelDbContext>(options =>
            {
                options.UseInMemoryDatabase("TestDatabase");
            });

            // Build service provider and seed database
            var serviceProvider = services.BuildServiceProvider();
            using var scope = serviceProvider.CreateScope();
            var context = scope.ServiceProvider.GetRequiredService<HotelDbContext>();

            SeedDatabase(context);
        });

        builder.UseEnvironment("Testing");
    }

    private static void SeedDatabase(HotelDbContext context)
    {
        context.Database.EnsureCreated();

        // Seed test data
        if (!context.Guests.Any())
        {
            context.Guests.AddRange(new[]
            {
                new Guest { Id = 1, FirstName = "John", LastName = "Doe", Email = "john@example.com" },
                new Guest { Id = 2, FirstName = "Jane", LastName = "Smith", Email = "jane@example.com" }
            });
        }

        if (!context.Rooms.Any())
        {
            context.Rooms.AddRange(new[]
            {
                new Room { Id = 101, RoomNumber = "101", RoomType = "Standard", PricePerNight = 99.99m, IsAvailable = true },
                new Room { Id = 102, RoomNumber = "102", RoomType = "Deluxe", PricePerNight = 149.99m, IsAvailable = true }
            });
        }

        context.SaveChanges();
    }
}

// Integration test class
public class BookingControllerIntegrationTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    private readonly CustomWebApplicationFactory<Program> _factory;

    public BookingControllerIntegrationTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task CreateBooking_WithValidData_ReturnsCreatedBooking()
    {
        // Arrange
        var createRequest = new CreateBookingRequest
        {
            GuestId = 1,
            RoomId = 101,
            CheckInDate = DateTime.Today.AddDays(1),
            CheckOutDate = DateTime.Today.AddDays(3),
            TotalAmount = 199.98m
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/bookings", createRequest);

        // Assert
        response.Should().HaveStatusCode(HttpStatusCode.Created);

        var booking = await response.Content.ReadFromJsonAsync<Booking>();
        booking.Should().NotBeNull();
        booking!.GuestId.Should().Be(createRequest.GuestId);
        booking.RoomId.Should().Be(createRequest.RoomId);
        booking.TotalAmount.Should().Be(createRequest.TotalAmount);

        // Verify location header
        response.Headers.Location.Should().NotBeNull();
        response.Headers.Location!.ToString().Should().Contain($"/api/bookings/{booking.Id}");
    }

    [Fact]
    public async Task GetBooking_WithExistingId_ReturnsBooking()
    {
        // Arrange - Create a booking first
        var createRequest = new CreateBookingRequest
        {
            GuestId = 1,
            RoomId = 101,
            CheckInDate = DateTime.Today.AddDays(1),
            CheckOutDate = DateTime.Today.AddDays(3),
            TotalAmount = 199.98m
        };

        var createResponse = await _client.PostAsJsonAsync("/api/bookings", createRequest);
        var createdBooking = await createResponse.Content.ReadFromJsonAsync<Booking>();

        // Act
        var getResponse = await _client.GetAsync($"/api/bookings/{createdBooking!.Id}");

        // Assert
        getResponse.Should().HaveStatusCode(HttpStatusCode.OK);

        var retrievedBooking = await getResponse.Content.ReadFromJsonAsync<Booking>();
        retrievedBooking.Should().NotBeNull();
        retrievedBooking!.Id.Should().Be(createdBooking.Id);
        retrievedBooking.GuestId.Should().Be(createRequest.GuestId);
    }

    [Fact]
    public async Task GetBooking_WithNonExistentId_ReturnsNotFound()
    {
        // Act
        var response = await _client.GetAsync("/api/bookings/99999");

        // Assert
        response.Should().HaveStatusCode(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task CreateBooking_WithInvalidData_ReturnsBadRequest()
    {
        // Arrange
        var invalidRequest = new CreateBookingRequest
        {
            GuestId = 999, // Non-existent guest
            RoomId = 101,
            CheckInDate = DateTime.Today.AddDays(3),
            CheckOutDate = DateTime.Today.AddDays(1), // Invalid dates
            TotalAmount = -100m // Invalid amount
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/bookings", invalidRequest);

        // Assert
        response.Should().HaveStatusCode(HttpStatusCode.BadRequest);
    }

    [Fact]
    public async Task GetAllBookings_ReturnsBookingsList()
    {
        // Arrange - Create multiple bookings
        var bookingRequests = new[]
        {
            new CreateBookingRequest
            {
                GuestId = 1,
                RoomId = 101,
                CheckInDate = DateTime.Today.AddDays(1),
                CheckOutDate = DateTime.Today.AddDays(2),
                TotalAmount = 99.99m
            },
            new CreateBookingRequest
            {
                GuestId = 2,
                RoomId = 102,
                CheckInDate = DateTime.Today.AddDays(5),
                CheckOutDate = DateTime.Today.AddDays(7),
                TotalAmount = 299.98m
            }
        };

        foreach (var request in bookingRequests)
        {
            await _client.PostAsJsonAsync("/api/bookings", request);
        }

        // Act
        var response = await _client.GetAsync("/api/bookings");

        // Assert
        response.Should().HaveStatusCode(HttpStatusCode.OK);

        var bookings = await response.Content.ReadFromJsonAsync<List<Booking>>();
        bookings.Should().NotBeNull();
        bookings!.Should().HaveCountGreaterOrEqualTo(2);
    }
}

// Database integration test with real database using TestContainers
public class BookingRepositoryIntegrationTests : IAsyncLifetime
{
    private readonly SqlServerContainer _sqlContainer;
    private HotelDbContext _context = null!;

    public BookingRepositoryIntegrationTests()
    {
        _sqlContainer = new SqlServerBuilder()
            .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
            .WithPassword("YourStrong!Passw0rd")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();

        var connectionString = _sqlContainer.GetConnectionString();
        var options = new DbContextOptionsBuilder<HotelDbContext>()
            .UseSqlServer(connectionString)
            .Options;

        _context = new HotelDbContext(options);
        await _context.Database.EnsureCreatedAsync();

        // Seed test data
        await SeedTestDataAsync();
    }

    public async Task DisposeAsync()
    {
        await _context.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }

    private async Task SeedTestDataAsync()
    {
        _context.Guests.Add(new Guest { FirstName = "Test", LastName = "User", Email = "test@example.com" });
        _context.Rooms.Add(new Room { RoomNumber = "201", RoomType = "Suite", PricePerNight = 199.99m, IsAvailable = true });
        await _context.SaveChangesAsync();
    }

    [Fact]
    public async Task AddBooking_WithValidData_SavesToDatabase()
    {
        // Arrange
        var repository = new BookingRepository(_context);
        var booking = new Booking
        {
            GuestId = 1,
            RoomId = 1,
            CheckInDate = DateTime.Today.AddDays(1),
            CheckOutDate = DateTime.Today.AddDays(3),
            TotalAmount = 399.98m
        };

        // Act
        var result = await repository.AddAsync(booking);

        // Assert
        result.Should().NotBeNull();
        result.Id.Should().BeGreaterThan(0);

        // Verify in database
        var savedBooking = await _context.Bookings.FindAsync(result.Id);
        savedBooking.Should().NotBeNull();
        savedBooking!.TotalAmount.Should().Be(399.98m);
    }
}
```

### When to Use and When Not to Use

**Use Integration Testing when:**

- Testing API endpoints and controllers
- Verifying database operations and queries
- Testing authentication and authorization
- Validating request/response serialization
- Testing middleware pipeline
- Verifying cross-component interactions

**Don't use Integration Testing when:**

- Testing pure business logic (use unit tests)
- Simple validation logic
- Performance-critical test suites
- Testing external service integrations (use contract tests)

### Pros and Cons and Alternatives

**Pros:**

- Tests realistic scenarios
- Catches integration issues
- Validates entire request pipeline
- Tests actual database interactions
- Provides confidence in deployments
- Tests serialization/deserialization

**Cons:**

- Slower execution than unit tests
- More complex setup and teardown
- Harder to isolate failures
- Database state management complexity
- Resource intensive
- Can be flaky due to external dependencies

**Alternatives:**

- Contract testing (Pact)
- Component testing
- End-to-end testing
- Database unit tests
- API testing tools (Postman, Newman)

—continued—

## 9.3 End-to-End Testing

End-to-end testing validates complete user workflows by testing the application from the user interface through all layers to the database. It simulates real user interactions to ensure the entire system works together correctly.

### Official Definition/Standards

End-to-end testing is a methodology used to test whether the flow of an application is performing as designed from start to finish. It tests the complete user journey and validates that all integrated components work together in production-like environments.

### Setup and Usage (Tools, Packages, Test Runners)

**Primary E2E Testing Tools:**

- **Playwright**: Modern, fast, cross-browser automation (Microsoft)
- **Selenium WebDriver**: Mature, widely-supported browser automation
- **Cypress**: JavaScript-based, developer-friendly testing framework
- **SpecFlow**: BDD framework for .NET with Gherkin syntax

**Essential Packages:**

```bash
# Playwright
dotnet add package Microsoft.Playwright
dotnet add package Microsoft.Playwright.NUnit

# Selenium
dotnet add package Selenium.WebDriver
dotnet add package Selenium.WebDriver.ChromeDriver
dotnet add package DotNetSeleniumExtras.WaitHelpers

# SpecFlow for BDD
dotnet add package SpecFlow
dotnet add package SpecFlow.NUnit
```

**Test Infrastructure:**

- Real browsers (Chrome, Firefox, Edge, Safari)
- Browser drivers and automation APIs
- Test data management and cleanup
- Screenshot and video capture
- Cross-platform and mobile testing

### Typical Test Architecture and Patterns

**Common Patterns:**

- **Page Object Model**: Encapsulate page elements and actions
- **Page Factory**: Initialize page elements automatically
- **Test Data Builders**: Create test data consistently
- **Base Test Classes**: Shared setup and teardown
- **Fluent APIs**: Chain actions for readable tests
- **BDD Scenarios**: Given-When-Then structure

**Project Structure:**

```
MyProject.E2ETests/
├── PageObjects/
│   ├── LoginPage.cs
│   ├── BookingPage.cs
│   └── DashboardPage.cs
├── TestData/
├── Helpers/
├── Features/  # For BDD/SpecFlow
└── Tests/
```

### Example E2E Test Code

```csharp
// Package references:
// <PackageReference Include="Microsoft.Playwright" Version="1.40.0" />
// <PackageReference Include="Microsoft.Playwright.NUnit" Version="1.40.0" />

using Microsoft.Playwright;
using Microsoft.Playwright.NUnit;
using NUnit.Framework;

// Playwright E2E Tests
[Parallelizable(ParallelScope.Self)]
[TestFixture]
public class HotelBookingE2ETests : PageTest
{
    private string _baseUrl = "https://localhost:7001"; // Your app URL

    [SetUp]
    public async Task Setup()
    {
        // Configure browser options
        await Context.Tracing.StartAsync(new()
        {
            Title = TestContext.CurrentContext.Test.Name,
            Screenshots = true,
            Snapshots = true,
            Sources = true
        });
    }

    [TearDown]
    public async Task TearDown()
    {
        // Save trace for debugging failures
        await Context.Tracing.StopAsync(new()
        {
            Path = Path.Combine(TestContext.CurrentContext.WorkDirectory,
                   $"{TestContext.CurrentContext.Test.Name}.zip")
        });
    }

    [Test]
    public async Task CompleteBookingFlow_ValidUser_BookingSuccessful()
    {
        // Arrange - Navigate to home page
        await Page.GotoAsync(_baseUrl);
        await Page.WaitForLoadStateAsync(LoadState.NetworkIdle);

        // Act & Assert - Check rooms availability
        await Page.ClickAsync("[data-testid='check-availability']");

        await Page.FillAsync("[data-testid='checkin-date']",
            DateTime.Today.AddDays(7).ToString("yyyy-MM-dd"));
        await Page.FillAsync("[data-testid='checkout-date']",
            DateTime.Today.AddDays(10).ToString("yyyy-MM-dd"));
        await Page.SelectOptionAsync("[data-testid='guests']", "2");

        await Page.ClickAsync("[data-testid='search-rooms']");
        await Page.WaitForSelectorAsync("[data-testid='room-list']");

        // Verify rooms are displayed
        var roomCards = await Page.Locator("[data-testid='room-card']").CountAsync();
        Assert.That(roomCards, Is.GreaterThan(0), "No rooms found for the selected dates");

        // Select first available room
        await Page.ClickAsync("[data-testid='room-card']:first-child [data-testid='book-room']");
        await Page.WaitForURLAsync("**/booking/**");

        // Fill guest information
        await Page.FillAsync("[data-testid='first-name']", "John");
        await Page.FillAsync("[data-testid='last-name']", "Doe");
        await Page.FillAsync("[data-testid='email']", "john.doe@example.com");
        await Page.FillAsync("[data-testid='phone']", "555-123-4567");

        // Fill payment information
        await Page.FillAsync("[data-testid='card-number']", "4111111111111111");
        await Page.FillAsync("[data-testid='expiry']", "12/25");
        await Page.FillAsync("[data-testid='cvv']", "123");
        await Page.FillAsync("[data-testid='cardholder-name']", "John Doe");

        // Submit booking
        await Page.ClickAsync("[data-testid='submit-booking']");

        // Wait for confirmation page
        await Page.WaitForURLAsync("**/booking/confirmation/**");
        await Page.WaitForSelectorAsync("[data-testid='booking-confirmation']");

        // Assert booking confirmation
        var confirmationMessage = await Page.TextContentAsync("[data-testid='confirmation-message']");
        Assert.That(confirmationMessage, Does.Contain("Your booking has been confirmed"));

        var bookingNumber = await Page.TextContentAsync("[data-testid='booking-number']");
        Assert.That(bookingNumber, Is.Not.Empty, "Booking number should be displayed");

        // Verify booking details
        var guestName = await Page.TextContentAsync("[data-testid='guest-name']");
        Assert.That(guestName, Does.Contain("John Doe"));

        var dates = await Page.TextContentAsync("[data-testid='booking-dates']");
        Assert.That(dates, Is.Not.Empty, "Booking dates should be displayed");
    }

    [Test]
    public async Task LoginFlow_ValidCredentials_RedirectsToDashboard()
    {
        // Navigate to login page
        await Page.GotoAsync($"{_baseUrl}/login");

        // Fill login form
        await Page.FillAsync("[data-testid='email']", "admin@hotel.com");
        await Page.FillAsync("[data-testid='password']", "AdminPassword123!");

        // Submit form
        await Page.ClickAsync("[data-testid='login-submit']");

        // Wait for redirect to dashboard
        await Page.WaitForURLAsync("**/dashboard");

        // Verify dashboard elements
        await Page.WaitForSelectorAsync("[data-testid='dashboard-header']");
        var welcomeMessage = await Page.TextContentAsync("[data-testid='welcome-message']");
        Assert.That(welcomeMessage, Does.Contain("Welcome"));
    }

    [Test]
    public async Task SearchRooms_NoAvailableRooms_ShowsNoResultsMessage()
    {
        await Page.GotoAsync(_baseUrl);

        // Search for rooms in the past (should have no results)
        await Page.FillAsync("[data-testid='checkin-date']",
            DateTime.Today.AddDays(-10).ToString("yyyy-MM-dd"));
        await Page.FillAsync("[data-testid='checkout-date']",
            DateTime.Today.AddDays(-5).ToString("yyyy-MM-dd"));

        await Page.ClickAsync("[data-testid='search-rooms']");

        // Wait for no results message
        await Page.WaitForSelectorAsync("[data-testid='no-rooms-message']");
        var message = await Page.TextContentAsync("[data-testid='no-rooms-message']");
        Assert.That(message, Does.Contain("No rooms available"));
    }
}

// Page Object Model example
public class BookingPage
{
    private readonly IPage _page;

    public BookingPage(IPage page)
    {
        _page = page;
    }

    // Locators
    private ILocator FirstNameInput => _page.Locator("[data-testid='first-name']");
    private ILocator LastNameInput => _page.Locator("[data-testid='last-name']");
    private ILocator EmailInput => _page.Locator("[data-testid='email']");
    private ILocator SubmitButton => _page.Locator("[data-testid='submit-booking']");
    private ILocator ConfirmationMessage => _page.Locator("[data-testid='confirmation-message']");

    // Actions
    public async Task FillGuestInformation(string firstName, string lastName, string email)
    {
        await FirstNameInput.FillAsync(firstName);
        await LastNameInput.FillAsync(lastName);
        await EmailInput.FillAsync(email);
    }

    public async Task SubmitBooking()
    {
        await SubmitButton.ClickAsync();
        await _page.WaitForURLAsync("**/booking/confirmation/**");
    }

    public async Task<string> GetConfirmationMessage()
    {
        await ConfirmationMessage.WaitForAsync();
        return await ConfirmationMessage.TextContentAsync() ?? string.Empty;
    }
}

// Selenium WebDriver alternative example
public class SeleniumE2ETests
{
    private IWebDriver _driver = null!;
    private readonly string _baseUrl = "https://localhost:7001";

    [SetUp]
    public void Setup()
    {
        var options = new ChromeOptions();
        options.AddArguments("--headless"); // Run in headless mode for CI
        _driver = new ChromeDriver(options);
        _driver.Manage().Timeouts().ImplicitWait = TimeSpan.FromSeconds(10);
    }

    [TearDown]
    public void TearDown()
    {
        _driver?.Quit();
        _driver?.Dispose();
    }

    [Test]
    public void BookRoom_ValidData_ShowsConfirmation()
    {
        // Navigate to home page
        _driver.Navigate().GoToUrl(_baseUrl);

        // Fill search form
        var checkinDate = _driver.FindElement(By.CssSelector("[data-testid='checkin-date']"));
        checkinDate.SendKeys(DateTime.Today.AddDays(7).ToString("MM/dd/yyyy"));

        var checkoutDate = _driver.FindElement(By.CssSelector("[data-testid='checkout-date']"));
        checkoutDate.SendKeys(DateTime.Today.AddDays(10).ToString("MM/dd/yyyy"));

        // Search rooms
        var searchButton = _driver.FindElement(By.CssSelector("[data-testid='search-rooms']"));
        searchButton.Click();

        // Wait for rooms to load
        var wait = new WebDriverWait(_driver, TimeSpan.FromSeconds(10));
        wait.Until(ExpectedConditions.ElementIsVisible(By.CssSelector("[data-testid='room-list']")));

        // Book first room
        var bookButton = _driver.FindElement(By.CssSelector("[data-testid='room-card']:first-child [data-testid='book-room']"));
        bookButton.Click();

        // Fill booking form
        _driver.FindElement(By.CssSelector("[data-testid='first-name']")).SendKeys("Jane");
        _driver.FindElement(By.CssSelector("[data-testid='last-name']")).SendKeys("Smith");
        _driver.FindElement(By.CssSelector("[data-testid='email']")).SendKeys("jane@example.com");

        // Submit booking
        _driver.FindElement(By.CssSelector("[data-testid='submit-booking']")).Click();

        // Verify confirmation
        wait.Until(ExpectedConditions.ElementIsVisible(By.CssSelector("[data-testid='booking-confirmation']")));
        var confirmationText = _driver.FindElement(By.CssSelector("[data-testid='confirmation-message']")).Text;

        Assert.That(confirmationText, Does.Contain("confirmed"));
    }
}

// BDD with SpecFlow example
[Binding]
public class BookingSteps
{
    private readonly IPage _page;
    private readonly BookingPage _bookingPage;

    public BookingSteps(IPage page)
    {
        _page = page;
        _bookingPage = new BookingPage(page);
    }

    [Given(@"I am on the hotel booking website")]
    public async Task GivenIAmOnTheHotelBookingWebsite()
    {
        await _page.GotoAsync("https://localhost:7001");
    }

    [When(@"I search for rooms from ""(.*)"" to ""(.*)""")]
    public async Task WhenISearchForRoomsFromTo(string checkin, string checkout)
    {
        await _page.FillAsync("[data-testid='checkin-date']", checkin);
        await _page.FillAsync("[data-testid='checkout-date']", checkout);
        await _page.ClickAsync("[data-testid='search-rooms']");
    }

    [Then(@"I should see available rooms")]
    public async Task ThenIShouldSeeAvailableRooms()
    {
        await _page.WaitForSelectorAsync("[data-testid='room-list']");
        var roomCount = await _page.Locator("[data-testid='room-card']").CountAsync();
        Assert.That(roomCount, Is.GreaterThan(0));
    }
}

// Feature file (BookingFlow.feature)
/*
Feature: Hotel Room Booking
    As a hotel guest
    I want to book a room online
    So that I can secure accommodation for my stay

Scenario: Successful room booking
    Given I am on the hotel booking website
    When I search for rooms from "2024-12-01" to "2024-12-03"
    Then I should see available rooms
    When I select the first available room
    And I fill in my guest details
    And I provide payment information
    And I submit the booking
    Then I should see a booking confirmation
*/
```

### When to Use and When Not to Use

**Use E2E Testing when:**

- Testing critical user journeys and workflows
- Validating cross-browser compatibility
- Testing complete feature functionality
- Verifying production-like scenarios
- Testing user interface interactions
- Validating third-party integrations

**Don't use E2E Testing when:**

- Testing individual business logic (use unit tests)
- Fast feedback is needed during development
- Testing internal APIs without UI
- Limited CI/CD execution time
- Testing detailed edge cases
- Validating specific component behavior

### Pros and Cons and Alternatives

**Pros:**

- Tests real user scenarios
- Validates entire application stack
- Catches integration and UI issues
- Provides high confidence in releases
- Tests cross-browser compatibility
- Validates user experience

**Cons:**

- Slow execution and feedback
- Brittle and hard to maintain
- Expensive to run and maintain
- Complex debugging and troubleshooting
- Flaky due to timing and environment issues
- Requires more infrastructure

**Alternatives:**

- Visual regression testing (Percy, Chromatic)
- Manual exploratory testing
- Smoke testing for critical paths
- User acceptance testing
- Component testing with Storybook
- API contract testing

## 9.4 Load Testing

Load testing evaluates how an application performs under expected and peak load conditions. It helps identify performance bottlenecks, resource limitations, and scalability issues before they impact users in production.

### Official Definition/Standards

Load testing is a type of performance testing that simulates realistic user loads to evaluate application behavior under normal and anticipated peak conditions. It measures response times, throughput, resource utilization, and identifies the breaking point of applications.

### Setup and Usage (Tools, Packages, Test Runners)

**Primary Load Testing Tools:**

- **NBomber**: .NET-native load testing framework
- **k6**: Modern JavaScript-based load testing tool
- **Apache JMeter**: GUI-based, widely-used performance testing tool
- **Artillery**: Node.js-based, cloud-native testing framework
- **Azure Load Testing**: Cloud-based service for load testing

**Essential Packages:**

```bash
# NBomber for .NET
dotnet add package NBomber
dotnet add package NBomber.Http

# For HTTP client testing
dotnet add package Microsoft.Extensions.Http
```

**Test Infrastructure:**

- Load generators and test agents
- Monitoring and metrics collection
- Resource monitoring (CPU, memory, database)
- Network and latency simulation
- Distributed testing capabilities

### Typical Test Architecture and Patterns

**Common Patterns:**

- **Ramp-up Testing**: Gradually increase load
- **Sustained Load**: Maintain constant load over time
- **Spike Testing**: Sudden load increases
- **Volume Testing**: Large amounts of data
- **Stress Testing**: Beyond normal capacity
- **Soak Testing**: Extended duration testing

### Example Load Test Code

```csharp
// Package references:
// <PackageReference Include="NBomber" Version="5.0.0" />
// <PackageReference Include="NBomber.Http" Version="5.0.0" />

using NBomber.CSharp;
using NBomber.Http.CSharp;

// NBomber Load Testing Example
public class HotelApiLoadTests
{
    public static void RunBookingLoadTest()
    {
        // HTTP client configuration
        using var httpClient = new HttpClient();
        httpClient.BaseAddress = new Uri("https://localhost:7001");

        // Test scenario for booking API
        var bookingScenario = Scenario.Create("booking_api_test", async context =>
        {
            var bookingRequest = CreateBookingRequest();

            var response = await httpClient.PostAsJsonAsync("/api/bookings", bookingRequest);

            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.InjectPerSec(rate: 10, during: TimeSpan.FromMinutes(5)), // Warm-up
            Simulation.InjectPerSec(rate: 50, during: TimeSpan.FromMinutes(10)), // Normal load
            Simulation.InjectPerSec(rate: 100, during: TimeSpan.FromMinutes(5))  // Peak load
        );

        // Test scenario for search API
        var searchScenario = Scenario.Create("search_api_test", async context =>
        {
            var searchQuery = "?checkin=2024-12-01&checkout=2024-12-03&guests=2";

            var response = await httpClient.GetAsync($"/api/rooms/search{searchQuery}");

            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.InjectPerSec(rate: 20, during: TimeSpan.FromMinutes(15))
        );

        // Run the load test
        NBomberRunner
            .RegisterScenarios(bookingScenario, searchScenario)
            .WithReportFolder("load_test_reports")
            .WithReportFormats(ReportFormat.Txt, ReportFormat.Html, ReportFormat.Csv)
            .Run();
    }

    private static CreateBookingRequest CreateBookingRequest()
    {
        var random = new Random();
        return new CreateBookingRequest
        {
            GuestId = random.Next(1, 1000),
            RoomId = random.Next(101, 200),
            CheckInDate = DateTime.Today.AddDays(random.Next(1, 30)),
            CheckOutDate = DateTime.Today.AddDays(random.Next(31, 60)),
            TotalAmount = random.Next(100, 500)
        };
    }
}

// Advanced NBomber scenario with data feeding
public class AdvancedLoadTests
{
    public static void RunDataDrivenLoadTest()
    {
        // Data feed for test data
        var guestData = Data.Feed(() => new
        {
            GuestId = Random.Shared.Next(1, 1000),
            RoomType = new[] { "Standard", "Deluxe", "Suite" }[Random.Shared.Next(0, 3)],
            Duration = Random.Shared.Next(1, 7)
        });

        var bookingWithDataFeed = Scenario.Create("booking_with_data_feed", async context =>
        {
            var data = context.Data;

            // Search for rooms first
            using var httpClient = new HttpClient();
            httpClient.BaseAddress = new Uri("https://localhost:7001");

            var searchResponse = await httpClient.GetAsync(
                $"/api/rooms/search?type={data.RoomType}&duration={data.Duration}");

            if (!searchResponse.IsSuccessStatusCode)
                return Response.Fail("Search failed");

            // Create booking
            var bookingRequest = new CreateBookingRequest
            {
                GuestId = data.GuestId,
                RoomId = Random.Shared.Next(101, 200),
                CheckInDate = DateTime.Today.AddDays(1),
                CheckOutDate = DateTime.Today.AddDays(1 + data.Duration),
                TotalAmount = 100 * data.Duration
            };

            var bookingResponse = await httpClient.PostAsJsonAsync("/api/bookings", bookingRequest);

            return bookingResponse.IsSuccessStatusCode ? Response.Ok() : Response.Fail("Booking failed");
        })
        .WithDataFeed(guestData)
        .WithLoadSimulations(
            Simulation.InjectPerSec(rate: 10, during: TimeSpan.FromMinutes(2)),
            Simulation.InjectPerSec(rate: 25, during: TimeSpan.FromMinutes(5)),
            Simulation.InjectPerSec(rate: 50, during: TimeSpan.FromMinutes(3))
        );

        NBomberRunner
            .RegisterScenarios(bookingWithDataFeed)
            .WithReportFolder("advanced_load_test_reports")
            .Run();
    }

    // Stress testing to find breaking point
    public static void RunStressTest()
    {
        var stressScenario = Scenario.Create("stress_test", async context =>
        {
            using var httpClient = new HttpClient();
            httpClient.BaseAddress = new Uri("https://localhost:7001");
            httpClient.Timeout = TimeSpan.FromSeconds(30);

            var response = await httpClient.GetAsync("/api/rooms");

            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.InjectPerSec(rate: 10, during: TimeSpan.FromMinutes(1)),   // Baseline
            Simulation.InjectPerSec(rate: 50, during: TimeSpan.FromMinutes(2)),   // Normal
            Simulation.InjectPerSec(rate: 100, during: TimeSpan.FromMinutes(2)),  // High
            Simulation.InjectPerSec(rate: 200, during: TimeSpan.FromMinutes(2)),  // Very high
            Simulation.InjectPerSec(rate: 500, during: TimeSpan.FromMinutes(2)),  // Extreme
            Simulation.InjectPerSec(rate: 1000, during: TimeSpan.FromMinutes(1))  // Breaking point
        );

        NBomberRunner
            .RegisterScenarios(stressScenario)
            .WithReportFolder("stress_test_reports")
            .Run();
    }
}

// Integration with ASP.NET Core TestServer for isolated testing
public class InProcessLoadTests
{
    [Test]
    public void LoadTest_BookingAPI_WithTestServer()
    {
        var webAppFactory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Configure in-memory database for testing
                    services.AddDbContext<HotelDbContext>(options =>
                        options.UseInMemoryDatabase("LoadTestDb"));
                });
            });

        var httpClient = webAppFactory.CreateClient();

        var scenario = Scenario.Create("in_process_test", async context =>
        {
            var bookingRequest = new CreateBookingRequest
            {
                GuestId = Random.Shared.Next(1, 100),
                RoomId = Random.Shared.Next(101, 110),
                CheckInDate = DateTime.Today.AddDays(1),
                CheckOutDate = DateTime.Today.AddDays(3),
                TotalAmount = 299.99m
            };

            var response = await httpClient.PostAsJsonAsync("/api/bookings", bookingRequest);

            return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.InjectPerSec(rate: 100, during: TimeSpan.FromMinutes(2))
        );

        NBomberRunner
            .RegisterScenarios(scenario)
            .WithReportFolder("in_process_load_test_reports")
            .Run();

        webAppFactory.Dispose();
    }
}

// k6 JavaScript alternative example
/*
// k6-load-test.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 10 }, // Ramp up
    { duration: '5m', target: 50 }, // Stay at 50 users
    { duration: '2m', target: 100 }, // Ramp up to 100 users
    { duration: '5m', target: 100 }, // Stay at 100 users
    { duration: '2m', target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests should be below 500ms
    http_req_failed: ['rate<0.02'],   // Error rate should be less than 2%
  },
};

const BASE_URL = 'https://localhost:7001';

export default function () {
  group('Hotel Booking Flow', function () {
    // Search for rooms
    let searchResponse = http.get(`${BASE_URL}/api/rooms/search?checkin=2024-12-01&checkout=2024-12-03`);
    check(searchResponse, {
      'search status is 200': (r) => r.status === 200,
      'search response time < 200ms': (r) => r.timings.duration < 200,
    });

    sleep(1);

    // Create booking
    let bookingPayload = JSON.stringify({
      guestId: Math.floor(Math.random() * 1000) + 1,
      roomId: Math.floor(Math.random() * 100) + 101,
      checkInDate: '2024-12-01',
      checkOutDate: '2024-12-03',
      totalAmount: 299.99
    });

    let bookingResponse = http.post(`${BASE_URL}/api/bookings`, bookingPayload, {
      headers: { 'Content-Type': 'application/json' },
    });

    check(bookingResponse, {
      'booking status is 201': (r) => r.status === 201,
      'booking response time < 500ms': (r) => r.timings.duration < 500,
    });

    sleep(1);
  });
}

// Run with: k6 run k6-load-test.js
*/
```

### When to Use and When Not to Use

**Use Load Testing when:**

- Preparing for production deployment
- Validating performance requirements
- Identifying system bottlenecks
- Planning capacity and scaling
- Testing under realistic conditions
- Verifying SLA compliance

**Don't use Load Testing when:**

- Testing individual components (use unit tests)
- Early development stages
- Testing business logic correctness
- Limited testing environment resources
- Functional issues still exist
- Testing non-performance requirements

### Pros and Cons and Alternatives

**Pros:**

- Identifies performance bottlenecks
- Validates scalability limits
- Provides capacity planning data
- Tests realistic user scenarios
- Prevents production performance issues
- Helps optimize resource usage

**Cons:**

- Requires significant infrastructure
- Time-consuming to setup and run
- Expensive to maintain
- May not represent real user behavior
- Can be complex to analyze results
- Environment differences affect results

**Alternatives:**

- Application Performance Monitoring (APM)
- Synthetic monitoring
- Profiling and benchmarking tools
- Cloud-based load testing services
- Manual performance testing
- Chaos engineering practices
