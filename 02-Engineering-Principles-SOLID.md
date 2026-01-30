# Engineering Principles & SOLID

> ⚠️ **HIGH PRIORITY**: These principles are fundamental to professional software development

---

## SOLID Principles

SOLID is an acronym for five design principles that make software more maintainable, flexible, and scalable.

---

## S - Single Responsibility Principle (SRP)

### Definition
**A class should have only one reason to change.**

Each class should have only one responsibility or job. If a class has multiple responsibilities, changes to one responsibility may affect or break the other.

### ❌ Bad Example

```csharp
// This class has MULTIPLE responsibilities
public class UserService
{
    public void CreateUser(User user)
    {
        // 1. Validation logic
        if (string.IsNullOrEmpty(user.Email))
            throw new ArgumentException("Email required");
        if (!user.Email.Contains("@"))
            throw new ArgumentException("Invalid email");

        // 2. Database logic
        using var connection = new SqlConnection(_connectionString);
        connection.Open();
        var command = new SqlCommand("INSERT INTO Users...", connection);
        command.ExecuteNonQuery();

        // 3. Email sending logic
        var smtpClient = new SmtpClient("smtp.server.com");
        var message = new MailMessage("noreply@app.com", user.Email);
        message.Subject = "Welcome!";
        smtpClient.Send(message);

        // 4. Logging logic
        File.AppendAllText("log.txt", $"User created: {user.Email}");
    }
}
```

### ✅ Good Example

```csharp
// Each class has ONE responsibility

public class UserValidator
{
    public ValidationResult Validate(User user)
    {
        var errors = new List<string>();

        if (string.IsNullOrEmpty(user.Email))
            errors.Add("Email required");
        else if (!user.Email.Contains("@"))
            errors.Add("Invalid email format");

        return new ValidationResult(errors);
    }
}

public class UserRepository
{
    private readonly IDbConnection _connection;

    public UserRepository(IDbConnection connection)
    {
        _connection = connection;
    }

    public async Task<int> CreateAsync(User user)
    {
        return await _connection.ExecuteAsync(
            "INSERT INTO Users (Email, Name) VALUES (@Email, @Name)",
            user);
    }
}

public class EmailService
{
    private readonly ISmtpClient _smtpClient;

    public async Task SendWelcomeEmailAsync(string email)
    {
        var message = new EmailMessage
        {
            To = email,
            Subject = "Welcome!",
            Body = "Welcome to our platform"
        };
        await _smtpClient.SendAsync(message);
    }
}

public class UserService
{
    private readonly UserValidator _validator;
    private readonly UserRepository _repository;
    private readonly EmailService _emailService;
    private readonly ILogger<UserService> _logger;

    public UserService(
        UserValidator validator,
        UserRepository repository,
        EmailService emailService,
        ILogger<UserService> logger)
    {
        _validator = validator;
        _repository = repository;
        _emailService = emailService;
        _logger = logger;
    }

    public async Task<Result> CreateUserAsync(User user)
    {
        // Orchestrates, but doesn't implement each concern
        var validation = _validator.Validate(user);
        if (!validation.IsValid)
            return Result.Failure(validation.Errors);

        var userId = await _repository.CreateAsync(user);

        await _emailService.SendWelcomeEmailAsync(user.Email);

        _logger.LogInformation("User created: {UserId}", userId);

        return Result.Success(userId);
    }
}
```

### Interview Tip
> "If I need to change validation rules, I only modify `UserValidator`. If I change how emails are sent, only `EmailService` changes. Each class has one reason to change."

---

## O - Open/Closed Principle (OCP)

### Definition
**Software entities should be open for extension but closed for modification.**

You should be able to add new functionality without changing existing code.

### ❌ Bad Example

```csharp
public class PaymentProcessor
{
    public void ProcessPayment(string paymentType, decimal amount)
    {
        // Every new payment type requires modifying this class
        if (paymentType == "CreditCard")
        {
            // Credit card processing
            Console.WriteLine($"Processing credit card payment: {amount}");
        }
        else if (paymentType == "PayPal")
        {
            // PayPal processing
            Console.WriteLine($"Processing PayPal payment: {amount}");
        }
        else if (paymentType == "Crypto")
        {
            // Added later - had to modify existing class!
            Console.WriteLine($"Processing crypto payment: {amount}");
        }
        // Adding new payment type = changing this class = risk of breaking existing code
    }
}
```

### ✅ Good Example

