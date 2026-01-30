# Design Patterns - Interview Questions

> 5 real-world interview questions with detailed answers and communication tactics

---

## Question 1: Thread-Safe Singleton

### The Question
> "Implement a thread-safe Singleton in C#. Explain why thread safety matters and what approaches you can use."

### Key Points to Cover
- Why thread safety is important
- Multiple implementation approaches
- Trade-offs of each approach
- Modern .NET preferred approach

### Detailed Answer

**Why Thread Safety Matters:**

```csharp
// ❌ NOT THREAD-SAFE - Race condition!
public class UnsafeSingleton
{
    private static UnsafeSingleton _instance;

    public static UnsafeSingleton Instance
    {
        get
        {
            // Two threads can both see _instance as null
            // and both create new instances!
            if (_instance == null)
            {
                _instance = new UnsafeSingleton();
            }
            return _instance;
        }
    }
}
```

**Approach 1: Lock (Simple but slightly slower):**

```csharp
public class LockSingleton
{
    private static LockSingleton _instance;
    private static readonly object _lock = new object();

    private LockSingleton() { }

    public static LockSingleton Instance
    {
        get
        {
            lock (_lock)
            {
                if (_instance == null)
                {
                    _instance = new LockSingleton();
                }
                return _instance;
            }
        }
    }
}
```

**Approach 2: Double-Check Locking (Better performance):**

```csharp
public class DoubleCheckSingleton
{
    private static volatile DoubleCheckSingleton _instance;
    private static readonly object _lock = new object();

    private DoubleCheckSingleton() { }

    public static DoubleCheckSingleton Instance
    {
        get
        {
            // First check - no lock needed if already initialized
            if (_instance == null)
            {
                lock (_lock)
                {
                    // Second check - in case another thread initialized while waiting
                    if (_instance == null)
                    {
                        _instance = new DoubleCheckSingleton();
                    }
                }
            }
            return _instance;
        }
    }
}
```

**Approach 3: Lazy<T> (Recommended - .NET 4+):**

```csharp
public class LazySingleton
{
    // Lazy<T> handles thread safety automatically
    private static readonly Lazy<LazySingleton> _lazy =
        new Lazy<LazySingleton>(() => new LazySingleton());

    private LazySingleton()
    {
        // Expensive initialization here
        Console.WriteLine("Singleton initialized");
    }

    public static LazySingleton Instance => _lazy.Value;

    public void DoWork()
    {
        Console.WriteLine("Working...");
    }
}
```

**Approach 4: Static Constructor (Simplest):**

```csharp
public class StaticSingleton
{
    // Static constructor is guaranteed thread-safe by CLR
    private static readonly StaticSingleton _instance = new StaticSingleton();

    // Explicit static constructor to tell C# compiler
    // not to mark type as beforefieldinit
    static StaticSingleton() { }

    private StaticSingleton()
    {
        Console.WriteLine("Singleton initialized");
    }

    public static StaticSingleton Instance => _instance;
}
```

**Real-World Example - Configuration Manager:**

```csharp
public class ConfigurationManager
{
    private static readonly Lazy<ConfigurationManager> _instance =
        new Lazy<ConfigurationManager>(() => new ConfigurationManager());

    private readonly Dictionary<string, string> _settings;

    private ConfigurationManager()
    {
        // Load configuration from file/environment
        _settings = new Dictionary<string, string>();
        LoadFromEnvironment();
        LoadFromFile();
    }

    public static ConfigurationManager Instance => _instance.Value;

    public string GetSetting(string key) =>
        _settings.TryGetValue(key, out var value) ? value : null;

    private void LoadFromEnvironment()
    {
        foreach (DictionaryEntry env in Environment.GetEnvironmentVariables())
        {
            _settings[env.Key.ToString()] = env.Value?.ToString();
        }
    }

    private void LoadFromFile()
    {
        if (File.Exists("appsettings.json"))
        {
            var json = File.ReadAllText("appsettings.json");
            // Parse and add to _settings
        }
    }
}
```

**Comparison:**

