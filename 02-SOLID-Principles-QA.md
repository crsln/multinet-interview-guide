# SOLID Principles - Interview Questions

> 5 real-world interview questions with detailed answers and communication tactics

---

## Question 1: Walk Through SOLID - Show Violations and Fixes

### The Question
> "Can you walk me through each SOLID principle? For each one, show me a code violation and how you'd fix it."

### Key Points to Cover
- Clear definition of each principle
- Concrete code examples showing violation
- Refactored solution
- Real-world context

### Detailed Answer

**S - Single Responsibility Principle:**

```csharp
// ❌ VIOLATION - Class has multiple responsibilities
public class UserService
{
    public void CreateUser(User user)
    {
        // 1. Validate user
        if (string.IsNullOrEmpty(user.Email))
            throw new ValidationException("Email required");

        // 2. Save to database
        _dbContext.Users.Add(user);
        _dbContext.SaveChanges();

        // 3. Send welcome email
        var smtp = new SmtpClient("smtp.server.com");
        smtp.Send("noreply@company.com", user.Email, "Welcome!", "...");

        // 4. Log the action
        File.AppendAllText("log.txt", $"User {user.Id} created");
    }
}

// ✅ FIX - Each class has one responsibility
public class UserValidator
{
    public ValidationResult Validate(User user) { /* validation logic */ }
}

public class UserRepository
{
    public void Add(User user) { /* database logic */ }
}

public class WelcomeEmailSender
{
    public Task SendAsync(User user) { /* email logic */ }
}

public class UserService
{
    private readonly UserValidator _validator;
    private readonly UserRepository _repository;
    private readonly WelcomeEmailSender _emailSender;
    private readonly ILogger<UserService> _logger;

    public async Task CreateUserAsync(User user)
    {
        _validator.Validate(user);
        _repository.Add(user);
        await _emailSender.SendAsync(user);
        _logger.LogInformation("User {UserId} created", user.Id);
    }
}
```

**O - Open/Closed Principle:**

```csharp
// ❌ VIOLATION - Must modify class to add new discount types
public class DiscountCalculator
{
    public decimal Calculate(Order order, string discountType)
    {
        switch (discountType)
        {
            case "percentage":
                return order.Total * 0.1m;
            case "fixed":
                return 50m;
            case "loyalty":
                return order.Customer.Points * 0.01m;
            // Need to add more cases here for new discount types!
            default:
                return 0;
        }
    }
}

// ✅ FIX - Open for extension, closed for modification
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

// New discount types - just add new class, no modification needed
public class SeasonalDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.2m;
}

public class DiscountCalculator
{
    public decimal Calculate(Order order, IDiscountStrategy strategy)
    {
        return strategy.Calculate(order);
    }
}
```

**L - Liskov Substitution Principle:**

```csharp
// ❌ VIOLATION - Subclass changes expected behavior
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }

    public int Area => Width * Height;
}

public class Square : Rectangle
{
    public override int Width
    {
        get => base.Width;
        set { base.Width = value; base.Height = value; } // Surprise!
    }

    public override int Height
    {
        get => base.Height;
        set { base.Width = value; base.Height = value; } // Surprise!
    }
}

// This breaks!
Rectangle rect = new Square();
rect.Width = 5;
rect.Height = 10;
Console.WriteLine(rect.Area); // Expected: 50, Actual: 100!

// ✅ FIX - Separate abstractions
public interface IShape
{
    int Area { get; }
}

public class Rectangle : IShape
{
    public int Width { get; }
    public int Height { get; }
    public int Area => Width * Height;

    public Rectangle(int width, int height)
    {
        Width = width;
        Height = height;
    }
}

public class Square : IShape
{
    public int Side { get; }
    public int Area => Side * Side;

    public Square(int side) => Side = side;
}
```

**I - Interface Segregation Principle:**

```csharp
// ❌ VIOLATION - Fat interface forces unnecessary implementations
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
    void TakeBreak();
}

public class Robot : IWorker
{
    public void Work() { /* works */ }
    public void Eat() => throw new NotImplementedException(); // Robots don't eat!
    public void Sleep() => throw new NotImplementedException(); // Or sleep!
    public void TakeBreak() => throw new NotImplementedException();
}

// ✅ FIX - Segregated interfaces
public interface IWorkable
{
    void Work();
}

public interface IFeedable
{
    void Eat();
}

public interface IRestable
{
    void Sleep();
    void TakeBreak();
}

public class Human : IWorkable, IFeedable, IRestable
{
    public void Work() { }
    public void Eat() { }
    public void Sleep() { }
    public void TakeBreak() { }
}

public class Robot : IWorkable
{
    public void Work() { } // Only implements what it needs
}
```

