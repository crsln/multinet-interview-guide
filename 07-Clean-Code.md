# Clean Code Principles

> Best practices for readable, maintainable code

---

## Naming Conventions

### General Rules

| Element | Convention | Example |
|---------|-----------|---------|
| Classes | PascalCase, nouns | `OrderService`, `PaymentProcessor` |
| Interfaces | PascalCase with I prefix | `IOrderRepository`, `IPaymentGateway` |
| Methods | PascalCase, verbs | `GetOrder`, `ProcessPayment`, `ValidateRequest` |
| Properties | PascalCase | `FirstName`, `OrderTotal` |
| Private fields | _camelCase | `_orderRepository`, `_logger` |
| Local variables | camelCase | `orderTotal`, `customerName` |
| Constants | PascalCase | `MaxRetryCount`, `DefaultTimeout` |
| Parameters | camelCase | `orderId`, `customerEmail` |
| Async methods | Suffix with Async | `GetOrderAsync`, `SaveAsync` |

### Naming Guidelines

```csharp
// ❌ BAD - Unclear names
public class Data { }
public void Process(string s) { }
public int x;
public bool flag;

// ✅ GOOD - Clear, descriptive names
public class CustomerOrder { }
public void ProcessPayment(string transactionId) { }
public int retryCount;
public bool isProcessed;
```

### Boolean Names

```csharp
// ❌ BAD - Not clear what true/false means
public bool Status;
public bool Check();

// ✅ GOOD - Question-style, clear meaning
public bool IsActive;
public bool HasPermission;
public bool CanProcess();
public bool ShouldRetry();
```

### Method Names

```csharp
// ❌ BAD - Vague or misleading
public void Handle(Order order);      // Handle what?
public Order Get();                   // Get from where?
public void DoStuff();                // What stuff?

// ✅ GOOD - Descriptive verbs
public void ProcessOrder(Order order);
public Order GetOrderById(int id);
public void SendOrderConfirmationEmail(Order order);

// Common verb prefixes:
// Get, Set - Accessors
// Create, Build, Make - Object creation
// Find, Search, Lookup - Queries
// Calculate, Compute - Calculations
// Validate, Check, Verify - Validation
// Send, Publish, Emit - Events/Messages
// Save, Store, Persist - Storage
// Load, Fetch, Retrieve - Data retrieval
// Convert, Transform, Map - Transformation
```

### Avoid Abbreviations

```csharp
// ❌ BAD
public int CalcTot(int qty, decimal prc);
public string GetCustNm(int custId);
public void ProcOrd(Order ord);

// ✅ GOOD
public decimal CalculateTotal(int quantity, decimal price);
public string GetCustomerName(int customerId);
public void ProcessOrder(Order order);

// Acceptable abbreviations: Id, Url, Html, Xml, Json
// These are widely understood
public int CustomerId;
public string ApiUrl;
```

---

## Method Best Practices

### Single Responsibility

```csharp
// ❌ BAD - Method does too many things
public async Task<OrderResult> ProcessOrder(Order order)
{
    // Validate
    if (order.Items.Count == 0) throw new Exception("No items");
    if (order.Total <= 0) throw new Exception("Invalid total");

    // Check inventory
    foreach (var item in order.Items)
    {
        var stock = await _db.GetStockAsync(item.ProductId);
        if (stock < item.Quantity) throw new Exception("No stock");
    }

    // Process payment
    var paymentResult = await _paymentGateway.ChargeAsync(order.Total);
    if (!paymentResult.Success) throw new Exception("Payment failed");

    // Save order
    await _db.SaveOrderAsync(order);

    // Send email
    await _emailService.SendAsync(order.CustomerEmail, "Order confirmed");

    // Update analytics
    await _analytics.TrackAsync("OrderPlaced", order.Id);

    return new OrderResult { Success = true };
}

// ✅ GOOD - Each method has one job
public async Task<OrderResult> ProcessOrder(Order order)
{
    ValidateOrder(order);
    await EnsureInventoryAvailable(order.Items);
    await ProcessPayment(order);
    await SaveOrder(order);
    await SendConfirmation(order);
    await TrackAnalytics(order);

    return OrderResult.Success(order.Id);
}

private void ValidateOrder(Order order)
{
    if (order.Items.Count == 0)
        throw new ValidationException("Order must have at least one item");
    if (order.Total <= 0)
        throw new ValidationException("Order total must be positive");
}

private async Task EnsureInventoryAvailable(IEnumerable<OrderItem> items)
{
    foreach (var item in items)
    {
        var stock = await _inventoryService.GetStockAsync(item.ProductId);
        if (stock < item.Quantity)
            throw new InsufficientStockException(item.ProductId);
    }
}
```