| Approach | Thread-Safe | Lazy | Complexity |
|----------|-------------|------|------------|
| Simple (unsafe) | ❌ | ✅ | Low |
| Lock | ✅ | ✅ | Medium |
| Double-Check | ✅ | ✅ | High |
| Lazy<T> | ✅ | ✅ | Low |
| Static Constructor | ✅ | ❌ | Low |

### Communication Tactics

🎯 **Structure your answer**: Explain why thread safety matters first, then show 2-3 approaches, recommend Lazy<T>, and give a real example.

💡 **Emphasize**: In modern .NET, use `Lazy<T>` - it's clean, efficient, and handles thread safety. Mention you understand the alternatives.

⚠️ **Avoid**: Don't just show one approach. Interviewers want to see you understand the options and can choose appropriately.

---

## Question 2: Factory vs Builder Pattern

### The Question
> "When would you choose Factory pattern versus Builder pattern? Can you give me examples of each and explain your decision criteria?"

### Key Points to Cover
- When each pattern applies
- Key differences
- Real-world examples
- Decision criteria

### Detailed Answer

**Factory Pattern - When to Use:**

Use Factory when:
- Object creation logic is complex
- You need to return different types based on input
- You want to hide instantiation details
- Objects have few constructor parameters

```csharp
// Factory Pattern - Creating different payment processors
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(decimal amount);
}

public class CreditCardProcessor : IPaymentProcessor
{
    public async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        // Credit card processing logic
        return new PaymentResult { Success = true };
    }
}

public class PayPalProcessor : IPaymentProcessor
{
    public async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        // PayPal processing logic
        return new PaymentResult { Success = true };
    }
}

public class CryptoProcessor : IPaymentProcessor
{
    public async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        // Crypto processing logic
        return new PaymentResult { Success = true };
    }
}

// Factory
public class PaymentProcessorFactory
{
    public IPaymentProcessor Create(string paymentMethod)
    {
        return paymentMethod.ToLower() switch
        {
            "creditcard" => new CreditCardProcessor(),
            "paypal" => new PayPalProcessor(),
            "crypto" => new CryptoProcessor(),
            _ => throw new ArgumentException($"Unknown payment method: {paymentMethod}")
        };
    }
}

// Usage
var factory = new PaymentProcessorFactory();
var processor = factory.Create("paypal");
await processor.ProcessAsync(100);
```

**Builder Pattern - When to Use:**

Use Builder when:
- Object has many optional parameters
- Object construction is multi-step
- You want fluent, readable construction
- Object should be immutable after creation

```csharp
// Builder Pattern - Creating complex objects with many options
public class EmailMessage
{
    public string From { get; }
    public List<string> To { get; }
    public List<string> Cc { get; }
    public List<string> Bcc { get; }
    public string Subject { get; }
    public string Body { get; }
    public bool IsHtml { get; }
    public List<Attachment> Attachments { get; }
    public Priority Priority { get; }

    // Private constructor - can only be created via builder
    private EmailMessage(EmailMessageBuilder builder)
    {
        From = builder.From;
        To = builder.To;
        Cc = builder.Cc;
        Bcc = builder.Bcc;
        Subject = builder.Subject;
        Body = builder.Body;
        IsHtml = builder.IsHtml;
        Attachments = builder.Attachments;
        Priority = builder.Priority;
    }

    public class EmailMessageBuilder
    {
        internal string From { get; private set; }
        internal List<string> To { get; } = new();
        internal List<string> Cc { get; } = new();
        internal List<string> Bcc { get; } = new();
        internal string Subject { get; private set; }
        internal string Body { get; private set; }
        internal bool IsHtml { get; private set; }
        internal List<Attachment> Attachments { get; } = new();
        internal Priority Priority { get; private set; } = Priority.Normal;

        public EmailMessageBuilder SetFrom(string from)
        {
            From = from;
            return this;
        }

        public EmailMessageBuilder AddTo(string to)
        {
            To.Add(to);
            return this;
        }

        public EmailMessageBuilder AddCc(string cc)
        {
            Cc.Add(cc);
            return this;
        }

        public EmailMessageBuilder SetSubject(string subject)
        {
            Subject = subject;
            return this;
        }

        public EmailMessageBuilder SetBody(string body, bool isHtml = false)
        {
            Body = body;
            IsHtml = isHtml;
            return this;
        }

        public EmailMessageBuilder AddAttachment(Attachment attachment)
        {
            Attachments.Add(attachment);
            return this;
        }

        public EmailMessageBuilder SetPriority(Priority priority)
        {
            Priority = priority;
            return this;
        }

        public EmailMessage Build()
        {
            // Validation
            if (string.IsNullOrEmpty(From))
                throw new InvalidOperationException("From is required");
            if (!To.Any())
                throw new InvalidOperationException("At least one recipient required");

            return new EmailMessage(this);
        }
    }
}

// Usage - fluent and readable
var email = new EmailMessage.EmailMessageBuilder()
    .SetFrom("noreply@company.com")
    .AddTo("customer@email.com")
    .AddCc("manager@company.com")
    .SetSubject("Order Confirmation")
    .SetBody("<h1>Thank you for your order!</h1>", isHtml: true)
    .SetPriority(Priority.High)
    .AddAttachment(new Attachment("invoice.pdf", invoiceBytes))
    .Build();
```