```csharp
// Abstraction
public interface IPaymentHandler
{
    string PaymentType { get; }
    Task<PaymentResult> ProcessAsync(decimal amount);
}

// Implementations - add new ones without changing existing code
public class CreditCardPaymentHandler : IPaymentHandler
{
    public string PaymentType => "CreditCard";

    public async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        // Credit card specific logic
        await Task.Delay(100);
        return new PaymentResult { Success = true };
    }
}

public class PayPalPaymentHandler : IPaymentHandler
{
    public string PaymentType => "PayPal";

    public async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        // PayPal specific logic
        await Task.Delay(100);
        return new PaymentResult { Success = true };
    }
}

// NEW: Adding crypto doesn't require changing ANY existing code!
public class CryptoPaymentHandler : IPaymentHandler
{
    public string PaymentType => "Crypto";

    public async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        await Task.Delay(100);
        return new PaymentResult { Success = true };
    }
}

// Processor is CLOSED for modification, OPEN for extension
public class PaymentProcessor
{
    private readonly Dictionary<string, IPaymentHandler> _handlers;

    public PaymentProcessor(IEnumerable<IPaymentHandler> handlers)
    {
        _handlers = handlers.ToDictionary(h => h.PaymentType);
    }

    public async Task<PaymentResult> ProcessPaymentAsync(string paymentType, decimal amount)
    {
        if (!_handlers.TryGetValue(paymentType, out var handler))
            throw new NotSupportedException($"Payment type {paymentType} not supported");

        return await handler.ProcessAsync(amount);
    }
}

// Registration in DI
services.AddScoped<IPaymentHandler, CreditCardPaymentHandler>();
services.AddScoped<IPaymentHandler, PayPalPaymentHandler>();
services.AddScoped<IPaymentHandler, CryptoPaymentHandler>();
services.AddScoped<PaymentProcessor>();
```

---

## L - Liskov Substitution Principle (LSP)

### Definition
**Objects of a superclass should be replaceable with objects of its subclasses without breaking the application.**

Derived classes must be substitutable for their base classes.

### ❌ Bad Example

```csharp
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }

    public int CalculateArea() => Width * Height;
}

public class Square : Rectangle
{
    // Square violates LSP by changing expected behavior
    public override int Width
    {
        get => base.Width;
        set
        {
            base.Width = value;
            base.Height = value; // Unexpected side effect!
        }
    }

    public override int Height
    {
        get => base.Height;
        set
        {
            base.Height = value;
            base.Width = value; // Unexpected side effect!
        }
    }
}

// This code breaks with Square!
public void ProcessRectangle(Rectangle rect)
{
    rect.Width = 5;
    rect.Height = 10;

    // Expectation: Area = 50
    // With Square: Area = 100 (because both became 10)
    Console.WriteLine(rect.CalculateArea()); // LSP violated!
}
```

### ✅ Good Example

```csharp
// Use abstraction instead of inheritance when behaviors differ
public interface IShape
{
    int CalculateArea();
}

public class Rectangle : IShape
{
    public int Width { get; }
    public int Height { get; }

    public Rectangle(int width, int height)
    {
        Width = width;
        Height = height;
    }

    public int CalculateArea() => Width * Height;
}

public class Square : IShape
{
    public int Side { get; }

    public Square(int side)
    {
        Side = side;
    }

    public int CalculateArea() => Side * Side;
}

// Both work correctly through the interface
public void ProcessShape(IShape shape)
{
    Console.WriteLine(shape.CalculateArea()); // Always works as expected
}
```

### Real-World LSP Example

```csharp
public interface INotificationSender
{
    Task SendAsync(Notification notification);
}

public class EmailSender : INotificationSender
{
    public async Task SendAsync(Notification notification)
    {
        // Works as expected
        await SendEmailAsync(notification.Recipient, notification.Message);
    }
}

public class SmsSender : INotificationSender
{
    public async Task SendAsync(Notification notification)
    {
        // Works as expected
        await SendSmsAsync(notification.Recipient, notification.Message);
    }
}

// ❌ BAD: This violates LSP
public class FakeSender : INotificationSender
{
    public Task SendAsync(Notification notification)
    {
        // Silently does nothing - violates the contract!
        return Task.CompletedTask;
    }
}

// ✅ GOOD: If you need a no-op, make it explicit
public class NullSender : INotificationSender
{
    public Task SendAsync(Notification notification)
    {
        // Explicitly documented as a no-op for testing/development
        Console.WriteLine($"[NULL SENDER] Would send: {notification.Message}");
        return Task.CompletedTask;
    }
}
```

---

## I - Interface Segregation Principle (ISP)

### Definition
**Clients should not be forced to depend on interfaces they do not use.**

Many specific interfaces are better than one general-purpose interface.

### ❌ Bad Example

```csharp
// Fat interface - forces all implementers to have all methods
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
    void AttendMeeting();
    void SubmitTimesheet();
    void TakeVacation();
}

// Robot can't eat, sleep, or take vacation!
public class RobotWorker : IWorker
{
    public void Work() { /* OK */ }

    public void Eat() => throw new NotSupportedException(); // Forced to implement!
    public void Sleep() => throw new NotSupportedException();
    public void AttendMeeting() { /* OK */ }
    public void SubmitTimesheet() => throw new NotSupportedException();
    public void TakeVacation() => throw new NotSupportedException();
}
```

