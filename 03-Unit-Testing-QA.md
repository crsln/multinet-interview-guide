# Unit Testing - Interview Questions

> 5 real-world interview questions with detailed answers and communication tactics

---

## Question 1: Why Is Unit Testing Necessary?

### The Question
> "Why is unit testing necessary? How would you convince a skeptical manager or team lead who says 'we don't have time for tests'?"

### Key Points to Cover
- Business value of testing
- Long-term time savings
- Quality and confidence benefits
- Addressing the "no time" objection

### Detailed Answer

**The Business Case for Unit Testing:**

**1. Bugs Are Cheaper to Fix Early:**

```
Cost to fix a bug at different stages:
- During development:      1x (baseline)
- During QA:               10x
- In production:           100x
- After customer finds it: 1000x (reputation damage)
```

**2. Tests Save Time, Not Cost Time:**

```csharp
// Without tests - the "quick" way that's actually slow:
// 1. Write code (30 min)
// 2. Manually test happy path (10 min)
// 3. Deploy
// 4. Bug found in production (2 hours to debug)
// 5. Fix and re-deploy (30 min)
// 6. Another edge case bug (repeat...)
// Total: 3+ hours, plus stress and reputation damage

// With tests:
// 1. Write test first (5 min)
// 2. Write code (30 min)
// 3. Test catches edge case immediately (0 min - already running)
// 4. Fix edge case (5 min)
// 5. Deploy with confidence
// Total: 40 min, deployed once, no production issues
```

**3. Enables Safe Refactoring:**

```csharp
// Without tests:
// "I'm afraid to touch this code because I might break something"
// Technical debt accumulates forever

// With tests:
public class PaymentCalculatorTests
{
    [Fact]
    public void Calculate_WithTaxAndDiscount_ReturnsCorrectTotal()
    {
        var calc = new PaymentCalculator();
        var result = calc.Calculate(100, taxRate: 0.18m, discount: 10);
        Assert.Equal(97.2m, result);
    }
}

// Now you can refactor PaymentCalculator with confidence
// If you break it, the test tells you immediately
```

**4. Tests Are Living Documentation:**

```csharp
// What does this method do with null input? Edge cases?
// Check the tests!

[Theory]
[InlineData(null, false)]
[InlineData("", false)]
[InlineData("invalid", false)]
[InlineData("user@example.com", true)]
[InlineData("USER@EXAMPLE.COM", true)]
public void IsValidEmail_WithVariousInputs_ReturnsExpected(string email, bool expected)
{
    var result = EmailValidator.IsValid(email);
    Assert.Equal(expected, result);
}
// This test documents exactly how the method handles edge cases
```

**Addressing "No Time for Tests":**

> "I understand the pressure to deliver quickly. But consider:
>
> 1. **We're already spending time on testing** - manually, inefficiently, repeatedly. Unit tests automate that.
>
> 2. **The time 'saved' is borrowed, not saved** - we'll pay it back with interest during debugging, production issues, and fear of refactoring.
>
> 3. **Start small** - don't test everything. Start with critical business logic. The payment calculation, the price engine, the validation rules.
>
> 4. **Tests actually speed up development** - I can run 500 tests in 10 seconds instead of 30 minutes of manual clicking.
>
> 5. **New team members ramp up faster** - tests show how the system works."

**Real-World Metric:**

```
Microsoft study on Windows teams:
- Teams with TDD had 60-90% fewer defects in production
- Initial development time increased 15-35%
- BUT: Total project time decreased due to fewer bugs and easier maintenance
```

### Communication Tactics

🎯 **Structure your answer**: Start with the cost of bugs, then show testing saves time in the long run. Give the refactoring and documentation benefits. End with addressing their concern directly.

💡 **Emphasize**: Testing is not about "quality vs speed" - it's about sustainable speed. Short-term we might be slightly slower, long-term we're much faster.

⚠️ **Avoid**: Don't be preachy or condescending. Acknowledge the real pressure to deliver. Focus on practical business outcomes, not testing theory.

---

## Question 2: [Fact] vs [Theory] in xUnit

### The Question
> "Explain the difference between [Fact] and [Theory] in xUnit. When would you use each? Can you show me examples?"

### Key Points to Cover
- [Fact] for single scenario tests
- [Theory] for data-driven tests
- Different data sources (InlineData, MemberData, ClassData)
- When to choose each