**Decision Criteria:**

| Criteria | Use Factory | Use Builder |
|----------|-------------|-------------|
| Object variety | Different types | Same type, different configs |
| Parameters | Few, required | Many, optional |
| Construction | Single step | Multi-step |
| Return type | Interface/base class | Concrete class |
| Focus | WHAT to create | HOW to configure |

**Real-World .NET Examples:**

```csharp
// Factory in .NET - LoggerFactory
var loggerFactory = LoggerFactory.Create(builder =>
{
    builder.AddConsole();
    builder.AddDebug();
});
var logger = loggerFactory.CreateLogger<MyClass>();  // Factory creates logger

// Builder in .NET - HostBuilder
var host = Host.CreateDefaultBuilder(args)
    .ConfigureWebHostDefaults(webBuilder =>
    {
        webBuilder.UseStartup<Startup>();  // Builder pattern
    })
    .ConfigureServices(services =>
    {
        services.AddControllers();
    })
    .Build();  // Finally build the host

// Builder in .NET - HttpClient
var request = new HttpRequestMessage()
{
    Method = HttpMethod.Post,
    RequestUri = new Uri("https://api.example.com/orders"),
    Headers = { { "Authorization", "Bearer token" } },
    Content = new StringContent(json, Encoding.UTF8, "application/json")
};
```

### Communication Tactics

🎯 **Structure your answer**: Define each pattern, show examples, then give clear decision criteria with a comparison table.

💡 **Emphasize**: Factory = "what type to create", Builder = "how to configure". They solve different problems.

⚠️ **Avoid**: Don't say one is better than the other. They're complementary. You can even combine them (Factory returns a Builder).

---

## Question 3: Repository and Unit of Work Together

### The Question
> "Explain the Repository and Unit of Work patterns. How do they work together, and can you show me an implementation?"

### Key Points to Cover
- Purpose of each pattern
- How they complement each other
- Practical implementation
- When to use (and when not to)

### Detailed Answer

**Repository Pattern - Purpose:**

- Abstracts data access logic
- Provides collection-like interface for entities
- Enables testing with mock repositories
- Centralizes query logic

**Unit of Work Pattern - Purpose:**

- Tracks changes across multiple repositories
- Coordinates transaction commit
- Ensures atomicity (all or nothing)
- Prevents multiple SaveChanges calls

**Implementation:**