### Keep Methods Short

```csharp
// Guideline: Methods should fit on one screen (20-30 lines max)
// If longer, consider extracting methods

// ❌ BAD - 100+ line method
public void ProcessReport() { /* ... massive method ... */ }

// ✅ GOOD - Composed of small, focused methods
public void ProcessReport()
{
    var data = FetchReportData();
    var processed = TransformData(data);
    var formatted = FormatOutput(processed);
    SaveReport(formatted);
}
```

### Limit Parameters

```csharp
// ❌ BAD - Too many parameters
public void CreateUser(
    string firstName,
    string lastName,
    string email,
    string phone,
    string address,
    string city,
    string country,
    DateTime dateOfBirth)
{ }

// ✅ GOOD - Use parameter object
public void CreateUser(CreateUserRequest request) { }

public class CreateUserRequest
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public string Phone { get; set; }
    public Address Address { get; set; }
    public DateTime DateOfBirth { get; set; }
}

// Or use builder pattern for optional parameters
var user = new UserBuilder()
    .WithName("John", "Doe")
    .WithEmail("john@example.com")
    .WithAddress(address)
    .Build();
```

### Avoid Flag Arguments

```csharp
// ❌ BAD - Boolean flag is unclear
public void ProcessOrder(Order order, bool urgent)
{
    if (urgent)
        PriorityProcess(order);
    else
        NormalProcess(order);
}

// ✅ GOOD - Separate methods with clear names
public void ProcessOrder(Order order) { /* normal processing */ }
public void ProcessUrgentOrder(Order order) { /* priority processing */ }

// Or use enum for multiple options
public enum ProcessingPriority { Normal, High, Critical }

public void ProcessOrder(Order order, ProcessingPriority priority = ProcessingPriority.Normal)
{
    // Clear what the parameter means
}
```

---

## Code Organization

### Class Structure

```csharp
public class OrderService : IOrderService
{
    // 1. Constants
    private const int MaxRetryAttempts = 3;

    // 2. Static fields
    private static readonly TimeSpan DefaultTimeout = TimeSpan.FromSeconds(30);

    // 3. Private fields
    private readonly IOrderRepository _repository;
    private readonly IPaymentService _paymentService;
    private readonly ILogger<OrderService> _logger;

    // 4. Constructor(s)
    public OrderService(
        IOrderRepository repository,
        IPaymentService paymentService,
        ILogger<OrderService> logger)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _paymentService = paymentService ?? throw new ArgumentNullException(nameof(paymentService));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    // 5. Public properties
    public int ProcessedCount { get; private set; }

    // 6. Public methods
    public async Task<Order> GetOrderAsync(int id)
    {
        return await _repository.GetByIdAsync(id);
    }

    public async Task<OrderResult> ProcessOrderAsync(Order order)
    {
        // Implementation
    }

    // 7. Private methods
    private void ValidateOrder(Order order)
    {
        // Validation logic
    }
}
```

### File Organization

```
📁 src/
├── 📁 MyApp.Domain/           # Domain entities, value objects
│   ├── 📁 Entities/
│   │   ├── Order.cs
│   │   └── Customer.cs
│   └── 📁 ValueObjects/
│       ├── Money.cs
│       └── Address.cs
│
├── 📁 MyApp.Application/      # Business logic, use cases
│   ├── 📁 Services/
│   │   └── OrderService.cs
│   ├── 📁 Commands/
│   │   └── CreateOrderCommand.cs
│   └── 📁 Queries/
│       └── GetOrderQuery.cs
│
├── 📁 MyApp.Infrastructure/   # External concerns
│   ├── 📁 Persistence/
│   │   ├── AppDbContext.cs
│   │   └── OrderRepository.cs
│   └── 📁 External/
│       └── PaymentGateway.cs
│
└── 📁 MyApp.Api/              # Presentation layer
    ├── 📁 Controllers/
    │   └── OrdersController.cs
    └── Program.cs
```

---

## SOLID in Practice

### Single Responsibility (Real Example)