### Detailed Answer

**[Fact] - Single Scenario Test:**

```csharp
// Use [Fact] when testing ONE specific scenario
// The test has no parameters

[Fact]
public void Add_TwoPositiveNumbers_ReturnsSum()
{
    // Arrange
    var calculator = new Calculator();

    // Act
    var result = calculator.Add(2, 3);

    // Assert
    Assert.Equal(5, result);
}

[Fact]
public void CreateOrder_WithEmptyCart_ThrowsException()
{
    // Arrange
    var service = new OrderService();
    var emptyCart = new Cart();

    // Act & Assert
    Assert.Throws<InvalidOperationException>(() => service.CreateOrder(emptyCart));
}

[Fact]
public async Task GetUser_WhenNotFound_ReturnsNull()
{
    // Arrange
    var mockRepo = new Mock<IUserRepository>();
    mockRepo.Setup(r => r.GetByIdAsync(999)).ReturnsAsync((User)null);
    var service = new UserService(mockRepo.Object);

    // Act
    var result = await service.GetUserAsync(999);

    // Assert
    Assert.Null(result);
}
```

**[Theory] - Data-Driven Test:**

```csharp
// Use [Theory] when testing MULTIPLE scenarios with same logic
// The test has parameters that vary

// InlineData - simple inline values
[Theory]
[InlineData(1, 2, 3)]
[InlineData(0, 0, 0)]
[InlineData(-1, 1, 0)]
[InlineData(100, 200, 300)]
public void Add_WithVariousInputs_ReturnsExpectedSum(int a, int b, int expected)
{
    var calculator = new Calculator();
    var result = calculator.Add(a, b);
    Assert.Equal(expected, result);
}

// InlineData with strings
[Theory]
[InlineData("hello", "HELLO")]
[InlineData("World", "WORLD")]
[InlineData("", "")]
[InlineData("123", "123")]
public void ToUpper_ReturnsUppercaseString(string input, string expected)
{
    var result = input.ToUpperInvariant();
    Assert.Equal(expected, result);
}
```

**[Theory] with MemberData - Complex Objects:**

```csharp
// MemberData - for complex test data or many test cases
public class DiscountCalculatorTests
{
    public static IEnumerable<object[]> DiscountTestCases()
    {
        // Regular customer, no discount
        yield return new object[]
        {
            new Customer { Type = CustomerType.Regular, TotalPurchases = 100 },
            100m,  // order total
            100m   // expected after discount
        };

        // Premium customer, 10% discount
        yield return new object[]
        {
            new Customer { Type = CustomerType.Premium, TotalPurchases = 1000 },
            100m,
            90m
        };

        // VIP customer, 20% discount
        yield return new object[]
        {
            new Customer { Type = CustomerType.VIP, TotalPurchases = 10000 },
            100m,
            80m
        };
    }

    [Theory]
    [MemberData(nameof(DiscountTestCases))]
    public void CalculateDiscount_ReturnsCorrectAmount(
        Customer customer,
        decimal orderTotal,
        decimal expectedTotal)
    {
        var calculator = new DiscountCalculator();
        var result = calculator.Calculate(customer, orderTotal);
        Assert.Equal(expectedTotal, result);
    }
}
```

**[Theory] with ClassData - Reusable Test Data:**

```csharp
// ClassData - for reusable test data across multiple test classes
public class ValidEmailTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { "user@example.com", true };
        yield return new object[] { "user.name@example.co.uk", true };
        yield return new object[] { "user+tag@example.com", true };
        yield return new object[] { "invalid", false };
        yield return new object[] { "@example.com", false };
        yield return new object[] { "user@", false };
        yield return new object[] { "", false };
        yield return new object[] { null, false };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(ValidEmailTestData))]
public void IsValidEmail_WithVariousInputs_ReturnsExpected(string email, bool expected)
{
    var result = EmailValidator.IsValid(email);
    Assert.Equal(expected, result);
}
```

**When to Choose Each:**

| Use Case | Attribute | Why |
|----------|-----------|-----|
| Single specific scenario | [Fact] | One clear test case |
| Testing edge cases | [Theory] | Multiple inputs, same logic |
| Validation with many inputs | [Theory] + InlineData | Quick to write |
| Complex objects as input | [Theory] + MemberData | Supports any type |
| Shared test data | [Theory] + ClassData | Reusable across tests |
| Async single test | [Fact] | Just return Task |
| Async with data | [Theory] | Same, with parameters |