```csharp
// Entities
public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public List<OrderItem> Items { get; set; } = new();
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal Price { get; set; }
}

// Generic Repository Interface
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate);
    void Add(T entity);
    void Update(T entity);
    void Remove(T entity);
}

// Specific Repository Interface (for custom queries)
public interface IOrderRepository : IRepository<Order>
{
    Task<IEnumerable<Order>> GetByCustomerAsync(int customerId);
    Task<IEnumerable<Order>> GetPendingOrdersAsync();
}

// Unit of Work Interface
public interface IUnitOfWork : IDisposable
{
    IOrderRepository Orders { get; }
    IRepository<OrderItem> OrderItems { get; }
    IRepository<Product> Products { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}

// Generic Repository Implementation
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly AppDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(AppDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id)
    {
        return await _dbSet.FindAsync(id);
    }

    public async Task<IEnumerable<T>> GetAllAsync()
    {
        return await _dbSet.ToListAsync();
    }

    public async Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate)
    {
        return await _dbSet.Where(predicate).ToListAsync();
    }

    public void Add(T entity)
    {
        _dbSet.Add(entity);
    }

    public void Update(T entity)
    {
        _dbSet.Update(entity);
    }

    public void Remove(T entity)
    {
        _dbSet.Remove(entity);
    }
}

// Specific Repository Implementation
public class OrderRepository : Repository<Order>, IOrderRepository
{
    public OrderRepository(AppDbContext context) : base(context) { }

    public async Task<IEnumerable<Order>> GetByCustomerAsync(int customerId)
    {
        return await _dbSet
            .Include(o => o.Items)
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.Id)
            .ToListAsync();
    }

    public async Task<IEnumerable<Order>> GetPendingOrdersAsync()
    {
        return await _dbSet
            .Where(o => o.Status == OrderStatus.Pending)
            .ToListAsync();
    }
}

// Unit of Work Implementation
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    private IOrderRepository? _orders;
    private IRepository<OrderItem>? _orderItems;
    private IRepository<Product>? _products;

    public UnitOfWork(AppDbContext context)
    {
        _context = context;
    }

    public IOrderRepository Orders =>
        _orders ??= new OrderRepository(_context);

    public IRepository<OrderItem> OrderItems =>
        _orderItems ??= new Repository<OrderItem>(_context);

    public IRepository<Product> Products =>
        _products ??= new Repository<Product>(_context);

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        return await _context.SaveChangesAsync(cancellationToken);
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}
```

**Using Repository and Unit of Work Together:**

```csharp
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;

    public OrderService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // Create order
        var order = new Order
        {
            CustomerId = request.CustomerId,
            Status = OrderStatus.Pending
        };

        // Add items and calculate total
        foreach (var item in request.Items)
        {
            var product = await _unitOfWork.Products.GetByIdAsync(item.ProductId);
            if (product == null)
                throw new NotFoundException($"Product {item.ProductId} not found");

            order.Items.Add(new OrderItem
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity,
                Price = product.Price
            });
        }

        order.Total = order.Items.Sum(i => i.Price * i.Quantity);

        // Add to repository
        _unitOfWork.Orders.Add(order);

        // Save all changes in one transaction
        await _unitOfWork.SaveChangesAsync();

        return order;
    }

    public async Task ProcessOrderAsync(int orderId)
    {
        var order = await _unitOfWork.Orders.GetByIdAsync(orderId);
        if (order == null)
            throw new NotFoundException($"Order {orderId} not found");

        // Update order status
        order.Status = OrderStatus.Processing;
        _unitOfWork.Orders.Update(order);

        // Update inventory (different repository, same transaction)
        foreach (var item in order.Items)
        {
            var product = await _unitOfWork.Products.GetByIdAsync(item.ProductId);
            product.StockQuantity -= item.Quantity;
            _unitOfWork.Products.Update(product);
        }

        // Single SaveChanges - atomic operation
        await _unitOfWork.SaveChangesAsync();
    }
}
```

**DI Registration:**

```csharp
// In Program.cs or Startup.cs
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

services.AddScoped<IUnitOfWork, UnitOfWork>();
services.AddScoped<IOrderRepository, OrderRepository>();
```

### Communication Tactics

🎯 **Structure your answer**: Explain each pattern's purpose, show how they complement each other, then demonstrate with code.

💡 **Emphasize**: Unit of Work ensures atomicity across multiple repository operations. One SaveChanges commits all changes or rolls back all.

⚠️ **Avoid**: Don't overcomplicate for simple apps. For simple CRUD, EF Core's DbContext already implements these patterns internally.

---

## Question 4: Strategy Pattern and Open/Closed Principle