```csharp
// ❌ BAD - Class does everything
public class Order
{
    public int Id { get; set; }
    public List<OrderItem> Items { get; set; }

    public decimal CalculateTotal() { /* ... */ }
    public void SaveToDatabase() { /* ... */ }
    public void SendConfirmationEmail() { /* ... */ }
    public void PrintInvoice() { /* ... */ }
    public void GeneratePdf() { /* ... */ }
}

// ✅ GOOD - Separated concerns
public class Order  // Just data and business rules
{
    public int Id { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal CalculateTotal() => Items.Sum(i => i.Subtotal);
}

public interface IOrderRepository
{
    Task SaveAsync(Order order);
}

public interface IOrderNotificationService
{
    Task SendConfirmationAsync(Order order);
}

public interface IInvoiceGenerator
{
    byte[] GeneratePdf(Order order);
    void Print(Order order);
}
```

### Open/Closed (Real Example)

```csharp
// ❌ BAD - Need to modify class for new discount types
public class DiscountCalculator
{
    public decimal Calculate(Order order, string discountType)
    {
        return discountType switch
        {
            "percentage" => order.Total * 0.1m,
            "fixed" => 50m,
            "loyalty" => order.Customer.Points * 0.01m,
            // Add more cases here... modifying existing code
            _ => 0m
        };
    }
}

// ✅ GOOD - Open for extension, closed for modification
public interface IDiscountStrategy
{
    decimal Calculate(Order order);
}

public class PercentageDiscount : IDiscountStrategy
{
    private readonly decimal _percentage;
    public PercentageDiscount(decimal percentage) => _percentage = percentage;
    public decimal Calculate(Order order) => order.Total * _percentage;
}

public class LoyaltyDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Customer.Points * 0.01m;
}

// New discount types can be added without modifying existing code
public class SeasonalDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.2m;
}
```

---

## Code Smells to Avoid

### Long Method

```csharp
// ❌ SMELL - Method doing too much
public void ProcessOrder(Order order) { /* 200 lines */ }

// ✅ FIX - Extract methods
public void ProcessOrder(Order order)
{
    ValidateOrder(order);
    ReserveInventory(order);
    ProcessPayment(order);
    UpdateOrderStatus(order);
    NotifyCustomer(order);
}
```

### Primitive Obsession

```csharp
// ❌ SMELL - Using primitives for domain concepts
public void ProcessPayment(decimal amount, string currency, string cardNumber)
{
    if (amount <= 0) throw new Exception();
    if (currency.Length != 3) throw new Exception();
    if (cardNumber.Length != 16) throw new Exception();
}

// ✅ FIX - Use value objects
public record Money(decimal Amount, Currency Currency)
{
    public Money(decimal amount, Currency currency)
    {
        if (amount < 0) throw new ArgumentException("Amount cannot be negative");
        Amount = amount;
        Currency = currency;
    }
}

public record CardNumber
{
    public string Value { get; }

    public CardNumber(string value)
    {
        if (string.IsNullOrEmpty(value) || value.Length != 16)
            throw new ArgumentException("Invalid card number");
        Value = value;
    }
}

public void ProcessPayment(Money amount, CardNumber card) { }
```

### Feature Envy

```csharp
// ❌ SMELL - Method uses another object's data too much
public class OrderCalculator
{
    public decimal CalculateDiscount(Customer customer)
    {
        if (customer.LoyaltyTier == "Gold" && customer.TotalOrders > 10)
            return customer.TotalSpent * 0.1m;
        if (customer.LoyaltyTier == "Silver")
            return customer.TotalSpent * 0.05m;
        return 0;
    }
}

// ✅ FIX - Move method to the class that owns the data
public class Customer
{
    public string LoyaltyTier { get; set; }
    public int TotalOrders { get; set; }
    public decimal TotalSpent { get; set; }

    public decimal CalculateDiscount()
    {
        if (LoyaltyTier == "Gold" && TotalOrders > 10)
            return TotalSpent * 0.1m;
        if (LoyaltyTier == "Silver")
            return TotalSpent * 0.05m;
        return 0;
    }
}
```

### God Class

```csharp
// ❌ SMELL - One class knows everything
public class OrderManager
{
    public void CreateOrder() { }
    public void ProcessPayment() { }
    public void SendEmail() { }
    public void GenerateReport() { }
    public void UpdateInventory() { }
    public void CalculateTax() { }
    public void ApplyDiscount() { }
    // ... 50 more methods
}

// ✅ FIX - Split into focused classes
public class OrderService { }
public class PaymentService { }
public class EmailService { }
public class ReportGenerator { }
public class InventoryService { }
public class TaxCalculator { }
public class DiscountService { }
```