### Communication Tactics

🎯 **Structure your answer**: Define both, show simple examples first, then progress to MemberData and ClassData. End with when-to-use decision table.

💡 **Emphasize**: [Theory] reduces code duplication - one test method, many scenarios. But don't overuse - if scenarios need different assertions, use [Fact].

⚠️ **Avoid**: Don't forget about async tests. Show that both work with async/await seamlessly.

---

## Question 3: Mocking with Moq

### The Question
> "Show me how you would mock a repository dependency using Moq. Can you also show how to verify that certain methods were called?"

### Key Points to Cover
- Basic mock setup
- Setup with specific arguments
- ReturnsAsync for async methods
- Verify method calls
- Different verification modes

### Detailed Answer

**Basic Mock Setup:**

```csharp
// Interface we're mocking
public interface IOrderRepository
{
    Task<Order> GetByIdAsync(int id);
    Task<List<Order>> GetByCustomerAsync(int customerId);
    Task AddAsync(Order order);
    Task UpdateAsync(Order order);
    Task DeleteAsync(int id);
}

// Basic test setup
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _mockRepository;
    private readonly Mock<ILogger<OrderService>> _mockLogger;
    private readonly OrderService _service;

    public OrderServiceTests()
    {
        _mockRepository = new Mock<IOrderRepository>();
        _mockLogger = new Mock<ILogger<OrderService>>();
        _service = new OrderService(_mockRepository.Object, _mockLogger.Object);
    }
}
```

**Setup - Returning Values:**

```csharp
[Fact]
public async Task GetOrder_WhenExists_ReturnsOrder()
{
    // Arrange - Setup mock to return specific value
    var expectedOrder = new Order { Id = 1, Total = 100 };

    _mockRepository
        .Setup(r => r.GetByIdAsync(1))
        .ReturnsAsync(expectedOrder);

    // Act
    var result = await _service.GetOrderAsync(1);

    // Assert
    Assert.NotNull(result);
    Assert.Equal(1, result.Id);
    Assert.Equal(100, result.Total);
}

[Fact]
public async Task GetOrder_WhenNotExists_ReturnsNull()
{
    // Arrange - Setup to return null
    _mockRepository
        .Setup(r => r.GetByIdAsync(It.IsAny<int>()))
        .ReturnsAsync((Order)null);

    // Act
    var result = await _service.GetOrderAsync(999);

    // Assert
    Assert.Null(result);
}
```

**Setup - With Argument Matching:**

```csharp
[Fact]
public async Task GetOrdersByCustomer_ReturnsFilteredOrders()
{
    // Arrange - Match specific argument
    var customer1Orders = new List<Order>
    {
        new Order { Id = 1, CustomerId = 1 },
        new Order { Id = 2, CustomerId = 1 }
    };

    _mockRepository
        .Setup(r => r.GetByCustomerAsync(1))  // Only when customerId is 1
        .ReturnsAsync(customer1Orders);

    _mockRepository
        .Setup(r => r.GetByCustomerAsync(2))  // Different return for customerId 2
        .ReturnsAsync(new List<Order>());

    // Act
    var result = await _service.GetOrdersByCustomerAsync(1);

    // Assert
    Assert.Equal(2, result.Count);
}

// Using It.Is<T> for complex matching
[Fact]
public async Task GetOrders_WithMinimumTotal_ReturnsFilteredOrders()
{
    _mockRepository
        .Setup(r => r.GetByIdAsync(It.Is<int>(id => id > 0)))
        .ReturnsAsync(new Order { Id = 1 });

    _mockRepository
        .Setup(r => r.GetByIdAsync(It.Is<int>(id => id <= 0)))
        .ThrowsAsync(new ArgumentException("Invalid ID"));
}
```

**Setup - Throwing Exceptions:**

```csharp
[Fact]
public async Task GetOrder_WhenRepositoryThrows_PropagatesException()
{
    // Arrange
    _mockRepository
        .Setup(r => r.GetByIdAsync(It.IsAny<int>()))
        .ThrowsAsync(new DatabaseException("Connection failed"));

    // Act & Assert
    await Assert.ThrowsAsync<DatabaseException>(
        () => _service.GetOrderAsync(1));
}
```

