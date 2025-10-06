---
slug: unit_testing
title: Unit Testing
tags: [dotnet, testing, unit_testing, quality]
---

# Unit Testing

Unit testing involves testing individual components or methods in isolation to verify they work as expected. It's the foundation of a robust testing strategy and enables rapid feedback during development.

## Official Definition/Standards

Unit testing is a software testing method where individual units or components of software are tested in isolation. In .NET, the standard approach follows the AAA pattern (Arrange, Act, Assert) and uses frameworks like xUnit, NUnit, or MSTest with mocking libraries like Moq.

## Setup and Usage (Tools, Packages, Test Runners)

### Primary Testing Frameworks:

- **xUnit.net**: Modern, extensible testing framework (recommended for new projects)
- **NUnit**: Feature-rich framework with extensive assertions
- **MSTest**: Microsoft's testing framework, integrated with Visual Studio

### Essential Packages:

```bash
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package Moq  # For mocking
dotnet add package FluentAssertions  # Enhanced assertions
```

### Test Runners:

- Visual Studio Test Explorer
- dotnet test CLI command
- JetBrains Rider
- VS Code with C# extension

## Typical Test Architecture and Patterns

### Common Patterns:

- **AAA Pattern**: Arrange (setup), Act (execute), Assert (verify)
- **Given-When-Then**: BDD-style test structure
- **Test Fixtures**: Shared setup and teardown logic
- **Parametrized Tests**: Data-driven testing
- **Mock/Stub/Fake**: Test doubles for dependencies

### Project Structure:

```
MyProject.sln
├── src/
│   └── MyProject/
│       ├── Services/
│       └── Models/
└── tests/
    └── MyProject.Tests/
        ├── Services/
        └── TestData/
```

## Example Test Code

