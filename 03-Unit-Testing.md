# Unit Testing

> ⚠️ **HIGH PRIORITY**: Be ready to articulate WHY unit testing is necessary

---

## Why Unit Testing is Necessary

### The Business Case

| Benefit | Explanation |
|---------|-------------|
| **Catches bugs early** | Bugs found in development cost 10x less to fix than bugs in production |
| **Enables refactoring** | Tests act as a safety net when changing code |
| **Documents behavior** | Tests show how code is supposed to work |
| **Reduces debugging time** | Failed tests pinpoint exactly what broke |
| **Increases confidence** | Deploy with assurance that core functionality works |
| **Faster development** | Counterintuitive but true—less time debugging means faster delivery |

### Interview Answer: "Why is unit testing necessary?"

> "Unit testing is essential for several reasons:
> 1. **Risk mitigation**: Tests catch regressions before they reach production, where fixing bugs is exponentially more expensive—especially in fintech where a bug could mean financial loss.
> 2. **Refactoring safety**: When I need to improve or change code, tests tell me immediately if I've broken existing functionality.
> 3. **Living documentation**: Tests demonstrate how components should behave, which is especially valuable for new team members.
> 4. **Design feedback**: Code that's hard to test often has design problems. Writing tests forces better separation of concerns.
> 5. **Development speed**: Although writing tests takes time upfront, it significantly reduces debugging time and production incidents."

---

## The Testing Pyramid

```
                    /\
                   /  \
                  / E2E \          Few - Slow - Expensive
                 /------\
                /        \
               /Integration\       Some - Medium
              /--------------\
             /                \
            /    Unit Tests    \   Many - Fast - Cheap
           /____________________\
```

| Layer | Count | Speed | Cost | Purpose |
|-------|-------|-------|------|---------|
| **Unit** | Many | Fast (ms) | Low | Test individual components in isolation |
| **Integration** | Some | Medium (s) | Medium | Test component interactions |
| **E2E** | Few | Slow (min) | High | Test complete user flows |

### What Makes a Good Unit Test

- **Fast**: Should run in milliseconds
- **Isolated**: No dependencies on database, file system, network
- **Repeatable**: Same result every time
- **Self-validating**: Pass or fail, no manual interpretation
- **Timely**: Written close to when the code is written

---

## xUnit Basics in .NET

### Setup

```bash
dotnet add package xUnit
dotnet add package xUnit.runner.visualstudio
dotnet add package Microsoft.NET.Test.Sdk
```

### Test Structure - Arrange, Act, Assert

```csharp
public class CalculatorTests
{
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        // Arrange - Set up test data and dependencies
        var calculator = new Calculator();

        // Act - Execute the code being tested
        var result = calculator.Add(2, 3);

        // Assert - Verify the expected outcome
        Assert.Equal(5, result);
    }
}
```

### Fact vs Theory

```csharp
public class PaymentValidatorTests
{
    // [Fact] - Single test case
    [Fact]
    public void Validate_NegativeAmount_ReturnsFalse()
    {
        var validator = new PaymentValidator();
        var result = validator.Validate(-100);
        Assert.False(result);
    }

    // [Theory] - Parameterized tests with multiple inputs
    [Theory]
    [InlineData(0, false)]         // Zero is invalid
    [InlineData(-1, false)]        // Negative is invalid
    [InlineData(0.01, true)]       // Minimum valid
    [InlineData(1000, true)]       // Normal amount
    [InlineData(1000000, true)]    // Large amount
    public void Validate_VariousAmounts_ReturnsExpectedResult(
        decimal amount,
        bool expected)
    {
        var validator = new PaymentValidator();
        var result = validator.Validate(amount);
        Assert.Equal(expected, result);
    }

    // [Theory] with complex data
    [Theory]
    [MemberData(nameof(GetTestPayments))]
    public void ValidatePayment_VariousPayments_ReturnsExpectedResult(
        Payment payment,
        bool expected)
    {
        var validator = new PaymentValidator();
        var result = validator.ValidatePayment(payment);
        Assert.Equal(expected, result);
    }

    public static IEnumerable<object[]> GetTestPayments()
    {
        yield return new object[]
        {
            new Payment { Amount = 100, Currency = "TRY" },
            true
        };
        yield return new object[]
        {
            new Payment { Amount = 100, Currency = null },
            false
        };
        yield return new object[]
        {
            new Payment { Amount = -50, Currency = "TRY" },
            false
        };
    }
}
```