**D - Dependency Inversion Principle:**

```csharp
// ❌ VIOLATION - High-level module depends on low-level module
public class OrderService
{
    private readonly SqlServerDatabase _database = new SqlServerDatabase();
    private readonly SmtpEmailSender _emailSender = new SmtpEmailSender();

    public void ProcessOrder(Order order)
    {
        _database.Save(order);      // Directly coupled to SQL Server
        _emailSender.Send(order);   // Directly coupled to SMTP
    }
}

// ✅ FIX - Depend on abstractions
public interface IOrderRepository
{
    Task SaveAsync(Order order);
}

public interface INotificationService
{
    Task NotifyAsync(Order order);
}

public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly INotificationService _notificationService;

    // Dependencies injected - can be SQL, MongoDB, Mock, etc.
    public OrderService(IOrderRepository repository, INotificationService notificationService)
    {
        _repository = repository;
        _notificationService = notificationService;
    }

    public async Task ProcessOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);
        await _notificationService.NotifyAsync(order);
    }
}
```

### Communication Tactics

🎯 **Structure your answer**: Go through S-O-L-I-D in order. For each: name it, explain in one sentence, show violation, show fix. Keep examples small and focused.

💡 **Emphasize**: These principles work together. SRP makes classes focused. OCP makes them extensible. LSP ensures reliable inheritance. ISP keeps interfaces lean. DIP enables all of this through abstractions.

⚠️ **Avoid**: Don't just recite acronyms. The interviewer wants to see you understand WHY and WHEN to apply them.

---

## Question 2: SRP Violation Code Review

### The Question
> "Look at this class. It handles user creation, validation, and email sending. What's wrong with it and how would you refactor it?"

```csharp
public class UserManager
{
    public bool CreateUser(string email, string password, string name)
    {
        // Validate
        if (!email.Contains("@")) return false;
        if (password.Length < 8) return false;
        if (string.IsNullOrEmpty(name)) return false;

        // Hash password
        var hash = BCrypt.HashPassword(password);

        // Save to database
        using var conn = new SqlConnection("...");
        conn.Open();
        var cmd = new SqlCommand("INSERT INTO Users...", conn);
        cmd.ExecuteNonQuery();

        // Send welcome email
        var client = new SmtpClient("smtp.server.com");
        client.Send("no-reply@company.com", email, "Welcome!", "...");

        // Log
        File.AppendAllText("users.log", $"{DateTime.Now}: Created {email}");

        return true;
    }
}
```

### Key Points to Cover
- Identify all the responsibilities (4-5)
- Explain problems with this approach
- Show step-by-step refactoring
- Mention testability benefits

### Detailed Answer

**Identifying the Problems:**

This class has at least 5 responsibilities:
1. **Validation** - checking email, password, name
2. **Password hashing** - security concern
3. **Database access** - persistence
4. **Email sending** - notification
5. **Logging** - audit/debugging

**Problems this causes:**
- Can't test validation without database/email
- Can't reuse validation elsewhere
- Can't change email provider without touching user logic
- Can't mock dependencies
- Changes to logging affect user creation

**Refactored Solution:**