### ✅ Good Example

```csharp
// Segregated interfaces - implement only what you need
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
    void TakeVacation();
}

public interface IMeetingAttendee
{
    void AttendMeeting();
}

public interface ITimesheetSubmitter
{
    void SubmitTimesheet();
}

// Human implements all relevant interfaces
public class HumanWorker : IWorkable, IFeedable, IRestable, IMeetingAttendee, ITimesheetSubmitter
{
    public void Work() { /* implementation */ }
    public void Eat() { /* implementation */ }
    public void Sleep() { /* implementation */ }
    public void TakeVacation() { /* implementation */ }
    public void AttendMeeting() { /* implementation */ }
    public void SubmitTimesheet() { /* implementation */ }
}

// Robot only implements what it can do
public class RobotWorker : IWorkable, IMeetingAttendee
{
    public void Work() { /* implementation */ }
    public void AttendMeeting() { /* via video feed */ }
}
```

### Practical ISP Example

```csharp
// ❌ BAD: One big repository interface
public interface IRepository<T>
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task CreateAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
    Task<IEnumerable<T>> SearchAsync(string query);
    Task BulkInsertAsync(IEnumerable<T> entities);
    Task<int> CountAsync();
}

// Some repositories don't need all these methods!

// ✅ GOOD: Segregated interfaces
public interface IReadRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
}

public interface IWriteRepository<T>
{
    Task CreateAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
}

public interface ISearchableRepository<T>
{
    Task<IEnumerable<T>> SearchAsync(string query);
}

public interface IBulkRepository<T>
{
    Task BulkInsertAsync(IEnumerable<T> entities);
}

// Read-only repository for reports
public class ReportRepository : IReadRepository<Report>
{
    public Task<Report?> GetByIdAsync(int id) { /* ... */ }
    public Task<IEnumerable<Report>> GetAllAsync() { /* ... */ }
    // No write methods needed!
}

// Full CRUD repository
public class UserRepository : IReadRepository<User>, IWriteRepository<User>, ISearchableRepository<User>
{
    // Implements only what's needed
}
```

---

## D - Dependency Inversion Principle (DIP)

### Definition
**High-level modules should not depend on low-level modules. Both should depend on abstractions.**

Also: Abstractions should not depend on details. Details should depend on abstractions.

### ❌ Bad Example

```csharp
// High-level module directly depends on low-level module
public class OrderService
{
    // Direct dependency on concrete implementation
    private readonly SqlServerDatabase _database = new SqlServerDatabase();
    private readonly SmtpEmailSender _emailSender = new SmtpEmailSender();

    public void CreateOrder(Order order)
    {
        // Tightly coupled - can't change database or email provider without changing this class
        _database.Save(order);
        _emailSender.Send(order.CustomerEmail, "Order confirmed");
    }
}

// Problems:
// 1. Can't unit test without real database and SMTP server
// 2. Can't switch to PostgreSQL or SendGrid without changing OrderService
// 3. OrderService is coupled to infrastructure details
```

### ✅ Good Example

```csharp
// Abstractions (defined at high level)
public interface IOrderRepository
{
    Task SaveAsync(Order order);
}

public interface INotificationService
{
    Task SendOrderConfirmationAsync(string email, Order order);
}

// High-level module depends on abstractions
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly INotificationService _notifications;

    // Dependencies injected through constructor
    public OrderService(IOrderRepository repository, INotificationService notifications)
    {
        _repository = repository;
        _notifications = notifications;
    }

    public async Task CreateOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);
        await _notifications.SendOrderConfirmationAsync(order.CustomerEmail, order);
    }
}

// Low-level modules implement abstractions
public class SqlServerOrderRepository : IOrderRepository
{
    private readonly DbContext _context;

    public SqlServerOrderRepository(DbContext context)
    {
        _context = context;
    }

    public async Task SaveAsync(Order order)
    {
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
    }
}

public class EmailNotificationService : INotificationService
{
    private readonly IEmailClient _emailClient;

    public EmailNotificationService(IEmailClient emailClient)
    {
        _emailClient = emailClient;
    }

    public async Task SendOrderConfirmationAsync(string email, Order order)
    {
        await _emailClient.SendAsync(email, "Order Confirmed", $"Order {order.Id} confirmed");
    }
}

// DI Container registration
services.AddScoped<IOrderRepository, SqlServerOrderRepository>();
services.AddScoped<INotificationService, EmailNotificationService>();
services.AddScoped<OrderService>();
```