### The Question
> "How does Strategy pattern support the Open/Closed Principle? Show me an example where adding new behavior doesn't require modifying existing code."

### Key Points to Cover
- Strategy pattern definition
- How it enables OCP
- Practical example
- Benefits for maintenance

### Detailed Answer

**Strategy Pattern Overview:**

Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. This allows the algorithm to vary independently from clients that use it.

**How It Supports OCP:**

```csharp
// ❌ WITHOUT Strategy - Violates OCP
public class ShippingCalculator
{
    public decimal Calculate(Order order, string shippingMethod)
    {
        switch (shippingMethod)
        {
            case "standard":
                return order.Weight * 5;
            case "express":
                return order.Weight * 10 + 20;
            case "overnight":
                return order.Weight * 15 + 50;
            // Must modify this class for every new shipping method!
            default:
                throw new ArgumentException("Unknown shipping method");
        }
    }
}

// ✅ WITH Strategy - Follows OCP
public interface IShippingStrategy
{
    string Name { get; }
    decimal Calculate(Order order);
}

public class StandardShipping : IShippingStrategy
{
    public string Name => "standard";

    public decimal Calculate(Order order)
    {
        return order.Weight * 5;
    }
}

public class ExpressShipping : IShippingStrategy
{
    public string Name => "express";

    public decimal Calculate(Order order)
    {
        return order.Weight * 10 + 20;
    }
}

public class OvernightShipping : IShippingStrategy
{
    public string Name => "overnight";

    public decimal Calculate(Order order)
    {
        return order.Weight * 15 + 50;
    }
}

// NEW shipping method - just add new class, no modification!
public class DroneShipping : IShippingStrategy
{
    public string Name => "drone";

    public decimal Calculate(Order order)
    {
        if (order.Weight > 5)
            throw new InvalidOperationException("Drone can't carry more than 5kg");

        return order.Weight * 8 + 15;
    }
}
```

**Using the Strategies:**

```csharp
public class ShippingService
{
    private readonly IEnumerable<IShippingStrategy> _strategies;

    // All strategies injected via DI
    public ShippingService(IEnumerable<IShippingStrategy> strategies)
    {
        _strategies = strategies;
    }

    public decimal CalculateShipping(Order order, string shippingMethod)
    {
        var strategy = _strategies.FirstOrDefault(s => s.Name == shippingMethod)
            ?? throw new ArgumentException($"Unknown shipping method: {shippingMethod}");

        return strategy.Calculate(order);
    }

    public IEnumerable<ShippingOption> GetAvailableOptions(Order order)
    {
        return _strategies.Select(s => new ShippingOption
        {
            Method = s.Name,
            Cost = s.Calculate(order)
        });
    }
}

// DI Registration
services.AddScoped<IShippingStrategy, StandardShipping>();
services.AddScoped<IShippingStrategy, ExpressShipping>();
services.AddScoped<IShippingStrategy, OvernightShipping>();
services.AddScoped<IShippingStrategy, DroneShipping>();
services.AddScoped<ShippingService>();
```

**Real-World Example - Payment Processing:**

```csharp
public interface IPaymentStrategy
{
    string Provider { get; }
    bool Supports(PaymentRequest request);
    Task<PaymentResult> ProcessAsync(PaymentRequest request);
}

public class CreditCardPayment : IPaymentStrategy
{
    public string Provider => "credit_card";

    public bool Supports(PaymentRequest request)
    {
        return !string.IsNullOrEmpty(request.CardNumber);
    }

    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        // Credit card processing
        return new PaymentResult { Success = true };
    }
}

public class PayPalPayment : IPaymentStrategy
{
    private readonly IPayPalClient _client;

    public PayPalPayment(IPayPalClient client) => _client = client;

    public string Provider => "paypal";

    public bool Supports(PaymentRequest request)
    {
        return !string.IsNullOrEmpty(request.PayPalEmail);
    }

    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        return await _client.ChargeAsync(request.PayPalEmail, request.Amount);
    }
}

// Context class
public class PaymentProcessor
{
    private readonly IEnumerable<IPaymentStrategy> _strategies;

    public PaymentProcessor(IEnumerable<IPaymentStrategy> strategies)
    {
        _strategies = strategies;
    }

    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        var strategy = _strategies.FirstOrDefault(s => s.Supports(request))
            ?? throw new InvalidOperationException("No payment method available");

        return await strategy.ProcessAsync(request);
    }
}
```