**Verify - Ensuring Methods Were Called:**

```csharp
[Fact]
public async Task CreateOrder_SavesOrderToRepository()
{
    // Arrange
    var newOrder = new Order { CustomerId = 1, Total = 100 };

    _mockRepository
        .Setup(r => r.AddAsync(It.IsAny<Order>()))
        .Returns(Task.CompletedTask);

    // Act
    await _service.CreateOrderAsync(newOrder);

    // Assert - Verify AddAsync was called exactly once
    _mockRepository.Verify(
        r => r.AddAsync(It.IsAny<Order>()),
        Times.Once);
}

[Fact]
public async Task CreateOrder_SavesOrderWithCorrectData()
{
    // Arrange
    Order capturedOrder = null;

    _mockRepository
        .Setup(r => r.AddAsync(It.IsAny<Order>()))
        .Callback<Order>(order => capturedOrder = order)  // Capture the argument
        .Returns(Task.CompletedTask);

    // Act
    await _service.CreateOrderAsync(new CreateOrderRequest
    {
        CustomerId = 1,
        Items = new[] { new OrderItem { ProductId = 1, Quantity = 2 } }
    });

    // Assert - Verify the captured order has correct data
    Assert.NotNull(capturedOrder);
    Assert.Equal(1, capturedOrder.CustomerId);
    Assert.Single(capturedOrder.Items);
}

[Fact]
public async Task ProcessOrder_WhenInvalid_DoesNotSave()
{
    // Arrange
    var invalidOrder = new Order { Total = -100 };  // Invalid

    // Act
    await Assert.ThrowsAsync<ValidationException>(
        () => _service.ProcessOrderAsync(invalidOrder));

    // Assert - Verify AddAsync was NEVER called
    _mockRepository.Verify(
        r => r.AddAsync(It.IsAny<Order>()),
        Times.Never);
}
```

**Verify - Different Times Options:**

```csharp
// Times options
_mockRepository.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Never);
_mockRepository.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Once);
_mockRepository.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Exactly(3));
_mockRepository.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.AtLeastOnce);
_mockRepository.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.AtMost(5));
_mockRepository.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Between(2, 4, Range.Inclusive));
```

**Complete Test Example:**

```csharp
[Fact]
public async Task ProcessPayment_WhenSuccessful_UpdatesOrderStatus()
{
    // Arrange
    var order = new Order { Id = 1, Status = OrderStatus.Pending, Total = 100 };
    var paymentResult = new PaymentResult { Success = true, TransactionId = "TXN123" };

    var mockRepo = new Mock<IOrderRepository>();
    var mockPayment = new Mock<IPaymentGateway>();

    mockRepo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(order);
    mockRepo.Setup(r => r.UpdateAsync(It.IsAny<Order>())).Returns(Task.CompletedTask);
    mockPayment.Setup(p => p.ChargeAsync(100)).ReturnsAsync(paymentResult);

    var service = new OrderService(mockRepo.Object, mockPayment.Object);

    // Act
    var result = await service.ProcessPaymentAsync(1);

    // Assert
    Assert.True(result.Success);

    // Verify payment was charged
    mockPayment.Verify(p => p.ChargeAsync(100), Times.Once);

    // Verify order was updated with correct status
    mockRepo.Verify(r => r.UpdateAsync(It.Is<Order>(o =>
        o.Id == 1 &&
        o.Status == OrderStatus.Paid &&
        o.TransactionId == "TXN123"
    )), Times.Once);
}
```

### Communication Tactics

🎯 **Structure your answer**: Start with basic Setup/ReturnsAsync, then show argument matching with It.Is, then demonstrate Verify with Times.

💡 **Emphasize**: Mocking isolates the unit under test. We're not testing the repository - we're testing how the service USES the repository.

⚠️ **Avoid**: Don't mock everything. Only mock external dependencies. Don't mock value objects or simple classes.

---

## Question 4: Testing Pyramid

### The Question
> "Explain the testing pyramid. Where does each type of test fit and how many of each should you have?"

### Key Points to Cover
- Three levels of the pyramid
- Characteristics of each level
- Cost/speed trade-offs
- Practical ratios