```csharp
// 1. Validation - its own concern
public interface IUserValidator
{
    ValidationResult Validate(CreateUserRequest request);
}

public class UserValidator : IUserValidator
{
    public ValidationResult Validate(CreateUserRequest request)
    {
        var errors = new List<string>();

        if (!request.Email.Contains("@"))
            errors.Add("Invalid email format");

        if (request.Password.Length < 8)
            errors.Add("Password must be at least 8 characters");

        if (string.IsNullOrWhiteSpace(request.Name))
            errors.Add("Name is required");

        return new ValidationResult(errors);
    }
}

// 2. Password hashing - its own concern
public interface IPasswordHasher
{
    string Hash(string password);
    bool Verify(string password, string hash);
}

public class BcryptPasswordHasher : IPasswordHasher
{
    public string Hash(string password) => BCrypt.HashPassword(password);
    public bool Verify(string password, string hash) => BCrypt.Verify(password, hash);
}

// 3. Repository - its own concern
public interface IUserRepository
{
    Task<User> CreateAsync(User user);
    Task<User?> GetByEmailAsync(string email);
}

public class UserRepository : IUserRepository
{
    private readonly AppDbContext _context;

    public UserRepository(AppDbContext context) => _context = context;

    public async Task<User> CreateAsync(User user)
    {
        _context.Users.Add(user);
        await _context.SaveChangesAsync();
        return user;
    }
}

// 4. Email - its own concern
public interface IWelcomeEmailSender
{
    Task SendAsync(User user);
}

// 5. Coordinating service - orchestrates the workflow
public class UserService
{
    private readonly IUserValidator _validator;
    private readonly IPasswordHasher _hasher;
    private readonly IUserRepository _repository;
    private readonly IWelcomeEmailSender _emailSender;
    private readonly ILogger<UserService> _logger;

    public UserService(
        IUserValidator validator,
        IPasswordHasher hasher,
        IUserRepository repository,
        IWelcomeEmailSender emailSender,
        ILogger<UserService> logger)
    {
        _validator = validator;
        _hasher = hasher;
        _repository = repository;
        _emailSender = emailSender;
        _logger = logger;
    }

    public async Task<Result<User>> CreateUserAsync(CreateUserRequest request)
    {
        // Validate
        var validation = _validator.Validate(request);
        if (!validation.IsValid)
            return Result<User>.Failure(validation.Errors);

        // Check if exists
        var existing = await _repository.GetByEmailAsync(request.Email);
        if (existing != null)
            return Result<User>.Failure("Email already registered");

        // Create user
        var user = new User
        {
            Email = request.Email,
            Name = request.Name,
            PasswordHash = _hasher.Hash(request.Password)
        };

        await _repository.CreateAsync(user);

        // Send welcome email (fire and forget or queue)
        _ = _emailSender.SendAsync(user);

        _logger.LogInformation("User {Email} created", user.Email);

        return Result<User>.Success(user);
    }
}
```

**Testing Benefits:**

```csharp
[Fact]
public async Task CreateUser_WithInvalidEmail_ReturnsValidationError()
{
    // Arrange - can now test validation in isolation!
    var validator = new UserValidator();

    // Act
    var result = validator.Validate(new CreateUserRequest
    {
        Email = "invalid-email",
        Password = "password123",
        Name = "John"
    });

    // Assert
    Assert.False(result.IsValid);
    Assert.Contains("Invalid email", result.Errors);
}

[Fact]
public async Task CreateUser_WithValidData_CreatesUserAndSendsEmail()
{
    // Arrange - can mock all dependencies!
    var mockRepo = new Mock<IUserRepository>();
    var mockEmail = new Mock<IWelcomeEmailSender>();

    var service = new UserService(
        new UserValidator(),
        new BcryptPasswordHasher(),
        mockRepo.Object,
        mockEmail.Object,
        Mock.Of<ILogger<UserService>>());

    // Act
    var result = await service.CreateUserAsync(validRequest);

    // Assert
    mockRepo.Verify(r => r.CreateAsync(It.IsAny<User>()), Times.Once);
    mockEmail.Verify(e => e.SendAsync(It.IsAny<User>()), Times.Once);
}
```

### Communication Tactics

🎯 **Structure your answer**: First identify all responsibilities, then explain why it's problematic, then show the refactored solution. End with testing benefits.

💡 **Emphasize**: Each class now has ONE reason to change. Validator changes for validation rules. Repository changes for database concerns. Email sender changes for email concerns.

⚠️ **Avoid**: Don't over-engineer. For very simple apps, this level of separation might be overkill. Show you understand trade-offs.

---

## Question 3: Open/Closed Principle - Tricky Follow-up

### The Question
> "Show me how Open/Closed Principle gets violated in code. And here's the tricky part - how would you explain OCP violation to a junior developer who asks 'but I'm just adding a case to a switch, what's wrong with that?'"

### Key Points to Cover
- Common OCP violations (switch statements, if-else chains)
- Why modification is risky
- How to refactor to be extension-friendly
- Simple explanation for juniors