### Testing Benefits of DIP

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task CreateOrder_SavesOrderAndSendsNotification()
    {
        // Arrange - use mocks instead of real implementations
        var mockRepo = new Mock<IOrderRepository>();
        var mockNotifications = new Mock<INotificationService>();

        var service = new OrderService(mockRepo.Object, mockNotifications.Object);
        var order = new Order { Id = 1, CustomerEmail = "test@test.com" };

        // Act
        await service.CreateOrderAsync(order);

        // Assert
        mockRepo.Verify(r => r.SaveAsync(order), Times.Once);
        mockNotifications.Verify(n =>
            n.SendOrderConfirmationAsync("test@test.com", order), Times.Once);
    }
}
```

---

## Other Important Principles

### DRY - Don't Repeat Yourself

**Every piece of knowledge should have a single, unambiguous representation in the system.**

```csharp
// ❌ BAD: Repeated validation logic
public class UserController
{
    public IActionResult Create(UserDto dto)
    {
        if (string.IsNullOrEmpty(dto.Email) || !dto.Email.Contains("@"))
            return BadRequest("Invalid email");
        // ...
    }

    public IActionResult Update(UserDto dto)
    {
        if (string.IsNullOrEmpty(dto.Email) || !dto.Email.Contains("@"))
            return BadRequest("Invalid email");
        // Same validation repeated!
    }
}

// ✅ GOOD: Single source of truth
public class EmailValidator
{
    public bool IsValid(string email)
        => !string.IsNullOrEmpty(email) && email.Contains("@");
}

// Or use FluentValidation
public class UserDtoValidator : AbstractValidator<UserDto>
{
    public UserDtoValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress();
    }
}
```

### KISS - Keep It Simple, Stupid

**The simplest solution that works is usually the best.**

```csharp
// ❌ Overcomplicated
public bool IsEven(int number)
{
    return Math.Abs(number) % 2 == 0 ? true : false;
}

// ✅ Simple
public bool IsEven(int number) => number % 2 == 0;
```

### YAGNI - You Aren't Gonna Need It

**Don't implement functionality until you actually need it.**

```csharp
// ❌ BAD: Adding features "just in case"
public class UserService
{
    public User GetUser(int id) { /* ... */ }
    public User GetUserByEmail(string email) { /* ... */ }
    public User GetUserByPhone(string phone) { /* Maybe we'll need this? */ }
    public User GetUserByNickname(string nickname) { /* What if? */ }
    public User GetUserBySocialSecurityNumber(string ssn) { /* Could be useful */ }
}

// ✅ GOOD: Only what's needed now
public class UserService
{
    public User GetUser(int id) { /* ... */ }
    public User GetUserByEmail(string email) { /* ... */ }
    // Add more when actually needed
}
```

### Separation of Concerns (SoC)

**Different concerns should be handled by different parts of the system.**

```csharp
// ✅ GOOD: Concerns are separated

// Presentation Layer - handles HTTP
[ApiController]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;

    [HttpPost]
    public async Task<IActionResult> Create(OrderDto dto)
    {
        var result = await _orderService.CreateAsync(dto);
        return result.IsSuccess ? Ok(result.Value) : BadRequest(result.Error);
    }
}

// Application Layer - orchestrates business logic
public class OrderService : IOrderService
{
    public async Task<Result<Order>> CreateAsync(OrderDto dto)
    {
        // Orchestration logic
    }
}

// Domain Layer - business rules
public class Order
{
    public void AddItem(OrderItem item)
    {
        // Business rules here
    }
}

// Infrastructure Layer - data access
public class OrderRepository : IOrderRepository
{
    public async Task SaveAsync(Order order)
    {
        // Database operations
    }
}
```

---

## Quick Reference

| Principle | Remember |
|-----------|----------|
| **S**RP | One class = One reason to change |
| **O**CP | Extend, don't modify |
| **L**SP | Derived classes must be substitutable |
| **I**SP | Many small interfaces > One big interface |
| **D**IP | Depend on abstractions, not concretions |
| **DRY** | Single source of truth |
| **KISS** | Simple > Complex |
| **YAGNI** | Build what you need now |

---

## Interview Q&A

**Q: Which SOLID principle do you think is most important?**
> A: They're all interconnected, but I find the **Dependency Inversion Principle** particularly impactful because it enables testability and flexibility. When you depend on abstractions, you can easily swap implementations, mock dependencies for testing, and adapt to changing requirements.

**Q: How do SOLID principles relate to each other?**
> A: They work together:
> - Following **SRP** often leads to more focused interfaces (**ISP**)
> - **OCP** is typically achieved through polymorphism enabled by **DIP**
> - Proper **LSP** compliance makes **OCP** possible
> - **DIP** facilitates all other principles by enabling loose coupling

**Q: Can you have too much abstraction?**
> A: Yes, over-engineering is a real problem. Apply YAGNI—only abstract when you have a concrete need. If you only ever have one implementation, you might not need an interface yet. Let patterns emerge from actual requirements, not hypothetical ones.
