# Clean Code - Interview Questions

> 5 real-world interview questions with detailed answers and communication tactics

---

## Question 1: Code Smell Identification

### The Question
> "Review this code and identify the code smells present. How would you improve it?"

```csharp
public class OrderManager
{
    public decimal ProcessOrder(int orderId, int customerId, string customerType,
        decimal amount, bool applyDiscount, string discountCode, bool sendEmail,
        string emailAddress, bool logTransaction, string logLevel)
    {
        // Get customer
        var conn = new SqlConnection("Server=prod;Database=Orders;...");
        conn.Open();
        var cmd = new SqlCommand($"SELECT * FROM Customers WHERE Id = {customerId}", conn);
        var reader = cmd.ExecuteReader();
        // ... more code

        decimal finalAmount = amount;

        // Apply discount
        if (applyDiscount)
        {
            if (customerType == "premium")
            {
                if (discountCode == "SUMMER20")
                {
                    finalAmount = amount * 0.8m;
                }
                else if (discountCode == "VIP10")
                {
                    finalAmount = amount * 0.9m;
                }
            }
            else if (customerType == "standard")
            {
                if (discountCode == "SUMMER20")
                {
                    finalAmount = amount * 0.85m;
                }
            }
        }

        // Save order
        var cmd2 = new SqlCommand($"INSERT INTO Orders VALUES ({orderId}, {customerId}, {finalAmount})", conn);
        cmd2.ExecuteNonQuery();

        // Send email
        if (sendEmail)
        {
            var smtp = new SmtpClient("smtp.company.com");
            smtp.Send("orders@company.com", emailAddress, "Order Confirmed", "...");
        }

        // Log
        if (logTransaction)
        {
            File.AppendAllText("C:\\logs\\orders.txt", $"{DateTime.Now}: Order {orderId} processed");
        }

        conn.Close();
        return finalAmount;
    }
}
```

### Key Points to Cover
- Identify each code smell by name
- Explain why each is problematic
- Show improved version

### Detailed Answer

**Code Smells Identified:**

```
1. LONG PARAMETER LIST (10 parameters!)
   - Hard to remember order
   - Easy to pass wrong values
   - Indicates method does too much

2. GOD METHOD
   - Method does everything: DB access, discount, email, logging
   - Violates Single Responsibility Principle
   - Impossible to test in isolation

3. SQL INJECTION VULNERABILITY
   - String concatenation in SQL
   - Critical security issue

4. MAGIC STRINGS
   - "premium", "standard", "SUMMER20", "VIP10"
   - Scattered throughout code

5. NESTED CONDITIONALS
   - Deep if-else nesting
   - Hard to read and modify

6. HARDCODED DEPENDENCIES
   - Direct instantiation of SqlConnection, SmtpClient
   - Can't mock for testing
   - Violates Dependency Inversion

7. RESOURCE MANAGEMENT
   - No using statements
   - Connection might not close on exception

8. PRIMITIVE OBSESSION
   - Boolean flags instead of proper types
   - String for customer type instead of enum
```

**Refactored Solution:**

```csharp
// Enums instead of magic strings
public enum CustomerType { Standard, Premium, VIP }
public enum DiscountCode { None, Summer20, Vip10 }

// Request object instead of many parameters
public record ProcessOrderRequest(
    int OrderId,
    int CustomerId,
    decimal Amount,
    DiscountCode DiscountCode = DiscountCode.None);

// Separate concerns into focused classes
public interface IDiscountCalculator
{
    decimal CalculateDiscount(decimal amount, CustomerType customerType, DiscountCode code);
}

public class DiscountCalculator : IDiscountCalculator
{
    private static readonly Dictionary<(CustomerType, DiscountCode), decimal> _rates = new()
    {
        [(CustomerType.Premium, DiscountCode.Summer20)] = 0.20m,
        [(CustomerType.Premium, DiscountCode.Vip10)] = 0.10m,
        [(CustomerType.Standard, DiscountCode.Summer20)] = 0.15m
    };

    public decimal CalculateDiscount(decimal amount, CustomerType type, DiscountCode code)
    {
        if (code == DiscountCode.None)
            return 0m;

        var rate = _rates.GetValueOrDefault((type, code), 0m);
        return amount * rate;
    }
}

// Clean order service
public class OrderService
{
    private readonly ICustomerRepository _customerRepo;
    private readonly IOrderRepository _orderRepo;
    private readonly IDiscountCalculator _discountCalculator;
    private readonly IOrderNotificationService _notificationService;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        ICustomerRepository customerRepo,
        IOrderRepository orderRepo,
        IDiscountCalculator discountCalculator,
        IOrderNotificationService notificationService,
        ILogger<OrderService> logger)
    {
        _customerRepo = customerRepo;
        _orderRepo = orderRepo;
        _discountCalculator = discountCalculator;
        _notificationService = notificationService;
        _logger = logger;
    }

    public async Task<OrderResult> ProcessOrderAsync(ProcessOrderRequest request)
    {
        var customer = await _customerRepo.GetByIdAsync(request.CustomerId)
            ?? throw new CustomerNotFoundException(request.CustomerId);

        var discount = _discountCalculator.CalculateDiscount(
            request.Amount,
            customer.Type,
            request.DiscountCode);

        var finalAmount = request.Amount - discount;

        var order = new Order
        {
            Id = request.OrderId,
            CustomerId = customer.Id,
            Amount = finalAmount
        };

        await _orderRepo.SaveAsync(order);

        _logger.LogInformation(
            "Order {OrderId} processed for {Amount:C}",
            order.Id, finalAmount);

        await _notificationService.SendOrderConfirmationAsync(customer, order);

        return new OrderResult(order.Id, finalAmount);
    }
}
```