### Magic Numbers/Strings

```csharp
// ❌ SMELL - Magic values
if (order.Total > 1000)
    discount = order.Total * 0.1;
if (order.Status == "pending")
    SendReminder();

// ✅ FIX - Use constants or enums
private const decimal DiscountThreshold = 1000m;
private const decimal DiscountPercentage = 0.1m;

public enum OrderStatus { Pending, Processing, Completed, Cancelled }

if (order.Total > DiscountThreshold)
    discount = order.Total * DiscountPercentage;
if (order.Status == OrderStatus.Pending)
    SendReminder();
```

### Nested Conditionals

```csharp
// ❌ SMELL - Deep nesting
public void ProcessOrder(Order order)
{
    if (order != null)
    {
        if (order.Items.Any())
        {
            if (order.Customer != null)
            {
                if (order.Customer.IsActive)
                {
                    // Finally do something
                }
            }
        }
    }
}

// ✅ FIX - Guard clauses (early return)
public void ProcessOrder(Order order)
{
    if (order == null)
        throw new ArgumentNullException(nameof(order));

    if (!order.Items.Any())
        throw new InvalidOperationException("Order has no items");

    if (order.Customer == null)
        throw new InvalidOperationException("Order has no customer");

    if (!order.Customer.IsActive)
        throw new InvalidOperationException("Customer is inactive");

    // Now do the actual work - no nesting
    ProcessValidOrder(order);
}
```

---

## Comments Best Practices

### When to Comment

```csharp
// ✅ GOOD - Explain WHY, not WHAT
// Using retry because payment gateway has intermittent failures
// during peak hours (based on incident INC-1234)
await Policy.Handle<PaymentException>()
    .WaitAndRetryAsync(3)
    .ExecuteAsync(() => _gateway.ProcessAsync(payment));

// ✅ GOOD - Document non-obvious behavior
// Note: Returns null for soft-deleted customers per GDPR requirements
public async Task<Customer?> GetCustomerAsync(int id)

// ✅ GOOD - Explain complex algorithms
// Using Levenshtein distance for fuzzy matching
// Threshold of 3 allows minor typos while preventing false positives

// ❌ BAD - Comment states the obvious
// Increment counter by 1
counter++;

// Get the customer from database
var customer = await _repository.GetByIdAsync(id);
```

### XML Documentation (Public APIs)

```csharp
/// <summary>
/// Processes a payment for the specified order.
/// </summary>
/// <param name="orderId">The unique identifier of the order.</param>
/// <param name="paymentMethod">The payment method to use.</param>
/// <returns>
/// A result indicating success or failure, including transaction ID on success.
/// </returns>
/// <exception cref="OrderNotFoundException">
/// Thrown when the specified order does not exist.
/// </exception>
/// <exception cref="PaymentException">
/// Thrown when payment processing fails after retry attempts.
/// </exception>
/// <example>
/// <code>
/// var result = await paymentService.ProcessPaymentAsync(123, PaymentMethod.CreditCard);
/// if (result.Success)
///     Console.WriteLine($"Transaction: {result.TransactionId}");
/// </code>
/// </example>
public async Task<PaymentResult> ProcessPaymentAsync(int orderId, PaymentMethod paymentMethod)
```

---

## Quick Reference

### Do's

- ✅ Use meaningful, descriptive names
- ✅ Keep methods short and focused
- ✅ Use guard clauses for validation
- ✅ Extract complex conditionals into methods
- ✅ Follow consistent formatting
- ✅ Write self-documenting code
- ✅ Use value objects for domain concepts
- ✅ Prefer composition over inheritance

### Don'ts

- ❌ Use abbreviations (except common ones)
- ❌ Use magic numbers/strings
- ❌ Write long methods (>30 lines)
- ❌ Use deep nesting
- ❌ Comment what code does (comment WHY)
- ❌ Use boolean parameters
- ❌ Have methods with >3 parameters
- ❌ Create god classes

### Code Review Checklist

1. **Readability**: Can I understand this in 5 minutes?
2. **Naming**: Are names clear and consistent?
3. **Methods**: Are they short and focused?
4. **Error handling**: Are exceptions handled properly?
5. **Tests**: Are critical paths tested?
6. **Security**: Any SQL injection, XSS, etc.?
7. **Performance**: Any obvious N+1, memory issues?