### Detailed Answer

**The Testing Pyramid:**

```
                    /\
                   /  \
                  / E2E\           Few, Slow, Expensive
                 /------\
                /        \
               /Integration\       Some, Medium Speed
              /--------------\
             /                \
            /    Unit Tests    \   Many, Fast, Cheap
           /____________________\
```

**1. Unit Tests (Base of Pyramid):**

```csharp
// CHARACTERISTICS:
// - Test single units in isolation
// - Fast (milliseconds)
// - No external dependencies
// - Many of these (70-80% of tests)

[Fact]
public void CalculateDiscount_PremiumCustomer_Returns10Percent()
{
    // Arrange
    var calculator = new DiscountCalculator();
    var customer = new Customer { Type = CustomerType.Premium };

    // Act
    var discount = calculator.Calculate(customer, 100);

    // Assert
    Assert.Equal(10m, discount);
}

// What to unit test:
// - Business logic
// - Calculations
// - Validation rules
// - Domain models
// - Pure functions
```

**2. Integration Tests (Middle of Pyramid):**

```csharp
// CHARACTERISTICS:
// - Test how components work together
// - Slower (seconds)
// - May use real database, HTTP
// - Some of these (15-20% of tests)

public class OrderApiIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrderApiIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task CreateOrder_WithValidData_Returns201()
    {
        // Arrange
        var order = new { CustomerId = 1, Items = new[] { new { ProductId = 1, Quantity = 2 } } };

        // Act
        var response = await _client.PostAsJsonAsync("/api/orders", order);

        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);

        var created = await response.Content.ReadFromJsonAsync<OrderDto>();
        Assert.NotNull(created);
        Assert.True(created.Id > 0);
    }

    [Fact]
    public async Task GetOrder_WhenExists_ReturnsOrder()
    {
        // This actually hits the API and database
        var response = await _client.GetAsync("/api/orders/1");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}

// What to integration test:
// - API endpoints
// - Database queries (EF Core)
// - External service integrations
// - Message queue consumers
// - Middleware pipelines
```

**3. End-to-End Tests (Top of Pyramid):**

```csharp
// CHARACTERISTICS:
// - Test complete user workflows
// - Slowest (minutes)
// - Most brittle/expensive
// - Few of these (5-10% of tests)

// Using Playwright for E2E
public class CheckoutE2ETests
{
    [Fact]
    public async Task CompleteCheckout_FromCartToConfirmation()
    {
        using var playwright = await Playwright.CreateAsync();
        var browser = await playwright.Chromium.LaunchAsync();
        var page = await browser.NewPageAsync();

        // Navigate to site
        await page.GotoAsync("https://localhost:5001");

        // Add item to cart
        await page.ClickAsync("[data-testid='product-1-add-to-cart']");

        // Go to checkout
        await page.ClickAsync("[data-testid='checkout-button']");

        // Fill payment details
        await page.FillAsync("[data-testid='card-number']", "4111111111111111");
        await page.FillAsync("[data-testid='expiry']", "12/25");
        await page.FillAsync("[data-testid='cvv']", "123");

        // Submit order
        await page.ClickAsync("[data-testid='place-order']");

        // Verify confirmation
        await page.WaitForSelectorAsync("[data-testid='order-confirmation']");
        var confirmationText = await page.TextContentAsync("[data-testid='order-confirmation']");

        Assert.Contains("Thank you for your order", confirmationText);
    }
}

// What to E2E test:
// - Critical user journeys
// - Checkout flows
// - Login/registration
// - Key business processes
```

**Test Ratio Guidelines:**

```
Recommended Ratio (Google's approach):
- Unit Tests:        70%
- Integration Tests: 20%
- E2E Tests:        10%

For a project with 100 tests:
- 70 unit tests (run in < 1 second total)
- 20 integration tests (run in ~30 seconds)
- 10 E2E tests (run in ~5 minutes)
```

**Trade-offs:**

| Aspect | Unit | Integration | E2E |
|--------|------|-------------|-----|
| Speed | Milliseconds | Seconds | Minutes |
| Cost to write | Low | Medium | High |
| Maintenance | Low | Medium | High |
| Confidence | Lower | Medium | Higher |
| Isolation | Complete | Partial | None |
| Flakiness | Rare | Sometimes | Common |

### Communication Tactics