### Communication Tactics

🎯 **Structure your answer**: Identify smells one by one with names, explain the problems, then show the improved code.

💡 **Emphasize**: Focus on the most critical issues first (SQL injection, SRP violation). Show you prioritize security and maintainability.

⚠️ **Avoid**: Don't just say "it's bad." Name specific smells and explain WHY they're problematic.

---

## Question 2: Refactoring a Long Method

### The Question
> "How would you refactor a 200-line method? Walk me through your approach."

### Key Points to Cover
- Systematic approach
- Extract method refactoring
- Preserving behavior
- Testing strategy

### Detailed Answer

**Step-by-Step Approach:**

```csharp
// STEP 1: Characterization Tests
// Before ANY refactoring, capture current behavior

[Fact]
public void ProcessOrder_CurrentBehavior_CaptureOutput()
{
    // Document what the method currently does
    // Even if behavior seems wrong, we need to preserve it
    var result = _service.ProcessOrder(testInput);

    // Assert current behavior (we'll verify this stays the same)
    Assert.Equal(expected, result);
}

// STEP 2: Identify Logical Sections
// Read through and mark sections with comments

public void LongMethod()
{
    // === SECTION 1: Validation (lines 1-30) ===
    // ...

    // === SECTION 2: Data Retrieval (lines 31-60) ===
    // ...

    // === SECTION 3: Business Logic (lines 61-120) ===
    // ...

    // === SECTION 4: Persistence (lines 121-160) ===
    // ...

    // === SECTION 5: Notifications (lines 161-200) ===
    // ...
}
```

**Extract Method Refactoring:**

```csharp
// STEP 3: Extract methods one section at a time

// BEFORE
public OrderResult ProcessOrder(OrderRequest request)
{
    // 30 lines of validation
    if (string.IsNullOrEmpty(request.CustomerEmail))
        throw new ValidationException("Email required");
    if (request.Items == null || !request.Items.Any())
        throw new ValidationException("Items required");
    if (request.Items.Any(i => i.Quantity <= 0))
        throw new ValidationException("Invalid quantity");
    // ... more validation

    // 30 lines of price calculation
    decimal subtotal = 0;
    foreach (var item in request.Items)
    {
        var product = GetProduct(item.ProductId);
        subtotal += product.Price * item.Quantity;
    }
    var discount = CalculateDiscount(request.CustomerId, subtotal);
    var tax = (subtotal - discount) * 0.18m;
    var total = subtotal - discount + tax;
    // ... more calculation

    // ... rest of the 140 lines
}

// AFTER - Extract Section by Section
public OrderResult ProcessOrder(OrderRequest request)
{
    ValidateRequest(request);

    var pricing = CalculatePricing(request);

    var order = CreateOrder(request, pricing);

    await SaveOrderAsync(order);

    await SendNotificationsAsync(order);

    return new OrderResult(order);
}

private void ValidateRequest(OrderRequest request)
{
    if (string.IsNullOrEmpty(request.CustomerEmail))
        throw new ValidationException("Email required");
    if (request.Items == null || !request.Items.Any())
        throw new ValidationException("Items required");
    if (request.Items.Any(i => i.Quantity <= 0))
        throw new ValidationException("Invalid quantity");
}

private OrderPricing CalculatePricing(OrderRequest request)
{
    decimal subtotal = request.Items.Sum(item =>
    {
        var product = GetProduct(item.ProductId);
        return product.Price * item.Quantity;
    });

    var discount = CalculateDiscount(request.CustomerId, subtotal);
    var tax = (subtotal - discount) * TaxRate;

    return new OrderPricing(subtotal, discount, tax);
}
```