**Benefits:**

| Benefit | Description |
|---------|-------------|
| Open/Closed | Add new strategies without modifying existing code |
| Single Responsibility | Each strategy handles one algorithm |
| Testability | Test each strategy in isolation |
| Runtime flexibility | Swap strategies at runtime |
| Clean code | No switch statements or if-else chains |

### Communication Tactics

🎯 **Structure your answer**: Show the OCP violation first (switch statement), then refactor to Strategy pattern, demonstrate adding new behavior.

💡 **Emphasize**: Adding DroneShipping required ZERO changes to existing code. That's OCP in action.

⚠️ **Avoid**: Don't use Strategy for everything. Simple cases with 2-3 options that rarely change don't need this complexity.

---

## Question 5: Notification System Design

### The Question
> "Design a notification system that can send notifications via email, SMS, and push notifications. Which patterns would you use?"

### Key Points to Cover
- Multiple patterns working together
- Observer or Strategy pattern for channels
- Factory for creating notifiers
- Extensibility for new channels

### Detailed Answer

**Approach: Combining Strategy + Observer + Factory:**

```csharp
// Domain Events (Observer pattern)
public interface INotificationEvent
{
    string UserId { get; }
    string Title { get; }
    string Message { get; }
}

public record OrderPlacedEvent(string UserId, string OrderId, decimal Total) : INotificationEvent
{
    public string Title => "Order Confirmed";
    public string Message => $"Your order #{OrderId} for {Total:C} has been placed.";
}

public record PaymentFailedEvent(string UserId, string OrderId, string Reason) : INotificationEvent
{
    public string Title => "Payment Failed";
    public string Message => $"Payment for order #{OrderId} failed: {Reason}";
}

// Strategy pattern - Notification channels
public interface INotificationChannel
{
    string ChannelType { get; }
    Task SendAsync(string userId, string title, string message);
}

public class EmailNotificationChannel : INotificationChannel
{
    private readonly IEmailService _emailService;
    private readonly IUserRepository _userRepository;

    public EmailNotificationChannel(IEmailService emailService, IUserRepository userRepository)
    {
        _emailService = emailService;
        _userRepository = userRepository;
    }

    public string ChannelType => "email";

    public async Task SendAsync(string userId, string title, string message)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        if (user?.Email == null) return;

        await _emailService.SendAsync(new EmailMessage
        {
            To = user.Email,
            Subject = title,
            Body = message
        });
    }
}

public class SmsNotificationChannel : INotificationChannel
{
    private readonly ISmsGateway _smsGateway;
    private readonly IUserRepository _userRepository;

    public SmsNotificationChannel(ISmsGateway smsGateway, IUserRepository userRepository)
    {
        _smsGateway = smsGateway;
        _userRepository = userRepository;
    }

    public string ChannelType => "sms";

    public async Task SendAsync(string userId, string title, string message)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        if (user?.PhoneNumber == null) return;

        // SMS is shorter, combine title and message
        await _smsGateway.SendAsync(user.PhoneNumber, $"{title}: {message}");
    }
}

public class PushNotificationChannel : INotificationChannel
{
    private readonly IPushService _pushService;

    public PushNotificationChannel(IPushService pushService)
    {
        _pushService = pushService;
    }

    public string ChannelType => "push";

    public async Task SendAsync(string userId, string title, string message)
    {
        await _pushService.SendAsync(new PushMessage
        {
            UserId = userId,
            Title = title,
            Body = message,
            Data = new { type = "notification" }
        });
    }
}

// User preferences
public class NotificationPreferences
{
    public string UserId { get; set; }
    public bool EmailEnabled { get; set; } = true;
    public bool SmsEnabled { get; set; } = false;
    public bool PushEnabled { get; set; } = true;

    public IEnumerable<string> GetEnabledChannels()
    {
        if (EmailEnabled) yield return "email";
        if (SmsEnabled) yield return "sms";
        if (PushEnabled) yield return "push";
    }
}

// Notification Service - Orchestrator
public class NotificationService
{
    private readonly IEnumerable<INotificationChannel> _channels;
    private readonly INotificationPreferencesRepository _preferencesRepo;
    private readonly ILogger<NotificationService> _logger;

    public NotificationService(
        IEnumerable<INotificationChannel> channels,
        INotificationPreferencesRepository preferencesRepo,
        ILogger<NotificationService> logger)
    {
        _channels = channels;
        _preferencesRepo = preferencesRepo;
        _logger = logger;
    }

    public async Task NotifyAsync(INotificationEvent notification)
    {
        var preferences = await _preferencesRepo.GetByUserIdAsync(notification.UserId)
            ?? new NotificationPreferences { UserId = notification.UserId };

        var enabledChannels = preferences.GetEnabledChannels();

        var tasks = _channels
            .Where(c => enabledChannels.Contains(c.ChannelType))
            .Select(async channel =>
            {
                try
                {
                    await channel.SendAsync(
                        notification.UserId,
                        notification.Title,
                        notification.Message);

                    _logger.LogInformation(
                        "Notification sent via {Channel} to {UserId}",
                        channel.ChannelType,
                        notification.UserId);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex,
                        "Failed to send notification via {Channel} to {UserId}",
                        channel.ChannelType,
                        notification.UserId);
                }
            });

        await Task.WhenAll(tasks);
    }
}

// DI Registration
services.AddScoped<INotificationChannel, EmailNotificationChannel>();
services.AddScoped<INotificationChannel, SmsNotificationChannel>();
services.AddScoped<INotificationChannel, PushNotificationChannel>();
services.AddScoped<NotificationService>();

// Usage
public class OrderService
{
    private readonly NotificationService _notifications;

    public async Task PlaceOrderAsync(Order order)
    {
        // ... place order logic ...

        // Send notification
        await _notifications.NotifyAsync(new OrderPlacedEvent(
            order.CustomerId,
            order.Id.ToString(),
            order.Total));
    }
}
```