🎯 **Structure your answer**: Draw/describe the pyramid, explain each level with examples, then discuss the ratios and trade-offs.

💡 **Emphasize**: The pyramid shape is intentional - more fast/cheap tests at the bottom, fewer slow/expensive tests at the top. Don't invert it!

⚠️ **Avoid**: Don't say "unit tests are enough." Integration and E2E tests catch different bugs. The pyramid is about BALANCE.

---

## Question 5: Testing Time and External Dependencies

### The Question
> "How do you test code that depends on DateTime.Now or calls external APIs? Show me techniques for both."

### Key Points to Cover
- Why DateTime.Now is problematic
- TimeProvider abstraction (.NET 8)
- Mocking HTTP clients
- Test doubles for external services

### Detailed Answer

**Problem with DateTime.Now:**

```csharp
// ❌ UNTESTABLE - How do you test expiration logic?
public class PromotionService
{
    public bool IsPromotionActive(Promotion promotion)
    {
        var now = DateTime.Now;  // Hardcoded dependency!
        return now >= promotion.StartDate && now <= promotion.EndDate;
    }
}

// Can't test:
// - What happens on the last day of promotion?
// - What happens after midnight?
// - Time zone issues?
```

**Solution 1: ITimeProvider Interface (Pre .NET 8):**

```csharp
// Create an abstraction
public interface ITimeProvider
{
    DateTime Now { get; }
    DateTime UtcNow { get; }
}

// Production implementation
public class SystemTimeProvider : ITimeProvider
{
    public DateTime Now => DateTime.Now;
    public DateTime UtcNow => DateTime.UtcNow;
}

// Test implementation
public class FakeTimeProvider : ITimeProvider
{
    private DateTime _now;

    public FakeTimeProvider(DateTime now) => _now = now;

    public DateTime Now => _now;
    public DateTime UtcNow => _now.ToUniversalTime();

    public void Advance(TimeSpan duration) => _now = _now.Add(duration);
}

// Refactored service
public class PromotionService
{
    private readonly ITimeProvider _timeProvider;

    public PromotionService(ITimeProvider timeProvider)
    {
        _timeProvider = timeProvider;
    }

    public bool IsPromotionActive(Promotion promotion)
    {
        var now = _timeProvider.Now;
        return now >= promotion.StartDate && now <= promotion.EndDate;
    }
}

// Now testable!
[Fact]
public void IsPromotionActive_WhenWithinDateRange_ReturnsTrue()
{
    // Arrange - Control time!
    var fakeTime = new FakeTimeProvider(new DateTime(2024, 6, 15));
    var service = new PromotionService(fakeTime);
    var promotion = new Promotion
    {
        StartDate = new DateTime(2024, 6, 1),
        EndDate = new DateTime(2024, 6, 30)
    };

    // Act
    var result = service.IsPromotionActive(promotion);

    // Assert
    Assert.True(result);
}

[Fact]
public void IsPromotionActive_WhenAfterEndDate_ReturnsFalse()
{
    // Arrange
    var fakeTime = new FakeTimeProvider(new DateTime(2024, 7, 1));  // After promotion
    var service = new PromotionService(fakeTime);
    var promotion = new Promotion
    {
        StartDate = new DateTime(2024, 6, 1),
        EndDate = new DateTime(2024, 6, 30)
    };

    // Act
    var result = service.IsPromotionActive(promotion);

    // Assert
    Assert.False(result);
}
```

**Solution 2: TimeProvider (.NET 8+):**

```csharp
// .NET 8 provides built-in TimeProvider
public class PromotionService
{
    private readonly TimeProvider _timeProvider;

    public PromotionService(TimeProvider timeProvider)
    {
        _timeProvider = timeProvider;
    }

    public bool IsPromotionActive(Promotion promotion)
    {
        var now = _timeProvider.GetUtcNow();
        return now >= promotion.StartDate && now <= promotion.EndDate;
    }
}

// In production
services.AddSingleton(TimeProvider.System);

// In tests
[Fact]
public void IsPromotionActive_Test()
{
    var fakeTime = new FakeTimeProvider();
    fakeTime.SetUtcNow(new DateTimeOffset(2024, 6, 15, 0, 0, 0, TimeSpan.Zero));

    var service = new PromotionService(fakeTime);
    // ... test logic
}
```