**Guidelines:**

```csharp
// NAMING: Method name should describe WHAT, not HOW
// BAD:
private void DoStuff() { }
private void HandleData() { }

// GOOD:
private void ValidateShippingAddress() { }
private decimal CalculateShippingCost() { }

// PARAMETERS: Keep extracted methods focused
// BAD: Passing the entire request when only email is needed
private void SendConfirmation(OrderRequest request) { }

// GOOD: Pass only what's needed
private void SendConfirmation(string email, Order order) { }

// SIZE: Aim for 10-20 lines per method
// If extracted method is still large, extract further
```

**Testing After Refactoring:**

```csharp
// STEP 4: Add unit tests for extracted methods

[Fact]
public void ValidateRequest_WithMissingEmail_ThrowsValidationException()
{
    var request = new OrderRequest { CustomerEmail = null };

    Assert.Throws<ValidationException>(() => _service.ValidateRequest(request));
}

[Fact]
public void CalculatePricing_WithDiscount_AppliesCorrectly()
{
    var pricing = _service.CalculatePricing(requestWithDiscount);

    Assert.Equal(expectedDiscount, pricing.Discount);
}

// Original characterization tests should still pass!
```

### Communication Tactics

🎯 **Structure your answer**: Emphasize safety - tests first, then systematic extraction. Show you refactor incrementally, not all at once.

💡 **Emphasize**: "I'd never refactor without tests. First characterization tests to capture current behavior, then extract methods one at a time."

⚠️ **Avoid**: Don't suggest rewriting from scratch. Show disciplined, safe refactoring approach.

---

## Question 3: When to Write Comments

### The Question
> "When should you write comments and when should you avoid them? Give me examples of good and bad comments."

### Key Points to Cover
- Self-documenting code preference
- When comments ARE valuable
- Types of bad comments
- XML documentation for APIs

### Detailed Answer

**The Hierarchy:**

```
1. Best: Self-documenting code (no comment needed)
2. Good: Comment explaining WHY (not what)
3. Bad: Comment explaining WHAT code does
4. Worst: Outdated/incorrect comment
```

**When to AVOID Comments:**

```csharp
// ❌ BAD: Stating the obvious
// Increment counter by 1
counter++;

// ❌ BAD: Repeating the code
// Get the customer by ID
var customer = repository.GetCustomerById(id);

// ❌ BAD: Commented-out code
// public void OldMethod() { ... }

// ❌ BAD: Comment to explain unclear code (fix the code instead!)
// Set x to 5 because that's the max retry count
int x = 5;
// BETTER: Just name it properly
int maxRetryCount = 5;
```

**When to WRITE Comments:**

```csharp
// ✅ GOOD: Explain WHY, not WHAT
// Using retry with exponential backoff because the payment gateway
// has intermittent failures during peak hours (see incident INC-1234)
await Policy
    .Handle<PaymentException>()
    .WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)))
    .ExecuteAsync(() => gateway.ProcessAsync(payment));

// ✅ GOOD: Document non-obvious behavior
// Returns null for soft-deleted customers per GDPR compliance requirements
public Customer? GetCustomer(int id)
{
    return _context.Customers
        .Where(c => c.Id == id && !c.IsDeleted)
        .FirstOrDefault();
}

// ✅ GOOD: Explain complex algorithms
// Using Levenshtein distance for fuzzy matching.
// Threshold of 3 allows minor typos while preventing false positives.
// Based on analysis of customer search patterns (see docs/search-analysis.md)
public bool IsFuzzyMatch(string input, string candidate)
{
    return LevenshteinDistance(input, candidate) <= 3;
}

// ✅ GOOD: Warn about non-obvious gotchas
// IMPORTANT: This method must be called before ProcessPayment
// because it initializes the fraud detection context
public void InitializeFraudCheck(Customer customer)

// ✅ GOOD: TODO with context (not just "TODO: fix this")
// TODO(jsmith): Optimize this query after we add the composite index
// Tracked in JIRA-456, scheduled for next sprint
public async Task<List<Order>> GetOrdersAsync()

// ✅ GOOD: External references
// Implementation based on RFC 7519 (JWT)
// See: https://tools.ietf.org/html/rfc7519#section-4.1
public JwtPayload ParseToken(string token)
```