### Common Assertions

```csharp
public class AssertionExamples
{
    [Fact]
    public void DemonstrateAssertions()
    {
        // Equality
        Assert.Equal(expected, actual);
        Assert.NotEqual(unexpected, actual);

        // Boolean
        Assert.True(condition);
        Assert.False(condition);

        // Null checks
        Assert.Null(obj);
        Assert.NotNull(obj);

        // Collections
        Assert.Empty(collection);
        Assert.NotEmpty(collection);
        Assert.Contains(item, collection);
        Assert.DoesNotContain(item, collection);
        Assert.Single(collection);           // Exactly one item
        Assert.All(collection, item => Assert.True(item > 0));

        // Type checks
        Assert.IsType<ExpectedType>(obj);
        Assert.IsAssignableFrom<BaseType>(obj);

        // String checks
        Assert.StartsWith("prefix", text);
        Assert.EndsWith("suffix", text);
        Assert.Contains("substring", text);
        Assert.Matches("regex pattern", text);

        // Range
        Assert.InRange(value, low, high);

        // Exceptions
        Assert.Throws<ArgumentException>(() => MethodThatThrows());
        var ex = Assert.Throws<CustomException>(() => MethodThatThrows());
        Assert.Equal("Expected message", ex.Message);

        // Async exceptions
        await Assert.ThrowsAsync<InvalidOperationException>(
            async () => await AsyncMethodThatThrows());
    }
}
```

### Test Lifecycle and Setup

```csharp
// Per-test setup with constructor/dispose
public class OrderServiceTests : IDisposable
{
    private readonly OrderService _service;
    private readonly Mock<IOrderRepository> _mockRepo;

    // Runs before EACH test
    public OrderServiceTests()
    {
        _mockRepo = new Mock<IOrderRepository>();
        _service = new OrderService(_mockRepo.Object);
    }

    // Runs after EACH test
    public void Dispose()
    {
        // Cleanup if needed
    }

    [Fact]
    public void Test1() { /* uses fresh _service */ }

    [Fact]
    public void Test2() { /* uses fresh _service */ }
}

// Shared setup with IClassFixture (expensive resources)
public class DatabaseFixture : IDisposable
{
    public IDbConnection Connection { get; }

    public DatabaseFixture()
    {
        // Runs once before all tests in the class
        Connection = new SqliteConnection("DataSource=:memory:");
        Connection.Open();
    }

    public void Dispose()
    {
        // Runs once after all tests complete
        Connection.Dispose();
    }
}

public class DatabaseTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public DatabaseTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public void Test1() { /* uses shared _fixture.Connection */ }
}
```

---

## Mocking with Moq

### Setup

```bash
dotnet add package Moq
```

### Basic Mocking

```csharp
public interface IPaymentGateway
{
    Task<PaymentResult> ProcessAsync(PaymentRequest request);
    Task<RefundResult> RefundAsync(string transactionId);
    bool IsAvailable { get; }
}

public class PaymentServiceTests
{
    [Fact]
    public async Task ProcessPayment_ValidRequest_ReturnsSuccess()
    {
        // Arrange
        var mockGateway = new Mock<IPaymentGateway>();

        // Setup method to return specific value
        mockGateway
            .Setup(g => g.ProcessAsync(It.IsAny<PaymentRequest>()))
            .ReturnsAsync(new PaymentResult { Success = true, TransactionId = "TXN123" });

        // Setup property
        mockGateway
            .Setup(g => g.IsAvailable)
            .Returns(true);

        var service = new PaymentService(mockGateway.Object);
        var request = new PaymentRequest { Amount = 100 };

        // Act
        var result = await service.ProcessPaymentAsync(request);

        // Assert
        Assert.True(result.Success);
        Assert.Equal("TXN123", result.TransactionId);
    }
}
```