### Detailed Answer

**Common OCP Violations:**

```csharp
// ❌ VIOLATION 1 - Switch statement that keeps growing
public class PaymentProcessor
{
    public PaymentResult Process(Payment payment)
    {
        switch (payment.Method)
        {
            case "credit_card":
                return ProcessCreditCard(payment);
            case "paypal":
                return ProcessPayPal(payment);
            case "apple_pay":
                return ProcessApplePay(payment);
            // Every new payment method = modify this class
            // What if we need to add Google Pay, crypto, bank transfer...?
            default:
                throw new NotSupportedException();
        }
    }
}

// ❌ VIOLATION 2 - Type checking with if-else
public decimal CalculateShipping(Order order)
{
    if (order.Customer is PremiumCustomer)
        return 0; // Free shipping
    else if (order.Customer is StandardCustomer)
        return order.Weight * 5;
    else if (order.Customer is NewCustomer)
        return order.Weight * 3; // Discounted
    // More customer types = more modifications
    return order.Weight * 10;
}
```

**Why Modification is Risky:**

1. **Regression risk**: Touching working code can break it
2. **Testing burden**: Must re-test all branches
3. **Merge conflicts**: Multiple developers adding cases
4. **Code bloat**: File grows endlessly
5. **Violates SRP**: Class knows about all payment types

**OCP-Compliant Solution:**

```csharp
// ✅ Strategy pattern - Open for extension, closed for modification
public interface IPaymentProcessor
{
    string PaymentMethod { get; }
    PaymentResult Process(Payment payment);
}

public class CreditCardProcessor : IPaymentProcessor
{
    public string PaymentMethod => "credit_card";

    public PaymentResult Process(Payment payment)
    {
        // Credit card specific logic
        return new PaymentResult { Success = true };
    }
}

public class PayPalProcessor : IPaymentProcessor
{
    public string PaymentMethod => "paypal";

    public PaymentResult Process(Payment payment)
    {
        // PayPal specific logic
        return new PaymentResult { Success = true };
    }
}

// New payment method = new class, no modification to existing code!
public class CryptoProcessor : IPaymentProcessor
{
    public string PaymentMethod => "crypto";

    public PaymentResult Process(Payment payment)
    {
        // Crypto specific logic
        return new PaymentResult { Success = true };
    }
}

// Payment service that doesn't need modification
public class PaymentService
{
    private readonly IEnumerable<IPaymentProcessor> _processors;

    public PaymentService(IEnumerable<IPaymentProcessor> processors)
    {
        _processors = processors;
    }

    public PaymentResult Process(Payment payment)
    {
        var processor = _processors.FirstOrDefault(p => p.PaymentMethod == payment.Method)
            ?? throw new NotSupportedException($"Payment method {payment.Method} not supported");

        return processor.Process(payment);
    }
}

// Registration in DI
services.AddScoped<IPaymentProcessor, CreditCardProcessor>();
services.AddScoped<IPaymentProcessor, PayPalProcessor>();
services.AddScoped<IPaymentProcessor, CryptoProcessor>();
// Adding new = just add new registration, nothing else changes
```

**Explanation for Junior Developer:**

> "Think of it like a power strip. The bad approach is like having a power strip where you have to open it up and solder new outlets whenever you get a new device. The good approach is having a power strip with standard outlets - any device with a compatible plug works without modifying the strip.
>
> When you add a case to that switch:
> 1. You're touching code that's already working and tested
> 2. You risk breaking the existing cases
> 3. Other developers might be adding cases too, causing merge conflicts
> 4. The file keeps growing - where does it end?
>
> With the interface approach:
> 1. Existing code is untouched
> 2. You add a new file for your new payment type
> 3. No risk of breaking existing functionality
> 4. Easy to test in isolation"

### Communication Tactics

🎯 **Structure your answer**: Show the violation, explain the risks, show the solution, then give the simple explanation. The "power strip" analogy works well.

💡 **Emphasize**: OCP is about managing RISK. Working code is valuable. Every modification risks breaking it.

⚠️ **Avoid**: Don't be dogmatic. Small switch statements with stable options (e.g., days of week) don't need OCP treatment. Show practical judgment.

---

## Question 4: Liskov Substitution - Bird/Ostrich Problem