```csharp
// Package references in test project:
// <PackageReference Include="xunit" Version="2.4.2" />
// <PackageReference Include="xunit.runner.visualstudio" Version="2.4.5" />
// <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
// <PackageReference Include="Moq" Version="4.20.69" />
// <PackageReference Include="FluentAssertions" Version="6.12.0" />

using Xunit;
using Moq;
using FluentAssertions;

// System under test
public class BookingService
{
    private readonly IBookingRepository _bookingRepository;
    private readonly IRoomService _roomService;
    private readonly IEmailService _emailService;

    public BookingService(IBookingRepository bookingRepository, IRoomService roomService, IEmailService emailService)
    {
        _bookingRepository = bookingRepository;
        _roomService = roomService;
        _emailService = emailService;
    }

    public async Task<BookingResult> CreateBookingAsync(CreateBookingRequest request)
    {
        if (request.CheckInDate >= request.CheckOutDate)
            return BookingResult.Failure("Check-in date must be before check-out date");

        var isAvailable = await _roomService.IsRoomAvailableAsync(request.RoomId, request.CheckInDate, request.CheckOutDate);
        if (!isAvailable)
            return BookingResult.Failure("Room is not available for the selected dates");

        var booking = new Booking
        {
            GuestId = request.GuestId,
            RoomId = request.RoomId,
            CheckInDate = request.CheckInDate,
            CheckOutDate = request.CheckOutDate,
            TotalAmount = request.TotalAmount
        };

        var savedBooking = await _bookingRepository.AddAsync(booking);
        await _emailService.SendBookingConfirmationAsync(savedBooking);

        return BookingResult.Success(savedBooking);
    }
}

public class BookingResult
{
    public bool IsSuccess { get; private set; }
    public string ErrorMessage { get; private set; } = string.Empty;
    public Booking? Booking { get; private set; }

    public static BookingResult Success(Booking booking) => new() { IsSuccess = true, Booking = booking };
    public static BookingResult Failure(string error) => new() { IsSuccess = false, ErrorMessage = error };
}

// Unit Tests
public class BookingServiceTests
{
    private readonly Mock<IBookingRepository> _mockBookingRepository;
    private readonly Mock<IRoomService> _mockRoomService;
    private readonly Mock<IEmailService> _mockEmailService;
    private readonly BookingService _bookingService;

    public BookingServiceTests()
    {
        _mockBookingRepository = new Mock<IBookingRepository>();
        _mockRoomService = new Mock<IRoomService>();
        _mockEmailService = new Mock<IEmailService>();
        _bookingService = new BookingService(_mockBookingRepository.Object, _mockRoomService.Object, _mockEmailService.Object);
    }

    [Fact]
    public async Task CreateBookingAsync_WithValidRequest_ReturnsSuccessResult()
    {
        // Arrange
        var request = new CreateBookingRequest
        {
            GuestId = 1,
            RoomId = 101,
            CheckInDate = DateTime.Today.AddDays(1),
            CheckOutDate = DateTime.Today.AddDays(3),
            TotalAmount = 200m
        };

        var expectedBooking = new Booking
        {
            Id = 1,
            GuestId = request.GuestId,
            RoomId = request.RoomId,
            CheckInDate = request.CheckInDate,
            CheckOutDate = request.CheckOutDate,
            TotalAmount = request.TotalAmount
        };

        _mockRoomService.Setup(x => x.IsRoomAvailableAsync(request.RoomId, request.CheckInDate, request.CheckOutDate))
                       .ReturnsAsync(true);
        _mockBookingRepository.Setup(x => x.AddAsync(It.IsAny<Booking>()))
                              .ReturnsAsync(expectedBooking);
        _mockEmailService.Setup(x => x.SendBookingConfirmationAsync(It.IsAny<Booking>()))
                         .Returns(Task.CompletedTask);

        // Act
        var result = await _bookingService.CreateBookingAsync(request);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Booking.Should().NotBeNull();
        result.Booking.Id.Should().Be(1);
        result.ErrorMessage.Should().BeEmpty();

        // Verify interactions
        _mockRoomService.Verify(x => x.IsRoomAvailableAsync(request.RoomId, request.CheckInDate, request.CheckOutDate), Times.Once);
        _mockBookingRepository.Verify(x => x.AddAsync(It.IsAny<Booking>()), Times.Once);
        _mockEmailService.Verify(x => x.SendBookingConfirmationAsync(expectedBooking), Times.Once);
    }

    [Fact]
    public async Task CreateBookingAsync_WithInvalidDates_ReturnsFailureResult()
    {
        // Arrange
        var request = new CreateBookingRequest
        {
            GuestId = 1,
            RoomId = 101,
            CheckInDate = DateTime.Today.AddDays(3),
            CheckOutDate = DateTime.Today.AddDays(1), // Invalid: check-out before check-in
            TotalAmount = 200m
        };

        // Act
        var result = await _bookingService.CreateBookingAsync(request);

        // Assert
        result.IsSuccess.Should().BeFalse();
        result.ErrorMessage.Should().Be("Check-in date must be before check-out date");
        result.Booking.Should().BeNull();

        // Verify no repository or email service calls were made
        _mockRoomService.Verify(x => x.IsRoomAvailableAsync(It.IsAny<int>(), It.IsAny<DateTime>(), It.IsAny<DateTime>()), Times.Never);
        _mockBookingRepository.Verify(x => x.AddAsync(It.IsAny<Booking>()), Times.Never);
        _mockEmailService.Verify(x => x.SendBookingConfirmationAsync(It.IsAny<Booking>()), Times.Never);
    }

    [Fact]
    public async Task CreateBookingAsync_WithUnavailableRoom_ReturnsFailureResult()
    {
        // Arrange
        var request = new CreateBookingRequest
        {
            GuestId = 1,
            RoomId = 101,
            CheckInDate = DateTime.Today.AddDays(1),
            CheckOutDate = DateTime.Today.AddDays(3),
            TotalAmount = 200m
        };

        _mockRoomService.Setup(x => x.IsRoomAvailableAsync(request.RoomId, request.CheckInDate, request.CheckOutDate))
                       .ReturnsAsync(false);

        // Act
        var result = await _bookingService.CreateBookingAsync(request);

        // Assert
        result.IsSuccess.Should().BeFalse();
        result.ErrorMessage.Should().Be("Room is not available for the selected dates");
        result.Booking.Should().BeNull();

        _mockBookingRepository.Verify(x => x.AddAsync(It.IsAny<Booking>()), Times.Never);
        _mockEmailService.Verify(x => x.SendBookingConfirmationAsync(It.IsAny<Booking>()), Times.Never);
    }

    [Theory]
    [InlineData(1, 101, 100.50)]
    [InlineData(2, 102, 250.75)]
    [InlineData(3, 103, 399.99)]
    public async Task CreateBookingAsync_WithDifferentValidInputs_ReturnsSuccessResult(int guestId, int roomId, decimal totalAmount)
    {
        // Arrange
        var request = new CreateBookingRequest
        {
            GuestId = guestId,
            RoomId = roomId,
            CheckInDate = DateTime.Today.AddDays(1),
            CheckOutDate = DateTime.Today.AddDays(3),
            TotalAmount = totalAmount
        };

        _mockRoomService.Setup(x => x.IsRoomAvailableAsync(It.IsAny<int>(), It.IsAny<DateTime>(), It.IsAny<DateTime>()))
                       .ReturnsAsync(true);
        _mockBookingRepository.Setup(x => x.AddAsync(It.IsAny<Booking>()))
                              .ReturnsAsync(new Booking { Id = 1 });

        // Act
        var result = await _bookingService.CreateBookingAsync(request);

        // Assert
        result.IsSuccess.Should().BeTrue();
    }
}

// Test data classes for complex scenarios
public class BookingTestData
{
    public static IEnumerable<object[]> GetValidBookingRequests()
    {
        yield return new object[]
        {
            new CreateBookingRequest
            {
                GuestId = 1,
                RoomId = 101,
                CheckInDate = DateTime.Today.AddDays(1),
                CheckOutDate = DateTime.Today.AddDays(2),
                TotalAmount = 99.99m
            }
        };

        yield return new object[]
        {
            new CreateBookingRequest
            {
                GuestId = 2,
                RoomId = 102,
                CheckInDate = DateTime.Today.AddDays(5),
                CheckOutDate = DateTime.Today.AddDays(7),
                TotalAmount = 199.98m
            }
        };
    }
}

public class BookingServiceComplexTests
{
    [Theory]
    [MemberData(nameof(BookingTestData.GetValidBookingRequests), MemberType = typeof(BookingTestData))]
    public async Task CreateBookingAsync_WithComplexTestData_ReturnsSuccessResult(CreateBookingRequest request)
    {
        // Arrange
        var mockRepository = new Mock<IBookingRepository>();
        var mockRoomService = new Mock<IRoomService>();
        var mockEmailService = new Mock<IEmailService>();
        var service = new BookingService(mockRepository.Object, mockRoomService.Object, mockEmailService.Object);

        mockRoomService.Setup(x => x.IsRoomAvailableAsync(It.IsAny<int>(), It.IsAny<DateTime>(), It.IsAny<DateTime>()))
                       .ReturnsAsync(true);
        mockRepository.Setup(x => x.AddAsync(It.IsAny<Booking>()))
                      .ReturnsAsync(new Booking { Id = 1 });

        // Act
        var result = await service.CreateBookingAsync(request);

        // Assert
        result.IsSuccess.Should().BeTrue();
    }
}
```

## When to Use and When Not to Use

### Use Unit Testing when:

- Testing business logic in isolation
- Verifying individual method behavior
- Ensuring code coverage for critical paths
- Enabling refactoring with confidence
- Supporting continuous integration
- Documenting expected behavior

### Don't use Unit Testing when:

- Testing UI interactions (use integration tests)
- Testing database queries (use integration tests)
- Testing third-party library integrations
- Over-testing simple property getters/setters
- Testing framework code that's already tested

## Pros and Cons and Alternatives

### Pros:

- Fast execution and feedback
- Isolates problems to specific components
- Enables safe refactoring
- Serves as living documentation
- Supports test-driven development
- Easy to automate in CI/CD

### Cons:

- Can miss integration issues
- Requires mocking dependencies
- May test implementation details
- Maintenance overhead for brittle tests
- False confidence from excessive mocking
- Time investment for comprehensive coverage

### Alternatives:

- Integration testing for broader coverage
- Property-based testing (FsCheck)
- Mutation testing for test quality
- Static analysis tools
- Code reviews and pair programming