**XML Documentation for Public APIs:**

```csharp
/// <summary>
/// Processes a payment for the specified order.
/// </summary>
/// <param name="orderId">The unique identifier of the order.</param>
/// <param name="amount">The amount to charge in the order's currency.</param>
/// <returns>
/// A payment result containing the transaction ID if successful,
/// or error details if the payment failed.
/// </returns>
/// <exception cref="OrderNotFoundException">
/// Thrown when the specified order does not exist.
/// </exception>
/// <exception cref="PaymentDeclinedException">
/// Thrown when the payment is declined by the gateway.
/// </exception>
/// <example>
/// <code>
/// var result = await paymentService.ProcessPaymentAsync(123, 99.99m);
/// if (result.Success)
///     Console.WriteLine($"Transaction: {result.TransactionId}");
/// </code>
/// </example>
public async Task<PaymentResult> ProcessPaymentAsync(int orderId, decimal amount)
```

**Self-Documenting Code Examples:**

```csharp
// Instead of:
// Check if user is admin
if (user.Role == 1) { }

// Write:
if (user.IsAdmin) { }

// Instead of:
// Calculate total with tax
var t = s * 1.18m;

// Write:
var totalWithTax = subtotal * (1 + taxRate);

// Instead of:
// Check if order can be cancelled
if (order.Status != 3 && order.Status != 4 && DateTime.Now < order.Date.AddDays(30)) { }

// Write:
if (order.CanBeCancelled) { }

public bool CanBeCancelled =>
    Status != OrderStatus.Shipped &&
    Status != OrderStatus.Delivered &&
    DateTime.UtcNow < CreatedAt.AddDays(CancellationWindowDays);
```

### Communication Tactics

🎯 **Structure your answer**: Show the spectrum from bad to good comments. Give concrete examples of each category.

💡 **Emphasize**: "Comments should explain WHY, not WHAT. If I need to explain what the code does, I should improve the code instead."

⚠️ **Avoid**: Don't say "never write comments." Show you understand when they're valuable (complex algorithms, non-obvious gotchas, public APIs).

---

## Question 4: Good Method Design

### The Question
> "What makes a good function or method according to clean code principles? Walk me through your criteria."

### Key Points to Cover
- Single responsibility
- Small size
- Clear naming
- Parameter count
- Side effects

### Detailed Answer

**The Criteria:**

```csharp
// 1. SINGLE RESPONSIBILITY
// A method should do ONE thing and do it well

// ❌ BAD: Does multiple things
public void ProcessOrder(Order order)
{
    ValidateOrder(order);           // Validation
    CalculateTax(order);            // Calculation
    SaveToDatabase(order);          // Persistence
    SendConfirmationEmail(order);   // Notification
    UpdateInventory(order);         // Inventory
}

// ✅ GOOD: Each method does one thing
public void ProcessOrder(Order order)
{
    ValidateOrder(order);
    var processedOrder = ApplyBusinessRules(order);
    await SaveOrderAsync(processedOrder);
    await PublishOrderCreatedEvent(processedOrder);  // Other concerns react to event
}
```

```csharp
// 2. SMALL SIZE
// Methods should fit on one screen (20-30 lines max)

// ❌ BAD: 100+ line method
public void GenerateReport() { /* 150 lines */ }

// ✅ GOOD: Composed of small, focused methods
public Report GenerateReport()
{
    var data = FetchReportData();
    var processedData = ProcessData(data);
    var formattedReport = FormatReport(processedData);
    return formattedReport;
}
```

```csharp
// 3. CLEAR NAMING
// Name describes what the method does (verb + noun)

// ❌ BAD: Unclear names
public void Handle() { }
public void Process() { }
public void DoIt() { }
public void Manage() { }

// ✅ GOOD: Descriptive names
public Customer GetCustomerById(int id) { }
public void SendOrderConfirmation(Order order) { }
public bool IsEligibleForDiscount(Customer customer) { }
public decimal CalculateTotalWithTax(Order order) { }

// Naming conventions:
// - GetX / FindX     - Retrieval
// - CreateX / BuildX - Creation
// - UpdateX          - Modification
// - DeleteX / RemoveX - Deletion
// - IsX / HasX / CanX - Boolean queries
// - CalculateX       - Computation
// - ValidateX        - Validation
```