### The Question
> "Explain Liskov Substitution Principle using the classic Bird and Ostrich example. Why is this violation problematic and how would you fix it?"

### Key Points to Cover
- Clear LSP definition
- Why Ostrich : Bird violates LSP
- Real-world consequences
- Proper design solution

### Detailed Answer

**LSP Definition:**

> Objects of a superclass should be replaceable with objects of a subclass without affecting the correctness of the program.

In simpler terms: If it looks like a duck and quacks like a duck but needs batteries, you have the wrong abstraction.

**The Classic Violation:**

```csharp
// ❌ LSP VIOLATION
public class Bird
{
    public virtual void Fly()
    {
        Console.WriteLine("Flying through the sky!");
    }
}

public class Sparrow : Bird
{
    public override void Fly()
    {
        Console.WriteLine("Sparrow flying!"); // ✅ Works fine
    }
}

public class Ostrich : Bird
{
    public override void Fly()
    {
        throw new NotSupportedException("Ostriches can't fly!"); // ❌ VIOLATION!
    }
}

// Code that expects Bird to work
public class BirdMigrationService
{
    public void MigrateBirds(IEnumerable<Bird> birds, Location destination)
    {
        foreach (var bird in birds)
        {
            bird.Fly(); // 💥 Crashes when it hits an Ostrich!
        }
    }
}
```

**Why This Is Problematic:**

1. **Breaks contracts**: Clients expect `Fly()` to work, not throw
2. **Requires special handling**: Now every caller needs to check type
3. **Violates expectations**: A Bird should behave like a Bird
4. **Makes polymorphism useless**: Can't treat all birds uniformly

```csharp
// Now callers must do this - defeats the purpose of polymorphism
foreach (var bird in birds)
{
    if (bird is not Ostrich) // Leaky abstraction!
    {
        bird.Fly();
    }
}
```

**Proper Design Solutions:**

```csharp
// ✅ SOLUTION 1: Separate capabilities via interfaces
public interface IFlyable
{
    void Fly();
}

public interface IWalkable
{
    void Walk();
}

public abstract class Bird : IWalkable
{
    public abstract void Walk();
    public abstract void Eat();
}

public class Sparrow : Bird, IFlyable
{
    public override void Walk() => Console.WriteLine("Hopping around");
    public override void Eat() => Console.WriteLine("Eating seeds");
    public void Fly() => Console.WriteLine("Flying!"); // Only flying birds implement this
}

public class Ostrich : Bird
{
    public override void Walk() => Console.WriteLine("Running fast!");
    public override void Eat() => Console.WriteLine("Eating plants");
    // No Fly() - ostriches don't pretend to fly
}

// Migration service only works with flyable birds
public class BirdMigrationService
{
    public void MigrateBirds(IEnumerable<IFlyable> flyingBirds, Location destination)
    {
        foreach (var bird in flyingBirds)
        {
            bird.Fly(); // ✅ All birds here can definitely fly
        }
    }
}
```

```csharp
// ✅ SOLUTION 2: Composition over inheritance
public interface IMovementBehavior
{
    void Move(Location destination);
}

public class FlyingBehavior : IMovementBehavior
{
    public void Move(Location destination)
    {
        Console.WriteLine($"Flying to {destination}");
    }
}

public class RunningBehavior : IMovementBehavior
{
    public void Move(Location destination)
    {
        Console.WriteLine($"Running to {destination}");
    }
}

public class Bird
{
    private readonly IMovementBehavior _movementBehavior;

    public Bird(IMovementBehavior movementBehavior)
    {
        _movementBehavior = movementBehavior;
    }

    public void MoveTo(Location destination)
    {
        _movementBehavior.Move(destination);
    }
}

// Usage
var sparrow = new Bird(new FlyingBehavior());
var ostrich = new Bird(new RunningBehavior());
```

**Real-World Example in .NET:**

```csharp
// LSP violation in practice - ReadOnlyCollection
public class MyCollection<T> : List<T>
{
    public new void Add(T item)
    {
        throw new NotSupportedException("Collection is read-only");
        // ❌ Violates LSP - List<T> promises Add() works
    }
}

// Better approach - use appropriate base type
public class MyCollection<T> : IReadOnlyList<T>
{
    // Only promises read operations - no violation
}
```

### Communication Tactics