**Adding New Channel (OCP in action):**

```csharp
// Add Slack notifications - no changes to existing code!
public class SlackNotificationChannel : INotificationChannel
{
    private readonly ISlackClient _slackClient;
    private readonly IUserRepository _userRepository;

    public SlackNotificationChannel(ISlackClient slackClient, IUserRepository userRepository)
    {
        _slackClient = slackClient;
        _userRepository = userRepository;
    }

    public string ChannelType => "slack";

    public async Task SendAsync(string userId, string title, string message)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        if (user?.SlackUserId == null) return;

        await _slackClient.SendDirectMessageAsync(user.SlackUserId, $"*{title}*\n{message}");
    }
}

// Just register it!
services.AddScoped<INotificationChannel, SlackNotificationChannel>();
```

**Patterns Used:**

| Pattern | Purpose |
|---------|---------|
| Strategy | Each channel is a strategy for sending notifications |
| Observer | Events trigger notifications to multiple channels |
| Factory (via DI) | DI container creates and injects channel implementations |
| Template Method | NotificationService defines the workflow, channels implement details |

### Communication Tactics

🎯 **Structure your answer**: Start with the problem, identify patterns, show implementation, demonstrate extensibility.

💡 **Emphasize**: Adding Slack required ONE new class and ONE DI registration. The core NotificationService never changed.

⚠️ **Avoid**: Don't over-engineer. For a simple app with only email, this is overkill. Show you understand trade-offs.

---

## Quick Review - Key Takeaways

| Question | Key Point |
|----------|-----------|
| Thread-Safe Singleton | Use Lazy<T> in modern .NET |
| Factory vs Builder | Factory = what type, Builder = how to configure |
| Repository + UoW | Repository abstracts data, UoW coordinates transactions |
| Strategy + OCP | New behavior = new class, no modifications |
| Notification System | Combine Strategy + Observer + DI for extensibility |