```csharp
// 4. LIMITED PARAMETERS (3 or fewer)

// ❌ BAD: Too many parameters
public void CreateUser(
    string firstName, string lastName, string email,
    string phone, string address, string city,
    string country, DateTime dateOfBirth) { }

// ✅ GOOD: Use parameter objects
public void CreateUser(CreateUserRequest request) { }

public record CreateUserRequest(
    string FirstName,
    string LastName,
    string Email,
    Address Address,
    DateTime DateOfBirth);
```

```csharp
// 5. NO SIDE EFFECTS (or clear naming if unavoidable)

// ❌ BAD: Hidden side effect
public bool CheckPassword(string password)
{
    var isValid = ValidatePassword(password);
    if (!isValid)
        _loginAttempts++;  // Side effect! Caller doesn't expect this
    return isValid;
}

// ✅ GOOD: Pure function
public bool IsValidPassword(string password)
{
    return password.Length >= 8
        && password.Any(char.IsDigit)
        && password.Any(char.IsUpper);
}

// ✅ GOOD: If side effect needed, name it clearly
public bool ValidatePasswordAndRecordAttempt(string password)
{
    var isValid = IsValidPassword(password);
    RecordLoginAttempt(isValid);
    return isValid;
}
```

```csharp
// 6. SINGLE LEVEL OF ABSTRACTION

// ❌ BAD: Mixed abstraction levels
public void ProcessOrder(Order order)
{
    ValidateOrder(order);                    // High-level
    var sql = "INSERT INTO Orders...";       // Low-level (SQL)
    SendEmail(order);                        // High-level
    File.WriteAllText("log.txt", "...");    // Low-level (file I/O)
}

// ✅ GOOD: Consistent abstraction level
public async Task ProcessOrderAsync(Order order)
{
    ValidateOrder(order);
    await SaveOrderAsync(order);
    await SendConfirmationAsync(order);
    LogOrderProcessed(order);
}
```

```csharp
// 7. AVOID FLAG ARGUMENTS

// ❌ BAD: Boolean flag
public void ProcessOrder(Order order, bool expedited)
{
    if (expedited)
        ProcessExpedited(order);
    else
        ProcessNormal(order);
}

// ✅ GOOD: Separate methods or use enum
public void ProcessOrder(Order order) { }
public void ProcessExpeditedOrder(Order order) { }

// Or
public void ProcessOrder(Order order, ProcessingSpeed speed) { }
```

**Summary Checklist:**

| Criteria | Check |
|----------|-------|
| Does ONE thing | ✓ |
| 20-30 lines max | ✓ |
| Descriptive name | ✓ |
| ≤3 parameters | ✓ |
| No hidden side effects | ✓ |
| Consistent abstraction | ✓ |
| No boolean flags | ✓ |

### Communication Tactics

🎯 **Structure your answer**: Walk through each criterion with examples. Show bad vs good for each.

💡 **Emphasize**: "A good method is like a good paragraph - it has one clear topic and is easy to read."

⚠️ **Avoid**: Don't be dogmatic. Acknowledge that guidelines can be bent for good reasons.

---

## Question 5: Naming Conventions

### The Question
> "How do you name things? Walk me through your naming conventions for classes, methods, variables, etc."

### Key Points to Cover
- C# conventions
- Meaningful names
- Context-appropriate length
- Common patterns

### Detailed Answer

**C# Naming Conventions:**

```csharp
// CLASSES - PascalCase, Nouns
public class OrderService { }
public class CustomerRepository { }
public class PaymentProcessor { }
public class ShippingCalculator { }

// INTERFACES - PascalCase with "I" prefix
public interface IOrderService { }
public interface IRepository<T> { }
public interface IPaymentGateway { }

// METHODS - PascalCase, Verbs
public void ProcessOrder() { }
public Customer GetCustomerById(int id) { }
public bool IsValid() { }
public async Task SendEmailAsync() { }

// PROPERTIES - PascalCase
public string FirstName { get; set; }
public DateTime CreatedAt { get; }
public bool IsActive { get; set; }

// PRIVATE FIELDS - _camelCase
private readonly ILogger<OrderService> _logger;
private readonly IOrderRepository _repository;
private int _retryCount;

// LOCAL VARIABLES - camelCase
var orderTotal = CalculateTotal();
var customer = await GetCustomerAsync(id);
var isValid = ValidateRequest(request);

// PARAMETERS - camelCase
public void ProcessOrder(int orderId, Customer customer) { }

// CONSTANTS - PascalCase
public const int MaxRetryCount = 3;
public const string DefaultCurrency = "TRY";
private const decimal TaxRate = 0.18m;

// ASYNC METHODS - Suffix with "Async"
public async Task<Order> GetOrderAsync(int id) { }
public async Task SendNotificationAsync() { }
```