**Testing External API Dependencies:**

```csharp
// Interface for external service
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(string cardToken, decimal amount);
}

// Production implementation
public class StripePaymentGateway : IPaymentGateway
{
    private readonly HttpClient _httpClient;

    public async Task<PaymentResult> ChargeAsync(string cardToken, decimal amount)
    {
        var response = await _httpClient.PostAsync("https://api.stripe.com/v1/charges", ...);
        // ... handle response
    }
}

// Test with mock
[Fact]
public async Task ProcessPayment_WhenGatewaySucceeds_ReturnsSuccess()
{
    // Arrange
    var mockGateway = new Mock<IPaymentGateway>();
    mockGateway
        .Setup(g => g.ChargeAsync("tok_123", 100))
        .ReturnsAsync(new PaymentResult { Success = true, TransactionId = "txn_123" });

    var service = new PaymentService(mockGateway.Object);

    // Act
    var result = await service.ProcessPaymentAsync("tok_123", 100);

    // Assert
    Assert.True(result.Success);
}
```

**Testing HttpClient with HttpMessageHandler:**

```csharp
// For when you need to test the actual HTTP client code
public class MockHttpMessageHandler : HttpMessageHandler
{
    private readonly HttpStatusCode _statusCode;
    private readonly string _responseContent;

    public MockHttpMessageHandler(HttpStatusCode statusCode, string responseContent)
    {
        _statusCode = statusCode;
        _responseContent = responseContent;
    }

    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        return Task.FromResult(new HttpResponseMessage
        {
            StatusCode = _statusCode,
            Content = new StringContent(_responseContent)
        });
    }
}

[Fact]
public async Task GetWeather_ReturnsWeatherData()
{
    // Arrange
    var mockHandler = new MockHttpMessageHandler(
        HttpStatusCode.OK,
        """{"temperature": 25, "condition": "sunny"}""");

    var httpClient = new HttpClient(mockHandler)
    {
        BaseAddress = new Uri("https://api.weather.com")
    };

    var service = new WeatherService(httpClient);

    // Act
    var weather = await service.GetCurrentWeatherAsync("Istanbul");

    // Assert
    Assert.Equal(25, weather.Temperature);
    Assert.Equal("sunny", weather.Condition);
}
```

**Using WireMock for HTTP Mocking:**

```csharp
// WireMock.Net - more powerful HTTP mocking
public class PaymentGatewayIntegrationTests : IDisposable
{
    private readonly WireMockServer _mockServer;
    private readonly PaymentGateway _gateway;

    public PaymentGatewayIntegrationTests()
    {
        _mockServer = WireMockServer.Start();
        _gateway = new PaymentGateway(new HttpClient
        {
            BaseAddress = new Uri(_mockServer.Urls[0])
        });
    }

    [Fact]
    public async Task Charge_WhenSuccessful_ReturnsTransactionId()
    {
        // Arrange - Setup mock server
        _mockServer
            .Given(Request.Create()
                .WithPath("/v1/charges")
                .UsingPost())
            .RespondWith(Response.Create()
                .WithStatusCode(200)
                .WithBody("""{"id": "txn_123", "status": "succeeded"}"""));

        // Act
        var result = await _gateway.ChargeAsync("tok_123", 100);

        // Assert
        Assert.Equal("txn_123", result.TransactionId);
    }

    public void Dispose() => _mockServer.Stop();
}
```

### Communication Tactics

🎯 **Structure your answer**: Start with WHY these dependencies are problematic, then show abstraction patterns, then demonstrate testing techniques.

💡 **Emphasize**: The key is dependency injection - inject time/HTTP clients so they can be replaced in tests. Mention .NET 8's built-in TimeProvider.

⚠️ **Avoid**: Don't suggest manipulating system time globally. That affects other tests and causes flaky tests.

---

## Quick Review - Key Takeaways

| Question | Key Point |
|----------|-----------|
| Why Test? | Bugs cheaper to fix early, enables safe refactoring |
| [Fact] vs [Theory] | Fact = one scenario, Theory = data-driven |
| Moq | Setup returns, Verify calls, use It.Is for matching |
| Testing Pyramid | 70% unit, 20% integration, 10% E2E |
| Time/External APIs | Inject abstractions, use TimeProvider in .NET 8 |