🎯 **Structure your answer**: State LSP clearly, show the Bird/Ostrich violation, explain why it breaks things, then show 2 solutions (interfaces and composition).

💡 **Emphasize**: LSP is about keeping PROMISES. If your base class promises something, all subclasses must deliver.

⚠️ **Avoid**: Don't say "never use inheritance." Instead, emphasize careful design of base class contracts.

---

## Question 5: SOLID and Testability

### The Question
> "How do SOLID principles improve the testability of code? Can you show me a before/after example?"

### Key Points to Cover
- How each principle enables testing
- Concrete before/after example
- Mocking and dependency injection
- Test isolation benefits

### Detailed Answer

**How Each Principle Helps Testing:**

| Principle | How It Helps Testing |
|-----------|---------------------|
| **SRP** | Small classes = small tests, one reason to test |
| **OCP** | New behavior = new test file, not changing existing tests |
| **LSP** | Can use test doubles confidently |
| **ISP** | Mock only what you need |
| **DIP** | Inject mocks instead of real dependencies |

**Before SOLID - Hard to Test:**

```csharp
// ❌ UNTESTABLE CODE
public class OrderProcessor
{
    public OrderResult Process(OrderRequest request)
    {
        // 1. Direct database dependency
        using var conn = new SqlConnection("Server=prod;Database=Orders...");
        var customer = GetCustomer(conn, request.CustomerId);

        // 2. Hardcoded business rules
        if (customer.CreditScore < 500)
            return OrderResult.Rejected("Low credit score");

        // 3. Direct external service call
        var paymentResult = new PaymentGateway("live-api-key")
            .Charge(customer.CardNumber, request.Total);

        if (!paymentResult.Success)
            return OrderResult.Rejected(paymentResult.Error);

        // 4. Direct file system access
        File.AppendAllText("C:\\logs\\orders.txt",
            $"{DateTime.Now}: Order processed for {customer.Email}");

        // 5. Direct email sending
        new SmtpClient("smtp.company.com")
            .Send("orders@company.com", customer.Email, "Order Confirmed", "...");

        return OrderResult.Success();
    }
}

// To test this, you would need:
// - Real database connection
// - Real payment gateway (and real money!)
// - Write access to C:\logs
// - SMTP server running
// This is an INTEGRATION test, not a unit test!
```

**After SOLID - Easy to Test:**

```csharp
// ✅ TESTABLE CODE
public interface ICustomerRepository
{
    Task<Customer> GetByIdAsync(int id);
}

public interface ICreditChecker
{
    bool IsApproved(Customer customer);
}

public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(string cardNumber, decimal amount);
}

public interface IOrderNotifier
{
    Task NotifyOrderConfirmedAsync(Customer customer, Order order);
}

public class OrderProcessor
{
    private readonly ICustomerRepository _customerRepo;
    private readonly ICreditChecker _creditChecker;
    private readonly IPaymentGateway _paymentGateway;
    private readonly IOrderNotifier _notifier;
    private readonly ILogger<OrderProcessor> _logger;

    public OrderProcessor(
        ICustomerRepository customerRepo,
        ICreditChecker creditChecker,
        IPaymentGateway paymentGateway,
        IOrderNotifier notifier,
        ILogger<OrderProcessor> logger)
    {
        _customerRepo = customerRepo;
        _creditChecker = creditChecker;
        _paymentGateway = paymentGateway;
        _notifier = notifier;
        _logger = logger;
    }

    public async Task<OrderResult> ProcessAsync(OrderRequest request)
    {
        var customer = await _customerRepo.GetByIdAsync(request.CustomerId);

        if (!_creditChecker.IsApproved(customer))
            return OrderResult.Rejected("Credit not approved");

        var paymentResult = await _paymentGateway.ChargeAsync(
            customer.CardNumber, request.Total);

        if (!paymentResult.Success)
            return OrderResult.Rejected(paymentResult.Error);

        _logger.LogInformation("Order processed for {Email}", customer.Email);

        await _notifier.NotifyOrderConfirmedAsync(customer, request.ToOrder());

        return OrderResult.Success();
    }
}
```

**Now Testing is Easy:**