**Meaningful Names:**

```csharp
// ❌ BAD: Unclear abbreviations
public void Proc() { }
public int CalcTot(int qty, decimal prc) { }
string custNm;
int x, y, z;
bool flag;

// ✅ GOOD: Clear, descriptive names
public void ProcessPayment() { }
public decimal CalculateTotal(int quantity, decimal price) { }
string customerName;
int orderCount, itemTotal, remainingStock;
bool isEligibleForDiscount;
```

**Context-Appropriate Length:**

```csharp
// SHORT names OK in small scopes
for (int i = 0; i < items.Count; i++) { }

items.Select(x => x.Name);  // Lambda - short scope

// LONGER names for wider scopes
public class CustomerOrderHistoryService { }
private readonly IPaymentTransactionRepository _paymentTransactionRepository;
public decimal CalculateTotalOrderValueWithDiscount() { }
```

**Boolean Naming:**

```csharp
// Use Is, Has, Can, Should for booleans
public bool IsActive { get; set; }
public bool HasOrders { get; }
public bool CanBeCancelled { get; }
public bool ShouldRetry { get; }

// Method names
public bool IsValidEmail(string email) { }
public bool HasPermission(User user, Permission permission) { }
public bool CanProcessOrder(Order order) { }
```

**Collection Naming:**

```csharp
// Plurals for collections
public List<Order> Orders { get; }
public IEnumerable<Customer> GetActiveCustomers() { }

// Or descriptive nouns
public List<Order> OrderHistory { get; }
public Queue<Message> MessageQueue { get; }
```

**Common Patterns:**

```csharp
// REPOSITORY PATTERN
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task<IEnumerable<Order>> GetAllAsync();
    Task<IEnumerable<Order>> FindAsync(Expression<Func<Order, bool>> predicate);
    Task AddAsync(Order order);
    Task UpdateAsync(Order order);
    Task DeleteAsync(int id);
}

// SERVICE PATTERN
public interface IOrderService
{
    Task<OrderResult> CreateOrderAsync(CreateOrderRequest request);
    Task<Order?> GetOrderAsync(int id);
    Task<bool> CancelOrderAsync(int id);
}

// FACTORY PATTERN
public interface IPaymentProcessorFactory
{
    IPaymentProcessor Create(string paymentMethod);
}

// HANDLER PATTERN (CQRS)
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Order>
{
    public async Task<Order> Handle(CreateOrderCommand request, CancellationToken ct) { }
}
```

**Domain-Specific Naming:**

```csharp
// Use domain language (Ubiquitous Language from DDD)

// Financial domain
public class PaymentTransaction { }
public decimal GrossAmount { get; }
public decimal NetAmount { get; }
public void Settle() { }
public void Refund() { }

// E-commerce domain
public class ShoppingCart { }
public void AddItem() { }
public void RemoveItem() { }
public void Checkout() { }

// Avoid technical jargon in domain code
// ❌ BAD
public class OrderDTO { }
public void SerializeOrder() { }

// ✅ GOOD
public class OrderSummary { }
public void ExportOrder() { }
```

### Communication Tactics

🎯 **Structure your answer**: Cover the main categories (classes, methods, variables) with examples. Mention C# conventions specifically.

💡 **Emphasize**: "Good naming makes code self-documenting. I should be able to understand what code does from its names alone."

⚠️ **Avoid**: Don't be rigid about length. "Descriptive" doesn't mean "verbose." `i` is fine in a loop. `customerOrderHistoryItemCount` might be too long.

---

## Quick Review - Key Takeaways

| Question | Key Point |
|----------|-----------|
| Code Smells | Name them specifically, prioritize by severity |
| Long Method | Tests first, extract incrementally |
| Comments | Explain WHY, not WHAT |
| Good Methods | One thing, small, clear name, few params |
| Naming | PascalCase for public, _camelCase for private, meaningful names |