### Argument Matching

```csharp
[Fact]
public async Task ProcessPayment_ArgumentMatching()
{
    var mock = new Mock<IPaymentGateway>();

    // Match ANY value of type
    mock.Setup(g => g.ProcessAsync(It.IsAny<PaymentRequest>()))
        .ReturnsAsync(new PaymentResult { Success = true });

    // Match specific value
    mock.Setup(g => g.RefundAsync("TXN123"))
        .ReturnsAsync(new RefundResult { Success = true });

    // Match with predicate
    mock.Setup(g => g.ProcessAsync(It.Is<PaymentRequest>(r => r.Amount > 0)))
        .ReturnsAsync(new PaymentResult { Success = true });

    // Match with regex
    mock.Setup(g => g.RefundAsync(It.IsRegex("^TXN\\d+$")))
        .ReturnsAsync(new RefundResult { Success = true });
}
```

### Verifying Calls

```csharp
[Fact]
public async Task ProcessPayment_CallsGatewayOnce()
{
    var mock = new Mock<IPaymentGateway>();
    mock.Setup(g => g.ProcessAsync(It.IsAny<PaymentRequest>()))
        .ReturnsAsync(new PaymentResult { Success = true });

    var service = new PaymentService(mock.Object);
    await service.ProcessPaymentAsync(new PaymentRequest { Amount = 100 });

    // Verify method was called
    mock.Verify(g => g.ProcessAsync(It.IsAny<PaymentRequest>()), Times.Once);

    // Verify with specific arguments
    mock.Verify(g => g.ProcessAsync(It.Is<PaymentRequest>(r => r.Amount == 100)));

    // Verify never called
    mock.Verify(g => g.RefundAsync(It.IsAny<string>()), Times.Never);

    // Times options
    // Times.Once()
    // Times.Never()
    // Times.Exactly(n)
    // Times.AtLeast(n)
    // Times.AtMost(n)
    // Times.Between(min, max, Range.Inclusive)
}
```

### Throwing Exceptions

```csharp
[Fact]
public async Task ProcessPayment_GatewayFails_ThrowsException()
{
    var mock = new Mock<IPaymentGateway>();

    // Setup to throw exception
    mock.Setup(g => g.ProcessAsync(It.IsAny<PaymentRequest>()))
        .ThrowsAsync(new PaymentGatewayException("Gateway unavailable"));

    var service = new PaymentService(mock.Object);

    // Assert exception is thrown
    await Assert.ThrowsAsync<PaymentGatewayException>(
        () => service.ProcessPaymentAsync(new PaymentRequest()));
}
```

### Sequential Returns

```csharp
[Fact]
public async Task RetryLogic_ReturnsSuccessOnSecondAttempt()
{
    var mock = new Mock<IPaymentGateway>();

    // First call fails, second succeeds
    mock.SetupSequence(g => g.ProcessAsync(It.IsAny<PaymentRequest>()))
        .ThrowsAsync(new TimeoutException())  // First call
        .ReturnsAsync(new PaymentResult { Success = true }); // Second call

    var service = new PaymentServiceWithRetry(mock.Object);
    var result = await service.ProcessWithRetryAsync(new PaymentRequest());

    Assert.True(result.Success);
    mock.Verify(g => g.ProcessAsync(It.IsAny<PaymentRequest>()), Times.Exactly(2));
}
```

### Callback for Side Effects

```csharp
[Fact]
public async Task ProcessPayment_LogsRequest()
{
    var mock = new Mock<IPaymentGateway>();
    PaymentRequest capturedRequest = null;

    mock.Setup(g => g.ProcessAsync(It.IsAny<PaymentRequest>()))
        .Callback<PaymentRequest>(req => capturedRequest = req)  // Capture argument
        .ReturnsAsync(new PaymentResult { Success = true });

    var service = new PaymentService(mock.Object);
    await service.ProcessPaymentAsync(new PaymentRequest { Amount = 500 });

    // Verify captured request
    Assert.NotNull(capturedRequest);
    Assert.Equal(500, capturedRequest.Amount);
}
```