```csharp
public class OrderProcessorTests
{
    private readonly Mock<ICustomerRepository> _customerRepoMock;
    private readonly Mock<ICreditChecker> _creditCheckerMock;
    private readonly Mock<IPaymentGateway> _paymentGatewayMock;
    private readonly Mock<IOrderNotifier> _notifierMock;
    private readonly OrderProcessor _processor;

    public OrderProcessorTests()
    {
        _customerRepoMock = new Mock<ICustomerRepository>();
        _creditCheckerMock = new Mock<ICreditChecker>();
        _paymentGatewayMock = new Mock<IPaymentGateway>();
        _notifierMock = new Mock<IOrderNotifier>();

        _processor = new OrderProcessor(
            _customerRepoMock.Object,
            _creditCheckerMock.Object,
            _paymentGatewayMock.Object,
            _notifierMock.Object,
            Mock.Of<ILogger<OrderProcessor>>());
    }

    [Fact]
    public async Task Process_WhenCreditNotApproved_ReturnsRejected()
    {
        // Arrange
        var customer = new Customer { Id = 1, Email = "test@test.com" };
        _customerRepoMock.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(customer);
        _creditCheckerMock.Setup(c => c.IsApproved(customer)).Returns(false);

        // Act
        var result = await _processor.ProcessAsync(new OrderRequest { CustomerId = 1 });

        // Assert
        Assert.False(result.IsSuccess);
        Assert.Equal("Credit not approved", result.Error);

        // Verify payment was never attempted
        _paymentGatewayMock.Verify(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()),
            Times.Never);
    }

    [Fact]
    public async Task Process_WhenPaymentFails_ReturnsRejected()
    {
        // Arrange
        var customer = new Customer { Id = 1, CardNumber = "4111..." };
        _customerRepoMock.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(customer);
        _creditCheckerMock.Setup(c => c.IsApproved(customer)).Returns(true);
        _paymentGatewayMock.Setup(p => p.ChargeAsync(It.IsAny<string>(), 100))
            .ReturnsAsync(new PaymentResult { Success = false, Error = "Insufficient funds" });

        // Act
        var result = await _processor.ProcessAsync(new OrderRequest { CustomerId = 1, Total = 100 });

        // Assert
        Assert.False(result.IsSuccess);
        Assert.Equal("Insufficient funds", result.Error);

        // Verify notification was never sent
        _notifierMock.Verify(n => n.NotifyOrderConfirmedAsync(It.IsAny<Customer>(), It.IsAny<Order>()),
            Times.Never);
    }

    [Fact]
    public async Task Process_WhenSuccessful_NotifiesCustomer()
    {
        // Arrange
        var customer = new Customer { Id = 1, CardNumber = "4111..." };
        _customerRepoMock.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(customer);
        _creditCheckerMock.Setup(c => c.IsApproved(customer)).Returns(true);
        _paymentGatewayMock.Setup(p => p.ChargeAsync(It.IsAny<string>(), It.IsAny<decimal>()))
            .ReturnsAsync(new PaymentResult { Success = true });

        // Act
        var result = await _processor.ProcessAsync(new OrderRequest { CustomerId = 1, Total = 100 });

        // Assert
        Assert.True(result.IsSuccess);
        _notifierMock.Verify(n => n.NotifyOrderConfirmedAsync(customer, It.IsAny<Order>()),
            Times.Once);
    }
}
```

**Key Benefits:**

1. **Fast tests**: No I/O, database, or network calls
2. **Isolated tests**: Each test controls all dependencies
3. **Deterministic**: Same input = same output always
4. **Easy to write**: Clear what to mock, clear what to assert
5. **Documents behavior**: Tests show how components interact

### Communication Tactics

🎯 **Structure your answer**: Show the untestable before code, explain why it's untestable, then show the SOLID version with actual tests.

💡 **Emphasize**: DIP is the key enabler - by depending on abstractions, you can inject test doubles. But all principles contribute.

⚠️ **Avoid**: Don't imply 100% test coverage is always the goal. Focus on testing business logic, not infrastructure wiring.

---

## Quick Review - Key Takeaways

| Question | Key Point |
|----------|-----------|
| SOLID Walkthrough | One sentence definition + violation + fix for each |
| SRP Code Review | Identify all responsibilities, show testing benefits |
| OCP Tricky Question | "Power strip" analogy, risk of modification |
| LSP Bird/Ostrich | Subclasses must keep promises, use interfaces |
| Testability | DIP enables mocking, all principles help isolation |