---

## Testing Best Practices

### Naming Convention

```csharp
// Pattern: MethodName_Scenario_ExpectedResult
public class OrderServiceTests
{
    [Fact]
    public void CreateOrder_WithValidData_ReturnsSuccessResult() { }

    [Fact]
    public void CreateOrder_WithNullCustomer_ThrowsArgumentNullException() { }

    [Fact]
    public void CreateOrder_WithZeroQuantity_ReturnsValidationError() { }

    [Fact]
    public async Task ProcessPayment_WhenGatewayTimeout_RetriesThreeTimes() { }
}
```

### One Assert Per Test (Generally)

```csharp
// ❌ BAD: Multiple unrelated assertions
[Fact]
public void CreateOrder_ValidatesEverything()
{
    var order = _service.CreateOrder(validData);

    Assert.NotNull(order);
    Assert.Equal("Pending", order.Status);
    Assert.True(order.CreatedAt > DateTime.MinValue);
    Assert.NotEmpty(order.OrderNumber);
    Assert.Single(order.Items);
    // If any fails, you don't know which without running debugger
}

// ✅ GOOD: Focused assertions (or logically grouped)
[Fact]
public void CreateOrder_ValidData_SetsStatusToPending()
{
    var order = _service.CreateOrder(validData);
    Assert.Equal("Pending", order.Status);
}

[Fact]
public void CreateOrder_ValidData_GeneratesOrderNumber()
{
    var order = _service.CreateOrder(validData);
    Assert.NotEmpty(order.OrderNumber);
}

// ✅ ALSO OK: Related assertions about same behavior
[Fact]
public void CreateOrder_ValidData_ReturnsFullyInitializedOrder()
{
    var order = _service.CreateOrder(validData);

    Assert.NotNull(order);
    Assert.NotEmpty(order.OrderNumber);
    Assert.True(order.CreatedAt > DateTime.MinValue);
    // These are all about "order is properly initialized"
}
```

### Test Data Builders

```csharp
public class PaymentRequestBuilder
{
    private decimal _amount = 100m;
    private string _currency = "TRY";
    private string _customerId = "CUST001";

    public PaymentRequestBuilder WithAmount(decimal amount)
    {
        _amount = amount;
        return this;
    }

    public PaymentRequestBuilder WithCurrency(string currency)
    {
        _currency = currency;
        return this;
    }

    public PaymentRequestBuilder WithCustomerId(string customerId)
    {
        _customerId = customerId;
        return this;
    }

    public PaymentRequest Build()
    {
        return new PaymentRequest
        {
            Amount = _amount,
            Currency = _currency,
            CustomerId = _customerId
        };
    }
}

// Usage in tests
[Fact]
public void ProcessPayment_HighValue_RequiresAdditionalVerification()
{
    var request = new PaymentRequestBuilder()
        .WithAmount(50000)
        .Build();

    var result = _service.Process(request);

    Assert.True(result.RequiresVerification);
}
```

### Testing Async Code

```csharp
public class AsyncTests
{
    [Fact]
    public async Task ProcessAsync_ValidData_ReturnsSuccess()
    {
        // Use async/await properly
        var result = await _service.ProcessAsync(data);
        Assert.True(result.Success);
    }

    [Fact]
    public async Task ProcessAsync_Cancellation_ThrowsOperationCanceled()
    {
        var cts = new CancellationTokenSource();
        cts.Cancel();

        await Assert.ThrowsAsync<OperationCanceledException>(
            () => _service.ProcessAsync(data, cts.Token));
    }
}
```

---

## TDD Basics (Test-Driven Development)

### The Red-Green-Refactor Cycle

1. **RED**: Write a failing test first
2. **GREEN**: Write minimum code to make the test pass
3. **REFACTOR**: Improve the code while keeping tests green

### TDD Example

```csharp
// Step 1: RED - Write failing test
[Fact]
public void Calculate_TenPercentDiscount_ReturnsDiscountedPrice()
{
    var calculator = new PriceCalculator();
    var result = calculator.ApplyDiscount(100, 10);
    Assert.Equal(90, result);
}

// Test fails - PriceCalculator doesn't exist

// Step 2: GREEN - Minimum code to pass
public class PriceCalculator
{
    public decimal ApplyDiscount(decimal price, decimal discountPercent)
    {
        return price - (price * discountPercent / 100);
    }
}

// Test passes!

// Step 3: REFACTOR - Improve while tests stay green
public class PriceCalculator
{
    public decimal ApplyDiscount(decimal price, decimal discountPercent)
    {
        if (price < 0)
            throw new ArgumentException("Price cannot be negative");
        if (discountPercent < 0 || discountPercent > 100)
            throw new ArgumentException("Discount must be between 0 and 100");

        var discountMultiplier = 1 - (discountPercent / 100);
        return price * discountMultiplier;
    }
}

// Add more tests for edge cases (TDD continues)
[Theory]
[InlineData(-1, 10)]
[InlineData(100, -5)]
[InlineData(100, 101)]
public void ApplyDiscount_InvalidInput_ThrowsArgumentException(
    decimal price,
    decimal discount)
{
    var calculator = new PriceCalculator();
    Assert.Throws<ArgumentException>(() => calculator.ApplyDiscount(price, discount));
}
```

---

## Quick Review Points

### Why Unit Testing?
- ✅ Catch bugs early (cheaper to fix)
- ✅ Enable safe refactoring
- ✅ Document expected behavior
- ✅ Force better design (testable = loosely coupled)
- ✅ Faster development (less debugging)

### xUnit Basics
- ✅ `[Fact]` for single test cases
- ✅ `[Theory]` with `[InlineData]` for parameterized tests
- ✅ Arrange-Act-Assert pattern
- ✅ Constructor = setup, IDisposable = teardown

### Moq Basics
- ✅ `new Mock<IInterface>()`
- ✅ `.Setup()` to configure behavior
- ✅ `.Returns()` / `.ReturnsAsync()` for return values
- ✅ `.Throws()` / `.ThrowsAsync()` for exceptions
- ✅ `.Verify()` to assert calls were made
- ✅ `It.IsAny<T>()` for argument matching

### Best Practices
- ✅ Test one thing per test
- ✅ Use descriptive names: `Method_Scenario_Expected`
- ✅ Keep tests independent
- ✅ Don't test framework code
- ✅ Test behavior, not implementation

---

## Interview Q&A

**Q: How do you decide what to test?**
> A: I focus on testing:
> 1. **Business logic** - The core rules that make the system valuable
> 2. **Edge cases** - Boundary conditions, null inputs, empty collections
> 3. **Error paths** - Exception handling, validation failures
> 4. **Public API** - The contract that other code depends on
>
> I avoid testing trivial code (simple getters/setters), framework functionality, or implementation details that might change.

**Q: What's the difference between a mock and a stub?**
> A: A **stub** provides canned answers to calls made during the test—it doesn't verify behavior. A **mock** is a stub that also verifies that certain calls were made. In Moq, `.Setup()` creates stub behavior, `.Verify()` adds mock verification.

**Q: How do you test code that depends on the database?**
> A: For unit tests, I use mocks (Moq) to isolate the code from the database. For integration tests, I use:
> 1. **In-memory database** (EF Core InMemory provider) for fast tests
> 2. **Testcontainers** for realistic database behavior with Docker
> 3. **Transaction rollback** to keep tests isolated
>
> The key is that unit tests never touch real infrastructure—that's what integration tests are for.

**Q: What is test coverage and how important is it?**
> A: Test coverage measures what percentage of code is executed by tests. While it's a useful metric, I don't chase high percentages blindly. 80% coverage with meaningful tests is better than 100% coverage with tests that don't actually verify behavior. I focus on covering critical paths, complex logic, and code that's frequently changed.
