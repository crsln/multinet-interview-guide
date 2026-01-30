# Design Patterns

> ⚠️ **HIGH PRIORITY**: Comprehensive patterns with C# examples

---

## Overview

Design patterns are reusable solutions to common software design problems. They provide a shared vocabulary for developers and proven approaches to recurring challenges.

---

# Creational Patterns

Patterns that deal with object creation mechanisms.

---

## Singleton

### Purpose
Ensure a class has only one instance and provide a global point of access to it.

### When to Use
- Configuration managers
- Logging services
- Connection pools
- Caching services

### C# Implementation

```csharp
// ❌ NAIVE (not thread-safe)
public class NaiveSingleton
{
    private static NaiveSingleton _instance;

    private NaiveSingleton() { }

    public static NaiveSingleton Instance
    {
        get
        {
            if (_instance == null)
                _instance = new NaiveSingleton(); // Race condition!
            return _instance;
        }
    }
}

// ✅ THREAD-SAFE with lock
public class ThreadSafeSingleton
{
    private static ThreadSafeSingleton _instance;
    private static readonly object _lock = new object();

    private ThreadSafeSingleton() { }

    public static ThreadSafeSingleton Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                        _instance = new ThreadSafeSingleton();
                }
            }
            return _instance;
        }
    }
}

// ✅ BEST: Lazy<T> (recommended in modern C#)
public class LazySingleton
{
    private static readonly Lazy<LazySingleton> _instance =
        new Lazy<LazySingleton>(() => new LazySingleton());

    private LazySingleton() { }

    public static LazySingleton Instance => _instance.Value;

    public void DoSomething() { /* ... */ }
}

// ✅ SIMPLEST: Static constructor (thread-safe by CLR)
public class StaticSingleton
{
    private static readonly StaticSingleton _instance = new StaticSingleton();

    // Static constructor - CLR guarantees thread-safe initialization
    static StaticSingleton() { }

    private StaticSingleton() { }

    public static StaticSingleton Instance => _instance;
}
```

### Real-World Example: Configuration Manager

```csharp
public sealed class AppConfiguration
{
    private static readonly Lazy<AppConfiguration> _instance =
        new Lazy<AppConfiguration>(() => new AppConfiguration());

    private readonly IConfiguration _configuration;

    private AppConfiguration()
    {
        _configuration = new ConfigurationBuilder()
            .AddJsonFile("appsettings.json")
            .AddEnvironmentVariables()
            .Build();
    }

    public static AppConfiguration Instance => _instance.Value;

    public string GetConnectionString(string name)
        => _configuration.GetConnectionString(name);

    public T GetSection<T>(string key) where T : new()
        => _configuration.GetSection(key).Get<T>() ?? new T();
}
```

### With Dependency Injection (Preferred in ASP.NET Core)

```csharp
// Modern approach - let DI container manage singleton
services.AddSingleton<IConfigurationService, ConfigurationService>();

// The class itself doesn't need singleton pattern
public class ConfigurationService : IConfigurationService
{
    private readonly IConfiguration _configuration;

    public ConfigurationService(IConfiguration configuration)
    {
        _configuration = configuration;
    }
}
```

---

## Factory Method

### Purpose
Define an interface for creating an object, but let subclasses decide which class to instantiate.

### When to Use
- When you don't know ahead of time which concrete class you need
- When creation logic is complex
- When you want to delegate instantiation to subclasses

### C# Implementation

```csharp
// Product interface
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(decimal amount);
}

// Concrete products
public class CreditCardProcessor : IPaymentProcessor
{
    public async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        await Task.Delay(100);
        return new PaymentResult { Success = true, Method = "CreditCard" };
    }
}

public class PayPalProcessor : IPaymentProcessor
{
    public async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        await Task.Delay(100);
        return new PaymentResult { Success = true, Method = "PayPal" };
    }
}

public class CryptoProcessor : IPaymentProcessor
{
    public async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        await Task.Delay(100);
        return new PaymentResult { Success = true, Method = "Crypto" };
    }
}

// Factory
public interface IPaymentProcessorFactory
{
    IPaymentProcessor Create(string paymentMethod);
}

public class PaymentProcessorFactory : IPaymentProcessorFactory
{
    private readonly IServiceProvider _serviceProvider;

    public PaymentProcessorFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IPaymentProcessor Create(string paymentMethod)
    {
        return paymentMethod.ToLower() switch
        {
            "creditcard" => _serviceProvider.GetRequiredService<CreditCardProcessor>(),
            "paypal" => _serviceProvider.GetRequiredService<PayPalProcessor>(),
            "crypto" => _serviceProvider.GetRequiredService<CryptoProcessor>(),
            _ => throw new NotSupportedException($"Payment method {paymentMethod} not supported")
        };
    }
}

// Usage
public class CheckoutService
{
    private readonly IPaymentProcessorFactory _factory;

    public CheckoutService(IPaymentProcessorFactory factory)
    {
        _factory = factory;
    }

    public async Task<PaymentResult> ProcessPaymentAsync(Order order)
    {
        var processor = _factory.Create(order.PaymentMethod);
        return await processor.ProcessAsync(order.Total);
    }
}
```

---

## Abstract Factory

### Purpose
Provide an interface for creating families of related objects without specifying their concrete classes.

### When to Use
- When you need to create families of related objects
- When you want to enforce that objects from one family work together

### C# Implementation

```csharp
// Abstract products
public interface IButton { void Render(); }
public interface ITextBox { void Render(); }
public interface ICheckbox { void Render(); }

// Abstract factory
public interface IUIFactory
{
    IButton CreateButton();
    ITextBox CreateTextBox();
    ICheckbox CreateCheckbox();
}

// Windows family
public class WindowsButton : IButton
{
    public void Render() => Console.WriteLine("Windows Button");
}

public class WindowsTextBox : ITextBox
{
    public void Render() => Console.WriteLine("Windows TextBox");
}

public class WindowsCheckbox : ICheckbox
{
    public void Render() => Console.WriteLine("Windows Checkbox");
}

public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton() => new WindowsButton();
    public ITextBox CreateTextBox() => new WindowsTextBox();
    public ICheckbox CreateCheckbox() => new WindowsCheckbox();
}

// Mac family
public class MacButton : IButton
{
    public void Render() => Console.WriteLine("Mac Button");
}

public class MacTextBox : ITextBox
{
    public void Render() => Console.WriteLine("Mac TextBox");
}

public class MacCheckbox : ICheckbox
{
    public void Render() => Console.WriteLine("Mac Checkbox");
}

public class MacUIFactory : IUIFactory
{
    public IButton CreateButton() => new MacButton();
    public ITextBox CreateTextBox() => new MacTextBox();
    public ICheckbox CreateCheckbox() => new MacCheckbox();
}

// Usage - works with any UI family
public class Application
{
    private readonly IButton _button;
    private readonly ITextBox _textBox;
    private readonly ICheckbox _checkbox;

    public Application(IUIFactory factory)
    {
        // All components from the same family
        _button = factory.CreateButton();
        _textBox = factory.CreateTextBox();
        _checkbox = factory.CreateCheckbox();
    }

    public void RenderUI()
    {
        _button.Render();
        _textBox.Render();
        _checkbox.Render();
    }
}
```

---

## Builder

### Purpose
Separate the construction of a complex object from its representation, allowing the same construction process to create different representations.

### When to Use
- Complex object construction with many parameters
- When you want fluent, readable object creation
- When construction requires multiple steps

### C# Implementation

```csharp
public class PaymentRequest
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }
    public string CustomerId { get; set; }
    public string Description { get; set; }
    public Dictionary<string, string> Metadata { get; set; }
    public bool CaptureImmediately { get; set; }
    public string IdempotencyKey { get; set; }
    public Address BillingAddress { get; set; }
}

// Fluent Builder
public class PaymentRequestBuilder
{
    private readonly PaymentRequest _request = new PaymentRequest
    {
        Currency = "TRY",
        CaptureImmediately = true,
        Metadata = new Dictionary<string, string>()
    };

    public PaymentRequestBuilder WithAmount(decimal amount)
    {
        _request.Amount = amount;
        return this;
    }

    public PaymentRequestBuilder WithCurrency(string currency)
    {
        _request.Currency = currency;
        return this;
    }

    public PaymentRequestBuilder ForCustomer(string customerId)
    {
        _request.CustomerId = customerId;
        return this;
    }

    public PaymentRequestBuilder WithDescription(string description)
    {
        _request.Description = description;
        return this;
    }

    public PaymentRequestBuilder AddMetadata(string key, string value)
    {
        _request.Metadata[key] = value;
        return this;
    }

    public PaymentRequestBuilder WithIdempotencyKey(string key)
    {
        _request.IdempotencyKey = key;
        return this;
    }

    public PaymentRequestBuilder DeferCapture()
    {
        _request.CaptureImmediately = false;
        return this;
    }

    public PaymentRequestBuilder WithBillingAddress(Action<AddressBuilder> configure)
    {
        var addressBuilder = new AddressBuilder();
        configure(addressBuilder);
        _request.BillingAddress = addressBuilder.Build();
        return this;
    }

    public PaymentRequest Build()
    {
        // Validation
        if (_request.Amount <= 0)
            throw new InvalidOperationException("Amount must be positive");
        if (string.IsNullOrEmpty(_request.CustomerId))
            throw new InvalidOperationException("Customer ID is required");

        return _request;
    }
}

// Usage - fluent and readable
var request = new PaymentRequestBuilder()
    .WithAmount(1500.00m)
    .WithCurrency("TRY")
    .ForCustomer("cust_123")
    .WithDescription("Order #12345")
    .AddMetadata("orderId", "12345")
    .AddMetadata("source", "web")
    .WithIdempotencyKey(Guid.NewGuid().ToString())
    .WithBillingAddress(addr => addr
        .WithStreet("123 Main St")
        .WithCity("Istanbul")
        .WithCountry("Turkey"))
    .Build();
```

---

## Prototype

### Purpose
Create new objects by cloning existing ones rather than constructing from scratch.

### When to Use
- When object creation is expensive
- When you need copies of complex objects
- When you want to avoid subclass explosion for object configuration

### C# Implementation

```csharp
public interface IPrototype<T>
{
    T Clone();
}

public class ReportTemplate : IPrototype<ReportTemplate>
{
    public string Title { get; set; }
    public string Header { get; set; }
    public string Footer { get; set; }
    public List<string> Columns { get; set; }
    public ReportStyle Style { get; set; }

    public ReportTemplate Clone()
    {
        return new ReportTemplate
        {
            Title = this.Title,
            Header = this.Header,
            Footer = this.Footer,
            Columns = new List<string>(this.Columns), // Deep copy
            Style = this.Style.Clone() // Deep copy
        };
    }
}

// Usage
var standardReport = new ReportTemplate
{
    Title = "Monthly Report",
    Header = "Company Name",
    Footer = "Confidential",
    Columns = new List<string> { "Date", "Amount", "Description" },
    Style = new ReportStyle { FontSize = 12, Color = "Black" }
};

// Clone and customize
var salesReport = standardReport.Clone();
salesReport.Title = "Sales Report";
salesReport.Columns.Add("Salesperson");

var financeReport = standardReport.Clone();
financeReport.Title = "Finance Report";
financeReport.Columns.Add("Account");
```

---

# Structural Patterns

Patterns that deal with object composition and relationships.

---

## Adapter

### Purpose
Convert the interface of a class into another interface that clients expect. Allows incompatible interfaces to work together.

### When to Use
- Integrating with third-party libraries
- Working with legacy code
- When you can't modify the source class

### C# Implementation

```csharp
// External payment gateway with incompatible interface
public class LegacyPaymentGateway
{
    public int MakePayment(string cardNumber, int amountInKurus, string currencyCode)
    {
        // Returns: 0 = success, 1 = declined, 2 = error
        return 0;
    }
}

// Our expected interface
public interface IPaymentGateway
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request);
}

// Adapter - bridges the gap
public class LegacyPaymentGatewayAdapter : IPaymentGateway
{
    private readonly LegacyPaymentGateway _legacyGateway;

    public LegacyPaymentGatewayAdapter(LegacyPaymentGateway legacyGateway)
    {
        _legacyGateway = legacyGateway;
    }

    public Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Convert from our model to legacy format
        var amountInKurus = (int)(request.Amount * 100);
        var currencyCode = request.Currency ?? "TRY";

        // Call legacy system
        var legacyResult = _legacyGateway.MakePayment(
            request.CardNumber,
            amountInKurus,
            currencyCode);

        // Convert result back to our format
        var result = legacyResult switch
        {
            0 => new PaymentResult { Success = true },
            1 => new PaymentResult { Success = false, Error = "Card declined" },
            _ => new PaymentResult { Success = false, Error = "Processing error" }
        };

        return Task.FromResult(result);
    }
}

// Usage - client code uses our interface
public class PaymentService
{
    private readonly IPaymentGateway _gateway;

    public PaymentService(IPaymentGateway gateway)
    {
        _gateway = gateway; // Could be adapter or native implementation
    }

    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        return await _gateway.ProcessPaymentAsync(request);
    }
}
```

---

## Decorator

### Purpose
Attach additional responsibilities to an object dynamically. Provides a flexible alternative to subclassing for extending functionality.

### When to Use
- Adding behavior without modifying existing code
- When extension by subclassing is impractical
- When you need to add/remove behaviors at runtime

### C# Implementation

```csharp
public interface INotificationService
{
    Task SendAsync(string recipient, string message);
}

// Base implementation
public class EmailNotificationService : INotificationService
{
    public async Task SendAsync(string recipient, string message)
    {
        Console.WriteLine($"Sending email to {recipient}: {message}");
        await Task.Delay(100);
    }
}

// Decorator base
public abstract class NotificationDecorator : INotificationService
{
    protected readonly INotificationService _inner;

    protected NotificationDecorator(INotificationService inner)
    {
        _inner = inner;
    }

    public virtual Task SendAsync(string recipient, string message)
    {
        return _inner.SendAsync(recipient, message);
    }
}

// Logging decorator
public class LoggingNotificationDecorator : NotificationDecorator
{
    private readonly ILogger _logger;

    public LoggingNotificationDecorator(INotificationService inner, ILogger logger)
        : base(inner)
    {
        _logger = logger;
    }

    public override async Task SendAsync(string recipient, string message)
    {
        _logger.LogInformation("Sending notification to {Recipient}", recipient);
        var sw = Stopwatch.StartNew();

        await base.SendAsync(recipient, message);

        _logger.LogInformation("Notification sent in {Elapsed}ms", sw.ElapsedMilliseconds);
    }
}

// Retry decorator
public class RetryNotificationDecorator : NotificationDecorator
{
    private readonly int _maxRetries;

    public RetryNotificationDecorator(INotificationService inner, int maxRetries = 3)
        : base(inner)
    {
        _maxRetries = maxRetries;
    }

    public override async Task SendAsync(string recipient, string message)
    {
        for (int attempt = 1; attempt <= _maxRetries; attempt++)
        {
            try
            {
                await base.SendAsync(recipient, message);
                return;
            }
            catch when (attempt < _maxRetries)
            {
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)));
            }
        }
    }
}

// Encryption decorator
public class EncryptedNotificationDecorator : NotificationDecorator
{
    private readonly IEncryptionService _encryption;

    public EncryptedNotificationDecorator(INotificationService inner, IEncryptionService encryption)
        : base(inner)
    {
        _encryption = encryption;
    }

    public override async Task SendAsync(string recipient, string message)
    {
        var encryptedMessage = _encryption.Encrypt(message);
        await base.SendAsync(recipient, encryptedMessage);
    }
}

// Usage - stack decorators as needed
INotificationService service = new EmailNotificationService();
service = new LoggingNotificationDecorator(service, logger);
service = new RetryNotificationDecorator(service, maxRetries: 3);
service = new EncryptedNotificationDecorator(service, encryption);

await service.SendAsync("user@email.com", "Your payment was processed");
// Now sends encrypted message, with retry logic, with logging
```

---

## Facade

### Purpose
Provide a unified interface to a set of interfaces in a subsystem. Defines a higher-level interface that makes the subsystem easier to use.

### When to Use
- Simplifying complex subsystem interactions
- Providing a simple API for library/framework
- Decoupling clients from subsystem details

### C# Implementation

```csharp
// Complex subsystem classes
public class InventoryService
{
    public bool CheckStock(string productId, int quantity)
        => true; // Simplified

    public void ReserveStock(string productId, int quantity) { }
    public void ReleaseStock(string productId, int quantity) { }
}

public class PaymentService
{
    public PaymentResult ProcessPayment(string customerId, decimal amount)
        => new PaymentResult { Success = true };

    public void RefundPayment(string transactionId) { }
}

public class ShippingService
{
    public string CreateShipment(string orderId, Address address)
        => "SHIP123";

    public void CancelShipment(string shipmentId) { }
}

public class NotificationService
{
    public void SendOrderConfirmation(string email, string orderId) { }
    public void SendShippingNotification(string email, string trackingNumber) { }
}

// Facade - simple interface to complex subsystem
public class OrderFacade
{
    private readonly InventoryService _inventory;
    private readonly PaymentService _payment;
    private readonly ShippingService _shipping;
    private readonly NotificationService _notifications;

    public OrderFacade(
        InventoryService inventory,
        PaymentService payment,
        ShippingService shipping,
        NotificationService notifications)
    {
        _inventory = inventory;
        _payment = payment;
        _shipping = shipping;
        _notifications = notifications;
    }

    // Simple method hides complex workflow
    public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
    {
        // Step 1: Check and reserve inventory
        foreach (var item in request.Items)
        {
            if (!_inventory.CheckStock(item.ProductId, item.Quantity))
                return OrderResult.Failed("Insufficient stock");

            _inventory.ReserveStock(item.ProductId, item.Quantity);
        }

        // Step 2: Process payment
        var paymentResult = _payment.ProcessPayment(request.CustomerId, request.Total);
        if (!paymentResult.Success)
        {
            // Rollback inventory
            foreach (var item in request.Items)
                _inventory.ReleaseStock(item.ProductId, item.Quantity);

            return OrderResult.Failed("Payment failed");
        }

        // Step 3: Create shipment
        var trackingNumber = _shipping.CreateShipment(request.OrderId, request.ShippingAddress);

        // Step 4: Send notifications
        _notifications.SendOrderConfirmation(request.CustomerEmail, request.OrderId);
        _notifications.SendShippingNotification(request.CustomerEmail, trackingNumber);

        return OrderResult.Success(request.OrderId, trackingNumber);
    }
}

// Client code - simple to use
var result = await _orderFacade.PlaceOrderAsync(orderRequest);
```

---

## Proxy

### Purpose
Provide a surrogate or placeholder for another object to control access to it.

### Types
- **Virtual Proxy**: Lazy loading of expensive objects
- **Protection Proxy**: Access control
- **Remote Proxy**: Represent object in different address space
- **Caching Proxy**: Cache results

### C# Implementation

```csharp
public interface IDocumentService
{
    Task<Document> GetDocumentAsync(string id);
    Task SaveDocumentAsync(Document document);
}

// Real implementation
public class DocumentService : IDocumentService
{
    public async Task<Document> GetDocumentAsync(string id)
    {
        // Expensive database call
        await Task.Delay(500);
        return new Document { Id = id, Content = "..." };
    }

    public async Task SaveDocumentAsync(Document document)
    {
        await Task.Delay(200);
    }
}

// Caching Proxy
public class CachingDocumentProxy : IDocumentService
{
    private readonly IDocumentService _realService;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _cacheDuration = TimeSpan.FromMinutes(5);

    public CachingDocumentProxy(IDocumentService realService, IMemoryCache cache)
    {
        _realService = realService;
        _cache = cache;
    }

    public async Task<Document> GetDocumentAsync(string id)
    {
        var cacheKey = $"doc_{id}";

        if (_cache.TryGetValue(cacheKey, out Document cached))
            return cached;

        var document = await _realService.GetDocumentAsync(id);

        _cache.Set(cacheKey, document, _cacheDuration);
        return document;
    }

    public async Task SaveDocumentAsync(Document document)
    {
        await _realService.SaveDocumentAsync(document);

        // Invalidate cache
        _cache.Remove($"doc_{document.Id}");
    }
}

// Access Control Proxy
public class SecureDocumentProxy : IDocumentService
{
    private readonly IDocumentService _realService;
    private readonly IUserContext _userContext;

    public SecureDocumentProxy(IDocumentService realService, IUserContext userContext)
    {
        _realService = realService;
        _userContext = userContext;
    }

    public async Task<Document> GetDocumentAsync(string id)
    {
        if (!await _userContext.HasPermissionAsync("documents:read"))
            throw new UnauthorizedAccessException("No read permission");

        return await _realService.GetDocumentAsync(id);
    }

    public async Task SaveDocumentAsync(Document document)
    {
        if (!await _userContext.HasPermissionAsync("documents:write"))
            throw new UnauthorizedAccessException("No write permission");

        await _realService.SaveDocumentAsync(document);
    }
}

// Lazy Loading Proxy
public class LazyDocumentProxy : IDocumentService
{
    private readonly Lazy<IDocumentService> _realService;

    public LazyDocumentProxy(Func<IDocumentService> factory)
    {
        _realService = new Lazy<IDocumentService>(factory);
    }

    public Task<Document> GetDocumentAsync(string id)
        => _realService.Value.GetDocumentAsync(id);

    public Task SaveDocumentAsync(Document document)
        => _realService.Value.SaveDocumentAsync(document);
}
```

---

## Composite

### Purpose
Compose objects into tree structures to represent part-whole hierarchies. Lets clients treat individual objects and compositions uniformly.

### When to Use
- Tree structures (file systems, UI hierarchies, org charts)
- When you want to treat groups and individuals the same way

### C# Implementation

```csharp
// Component interface
public interface IPriceable
{
    string Name { get; }
    decimal CalculatePrice();
    void Display(int indent = 0);
}

// Leaf - individual item
public class Product : IPriceable
{
    public string Name { get; }
    public decimal Price { get; }

    public Product(string name, decimal price)
    {
        Name = name;
        Price = price;
    }

    public decimal CalculatePrice() => Price;

    public void Display(int indent = 0)
    {
        Console.WriteLine($"{new string(' ', indent)}- {Name}: {Price:C}");
    }
}

// Composite - contains other components
public class ProductBundle : IPriceable
{
    public string Name { get; }
    private readonly List<IPriceable> _items = new();
    private readonly decimal _discountPercent;

    public ProductBundle(string name, decimal discountPercent = 0)
    {
        Name = name;
        _discountPercent = discountPercent;
    }

    public void Add(IPriceable item) => _items.Add(item);
    public void Remove(IPriceable item) => _items.Remove(item);

    public decimal CalculatePrice()
    {
        var total = _items.Sum(i => i.CalculatePrice());
        return total * (1 - _discountPercent / 100);
    }

    public void Display(int indent = 0)
    {
        Console.WriteLine($"{new string(' ', indent)}+ {Name} ({_discountPercent}% off): {CalculatePrice():C}");
        foreach (var item in _items)
        {
            item.Display(indent + 2);
        }
    }
}

// Usage
var laptop = new Product("Laptop", 15000m);
var mouse = new Product("Mouse", 500m);
var keyboard = new Product("Keyboard", 800m);

var peripheralsBundle = new ProductBundle("Peripherals Bundle", 10);
peripheralsBundle.Add(mouse);
peripheralsBundle.Add(keyboard);

var workstationBundle = new ProductBundle("Workstation Bundle", 5);
workstationBundle.Add(laptop);
workstationBundle.Add(peripheralsBundle);

// Treats both individual products and bundles the same
workstationBundle.Display();
Console.WriteLine($"Total: {workstationBundle.CalculatePrice():C}");
```

---

# Behavioral Patterns

Patterns that deal with object interaction and responsibility distribution.

---

## Strategy

### Purpose
Define a family of algorithms, encapsulate each one, and make them interchangeable. Lets the algorithm vary independently from clients that use it.

### When to Use
- Multiple algorithms for the same task
- When you need to switch algorithms at runtime
- To eliminate conditional statements

### C# Implementation

```csharp
// Strategy interface
public interface IDiscountStrategy
{
    decimal CalculateDiscount(Order order);
}

// Concrete strategies
public class NoDiscount : IDiscountStrategy
{
    public decimal CalculateDiscount(Order order) => 0;
}

public class PercentageDiscount : IDiscountStrategy
{
    private readonly decimal _percentage;

    public PercentageDiscount(decimal percentage)
    {
        _percentage = percentage;
    }

    public decimal CalculateDiscount(Order order)
        => order.Subtotal * (_percentage / 100);
}

public class FixedAmountDiscount : IDiscountStrategy
{
    private readonly decimal _amount;

    public FixedAmountDiscount(decimal amount)
    {
        _amount = amount;
    }

    public decimal CalculateDiscount(Order order)
        => Math.Min(_amount, order.Subtotal);
}

public class LoyaltyDiscount : IDiscountStrategy
{
    public decimal CalculateDiscount(Order order)
    {
        return order.Customer.LoyaltyTier switch
        {
            "Gold" => order.Subtotal * 0.15m,
            "Silver" => order.Subtotal * 0.10m,
            "Bronze" => order.Subtotal * 0.05m,
            _ => 0
        };
    }
}

// Context
public class OrderProcessor
{
    private IDiscountStrategy _discountStrategy;

    public OrderProcessor(IDiscountStrategy discountStrategy)
    {
        _discountStrategy = discountStrategy;
    }

    public void SetDiscountStrategy(IDiscountStrategy strategy)
    {
        _discountStrategy = strategy;
    }

    public decimal CalculateTotal(Order order)
    {
        var discount = _discountStrategy.CalculateDiscount(order);
        return order.Subtotal - discount;
    }
}

// Usage
var processor = new OrderProcessor(new NoDiscount());

// Apply promotional discount
processor.SetDiscountStrategy(new PercentageDiscount(20));
var promoTotal = processor.CalculateTotal(order);

// Switch to loyalty discount
processor.SetDiscountStrategy(new LoyaltyDiscount());
var loyaltyTotal = processor.CalculateTotal(order);
```

---

## Observer

### Purpose
Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

### C# Implementation (Using Events)

```csharp
// Using C# events (idiomatic approach)
public class Order
{
    public string OrderId { get; set; }
    public OrderStatus Status { get; private set; }

    // Event declaration
    public event EventHandler<OrderStatusChangedEventArgs> StatusChanged;

    public void UpdateStatus(OrderStatus newStatus)
    {
        var oldStatus = Status;
        Status = newStatus;

        // Notify observers
        StatusChanged?.Invoke(this, new OrderStatusChangedEventArgs
        {
            OrderId = OrderId,
            OldStatus = oldStatus,
            NewStatus = newStatus
        });
    }
}

public class OrderStatusChangedEventArgs : EventArgs
{
    public string OrderId { get; set; }
    public OrderStatus OldStatus { get; set; }
    public OrderStatus NewStatus { get; set; }
}

// Observers
public class EmailNotifier
{
    public void OnOrderStatusChanged(object sender, OrderStatusChangedEventArgs e)
    {
        Console.WriteLine($"Sending email: Order {e.OrderId} changed from {e.OldStatus} to {e.NewStatus}");
    }
}

public class InventoryManager
{
    public void OnOrderStatusChanged(object sender, OrderStatusChangedEventArgs e)
    {
        if (e.NewStatus == OrderStatus.Cancelled)
        {
            Console.WriteLine($"Releasing inventory for order {e.OrderId}");
        }
    }
}

public class AnalyticsTracker
{
    public void OnOrderStatusChanged(object sender, OrderStatusChangedEventArgs e)
    {
        Console.WriteLine($"Tracking: Order {e.OrderId} status change");
    }
}

// Usage
var order = new Order { OrderId = "ORD123" };

var emailNotifier = new EmailNotifier();
var inventoryManager = new InventoryManager();
var analyticsTracker = new AnalyticsTracker();

// Subscribe observers
order.StatusChanged += emailNotifier.OnOrderStatusChanged;
order.StatusChanged += inventoryManager.OnOrderStatusChanged;
order.StatusChanged += analyticsTracker.OnOrderStatusChanged;

// Trigger notifications
order.UpdateStatus(OrderStatus.Processing);
order.UpdateStatus(OrderStatus.Shipped);

// Unsubscribe
order.StatusChanged -= analyticsTracker.OnOrderStatusChanged;
```

---

## Command

### Purpose
Encapsulate a request as an object, allowing you to parameterize clients with different requests, queue or log requests, and support undoable operations.

### When to Use
- Undo/redo functionality
- Queuing operations
- Transactional behavior
- CQRS pattern

### C# Implementation

```csharp
// Command interface
public interface ICommand
{
    Task ExecuteAsync();
    Task UndoAsync();
}

// Concrete commands
public class CreateOrderCommand : ICommand
{
    private readonly IOrderRepository _repository;
    private readonly Order _order;
    private int _createdOrderId;

    public CreateOrderCommand(IOrderRepository repository, Order order)
    {
        _repository = repository;
        _order = order;
    }

    public async Task ExecuteAsync()
    {
        _createdOrderId = await _repository.CreateAsync(_order);
    }

    public async Task UndoAsync()
    {
        await _repository.DeleteAsync(_createdOrderId);
    }
}

public class UpdateInventoryCommand : ICommand
{
    private readonly IInventoryService _inventory;
    private readonly string _productId;
    private readonly int _quantity;

    public UpdateInventoryCommand(IInventoryService inventory, string productId, int quantity)
    {
        _inventory = inventory;
        _productId = productId;
        _quantity = quantity;
    }

    public async Task ExecuteAsync()
    {
        await _inventory.DecrementAsync(_productId, _quantity);
    }

    public async Task UndoAsync()
    {
        await _inventory.IncrementAsync(_productId, _quantity);
    }
}

// Command invoker with undo support
public class CommandInvoker
{
    private readonly Stack<ICommand> _executedCommands = new();

    public async Task ExecuteAsync(ICommand command)
    {
        await command.ExecuteAsync();
        _executedCommands.Push(command);
    }

    public async Task UndoLastAsync()
    {
        if (_executedCommands.Count > 0)
        {
            var command = _executedCommands.Pop();
            await command.UndoAsync();
        }
    }

    public async Task UndoAllAsync()
    {
        while (_executedCommands.Count > 0)
        {
            var command = _executedCommands.Pop();
            await command.UndoAsync();
        }
    }
}

// Usage
var invoker = new CommandInvoker();

try
{
    await invoker.ExecuteAsync(new CreateOrderCommand(orderRepo, order));
    await invoker.ExecuteAsync(new UpdateInventoryCommand(inventory, "PROD1", 5));
    await invoker.ExecuteAsync(new ChargePaymentCommand(payment, order.Total));
}
catch (Exception)
{
    // Rollback all executed commands
    await invoker.UndoAllAsync();
    throw;
}
```

---

## Chain of Responsibility

### Purpose
Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it.

### C# Implementation (ASP.NET Core Middleware Style)

```csharp
// Handler interface
public interface IRequestHandler
{
    IRequestHandler SetNext(IRequestHandler handler);
    Task<Response> HandleAsync(Request request);
}

// Base handler
public abstract class BaseHandler : IRequestHandler
{
    private IRequestHandler _nextHandler;

    public IRequestHandler SetNext(IRequestHandler handler)
    {
        _nextHandler = handler;
        return handler;
    }

    public virtual async Task<Response> HandleAsync(Request request)
    {
        if (_nextHandler != null)
            return await _nextHandler.HandleAsync(request);

        return new Response { Success = true };
    }
}

// Concrete handlers
public class AuthenticationHandler : BaseHandler
{
    public override async Task<Response> HandleAsync(Request request)
    {
        if (string.IsNullOrEmpty(request.AuthToken))
        {
            return new Response { Success = false, Error = "Not authenticated" };
        }

        // Validate token...
        request.UserId = "user123"; // Set from token

        return await base.HandleAsync(request);
    }
}

public class AuthorizationHandler : BaseHandler
{
    public override async Task<Response> HandleAsync(Request request)
    {
        if (!await HasPermissionAsync(request.UserId, request.Resource))
        {
            return new Response { Success = false, Error = "Not authorized" };
        }

        return await base.HandleAsync(request);
    }

    private Task<bool> HasPermissionAsync(string userId, string resource)
        => Task.FromResult(true);
}

public class ValidationHandler : BaseHandler
{
    public override async Task<Response> HandleAsync(Request request)
    {
        if (request.Data == null)
        {
            return new Response { Success = false, Error = "Invalid request data" };
        }

        return await base.HandleAsync(request);
    }
}

public class RateLimitHandler : BaseHandler
{
    public override async Task<Response> HandleAsync(Request request)
    {
        if (await IsRateLimitedAsync(request.UserId))
        {
            return new Response { Success = false, Error = "Rate limit exceeded" };
        }

        return await base.HandleAsync(request);
    }

    private Task<bool> IsRateLimitedAsync(string userId)
        => Task.FromResult(false);
}

// Build the chain
var handler = new AuthenticationHandler();
handler
    .SetNext(new RateLimitHandler())
    .SetNext(new AuthorizationHandler())
    .SetNext(new ValidationHandler());

// Use the chain
var response = await handler.HandleAsync(request);
```

---

## Mediator (MediatR)

### Purpose
Define an object that encapsulates how a set of objects interact. Promotes loose coupling by keeping objects from referring to each other explicitly.

### C# Implementation (MediatR Library)

```bash
dotnet add package MediatR
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection
```

```csharp
// Request (Command)
public class CreateOrderCommand : IRequest<OrderResult>
{
    public string CustomerId { get; set; }
    public List<OrderItemDto> Items { get; set; }
    public string PaymentMethod { get; set; }
}

// Response
public class OrderResult
{
    public bool Success { get; set; }
    public string OrderId { get; set; }
    public string Error { get; set; }
}

// Handler
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, OrderResult>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IPaymentService _paymentService;

    public CreateOrderHandler(IOrderRepository orderRepository, IPaymentService paymentService)
    {
        _orderRepository = orderRepository;
        _paymentService = paymentService;
    }

    public async Task<OrderResult> Handle(CreateOrderCommand request, CancellationToken cancellationToken)
    {
        var order = new Order
        {
            CustomerId = request.CustomerId,
            Items = request.Items.Select(i => new OrderItem(i.ProductId, i.Quantity)).ToList()
        };

        await _orderRepository.CreateAsync(order);

        return new OrderResult { Success = true, OrderId = order.Id };
    }
}

// Query
public class GetOrderQuery : IRequest<OrderDto>
{
    public string OrderId { get; set; }
}

public class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDto>
{
    private readonly IOrderRepository _repository;

    public GetOrderHandler(IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task<OrderDto> Handle(GetOrderQuery request, CancellationToken cancellationToken)
    {
        var order = await _repository.GetByIdAsync(request.OrderId);
        return OrderDto.FromEntity(order);
    }
}

// Notification (Event)
public class OrderCreatedNotification : INotification
{
    public string OrderId { get; set; }
    public string CustomerId { get; set; }
}

// Multiple handlers for same notification
public class SendOrderConfirmationEmail : INotificationHandler<OrderCreatedNotification>
{
    public async Task Handle(OrderCreatedNotification notification, CancellationToken cancellationToken)
    {
        // Send email
    }
}

public class UpdateInventory : INotificationHandler<OrderCreatedNotification>
{
    public async Task Handle(OrderCreatedNotification notification, CancellationToken cancellationToken)
    {
        // Update inventory
    }
}

// Usage in Controller
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrdersController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderCommand command)
    {
        var result = await _mediator.Send(command);

        if (result.Success)
        {
            // Publish notification
            await _mediator.Publish(new OrderCreatedNotification
            {
                OrderId = result.OrderId,
                CustomerId = command.CustomerId
            });

            return Ok(result);
        }

        return BadRequest(result.Error);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(string id)
    {
        var order = await _mediator.Send(new GetOrderQuery { OrderId = id });
        return order != null ? Ok(order) : NotFound();
    }
}

// DI Registration
services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
```

---

## State

### Purpose
Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

### C# Implementation

```csharp
// State interface
public interface IOrderState
{
    string Name { get; }
    bool CanCancel { get; }
    bool CanShip { get; }

    IOrderState Confirm(Order order);
    IOrderState Ship(Order order);
    IOrderState Deliver(Order order);
    IOrderState Cancel(Order order);
}

// Concrete states
public class PendingState : IOrderState
{
    public string Name => "Pending";
    public bool CanCancel => true;
    public bool CanShip => false;

    public IOrderState Confirm(Order order)
    {
        order.ConfirmedAt = DateTime.UtcNow;
        return new ConfirmedState();
    }

    public IOrderState Ship(Order order)
        => throw new InvalidOperationException("Cannot ship pending order");

    public IOrderState Deliver(Order order)
        => throw new InvalidOperationException("Cannot deliver pending order");

    public IOrderState Cancel(Order order)
    {
        order.CancelledAt = DateTime.UtcNow;
        return new CancelledState();
    }
}

public class ConfirmedState : IOrderState
{
    public string Name => "Confirmed";
    public bool CanCancel => true;
    public bool CanShip => true;

    public IOrderState Confirm(Order order)
        => throw new InvalidOperationException("Already confirmed");

    public IOrderState Ship(Order order)
    {
        order.ShippedAt = DateTime.UtcNow;
        return new ShippedState();
    }

    public IOrderState Deliver(Order order)
        => throw new InvalidOperationException("Must ship first");

    public IOrderState Cancel(Order order)
    {
        order.CancelledAt = DateTime.UtcNow;
        // Refund logic here
        return new CancelledState();
    }
}

public class ShippedState : IOrderState
{
    public string Name => "Shipped";
    public bool CanCancel => false;
    public bool CanShip => false;

    public IOrderState Confirm(Order order)
        => throw new InvalidOperationException("Already shipped");

    public IOrderState Ship(Order order)
        => throw new InvalidOperationException("Already shipped");

    public IOrderState Deliver(Order order)
    {
        order.DeliveredAt = DateTime.UtcNow;
        return new DeliveredState();
    }

    public IOrderState Cancel(Order order)
        => throw new InvalidOperationException("Cannot cancel shipped order");
}

// Order (Context)
public class Order
{
    private IOrderState _state = new PendingState();

    public string OrderId { get; set; }
    public string Status => _state.Name;
    public DateTime? ConfirmedAt { get; set; }
    public DateTime? ShippedAt { get; set; }
    public DateTime? DeliveredAt { get; set; }
    public DateTime? CancelledAt { get; set; }

    public bool CanCancel => _state.CanCancel;
    public bool CanShip => _state.CanShip;

    public void Confirm() => _state = _state.Confirm(this);
    public void Ship() => _state = _state.Ship(this);
    public void Deliver() => _state = _state.Deliver(this);
    public void Cancel() => _state = _state.Cancel(this);
}

// Usage
var order = new Order { OrderId = "ORD123" };
Console.WriteLine(order.Status); // "Pending"

order.Confirm();
Console.WriteLine(order.Status); // "Confirmed"

order.Ship();
Console.WriteLine(order.Status); // "Shipped"
Console.WriteLine(order.CanCancel); // false

order.Deliver();
Console.WriteLine(order.Status); // "Delivered"
```

---

## Template Method

### Purpose
Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Lets subclasses redefine certain steps of an algorithm without changing its structure.

### C# Implementation

```csharp
public abstract class ReportGenerator
{
    // Template method - defines the algorithm structure
    public async Task<Report> GenerateReportAsync(ReportRequest request)
    {
        var report = new Report { GeneratedAt = DateTime.UtcNow };

        // Step 1: Fetch data (varies by report type)
        var data = await FetchDataAsync(request);

        // Step 2: Validate data (common logic with hook)
        ValidateData(data);

        // Step 3: Transform data (varies by report type)
        var transformedData = TransformData(data);

        // Step 4: Format output (varies by report type)
        report.Content = FormatOutput(transformedData);

        // Step 5: Optional post-processing hook
        await PostProcessAsync(report);

        return report;
    }

    // Abstract methods - must be implemented
    protected abstract Task<object> FetchDataAsync(ReportRequest request);
    protected abstract object TransformData(object data);
    protected abstract string FormatOutput(object data);

    // Virtual methods with default implementation (hooks)
    protected virtual void ValidateData(object data)
    {
        if (data == null)
            throw new InvalidOperationException("No data available");
    }

    protected virtual Task PostProcessAsync(Report report)
    {
        return Task.CompletedTask;
    }
}

// Concrete implementations
public class SalesReportGenerator : ReportGenerator
{
    private readonly ISalesRepository _repository;

    public SalesReportGenerator(ISalesRepository repository)
    {
        _repository = repository;
    }

    protected override async Task<object> FetchDataAsync(ReportRequest request)
    {
        return await _repository.GetSalesAsync(request.StartDate, request.EndDate);
    }

    protected override object TransformData(object data)
    {
        var sales = (List<Sale>)data;
        return sales.GroupBy(s => s.ProductCategory)
                   .Select(g => new { Category = g.Key, Total = g.Sum(s => s.Amount) })
                   .ToList();
    }

    protected override string FormatOutput(object data)
    {
        return JsonSerializer.Serialize(data);
    }

    protected override async Task PostProcessAsync(Report report)
    {
        // Sales-specific: also send to analytics
        await _analytics.TrackReportGenerated("sales", report.GeneratedAt);
    }
}

public class InventoryReportGenerator : ReportGenerator
{
    protected override async Task<object> FetchDataAsync(ReportRequest request)
    {
        return await _inventoryRepo.GetCurrentStockAsync();
    }

    protected override object TransformData(object data)
    {
        var inventory = (List<InventoryItem>)data;
        return inventory.Where(i => i.Quantity < i.ReorderLevel).ToList();
    }

    protected override string FormatOutput(object data)
    {
        var lowStock = (List<InventoryItem>)data;
        return string.Join("\n", lowStock.Select(i => $"{i.ProductName}: {i.Quantity} units"));
    }
}
```

---

# Enterprise/Architectural Patterns

---

## Repository & Unit of Work

### C# Implementation

```csharp
// Generic repository interface
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate);
    Task AddAsync(T entity);
    void Update(T entity);
    void Remove(T entity);
}

// Unit of Work interface
public interface IUnitOfWork : IDisposable
{
    IRepository<Order> Orders { get; }
    IRepository<Product> Products { get; }
    IRepository<Customer> Customers { get; }
    Task<int> SaveChangesAsync();
}

// Generic repository implementation
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly DbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(DbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id)
        => await _dbSet.FindAsync(id);

    public async Task<IEnumerable<T>> GetAllAsync()
        => await _dbSet.ToListAsync();

    public async Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate)
        => await _dbSet.Where(predicate).ToListAsync();

    public async Task AddAsync(T entity)
        => await _dbSet.AddAsync(entity);

    public void Update(T entity)
        => _dbSet.Update(entity);

    public void Remove(T entity)
        => _dbSet.Remove(entity);
}

// Unit of Work implementation
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public UnitOfWork(AppDbContext context)
    {
        _context = context;
        Orders = new Repository<Order>(_context);
        Products = new Repository<Product>(_context);
        Customers = new Repository<Customer>(_context);
    }

    public IRepository<Order> Orders { get; }
    public IRepository<Product> Products { get; }
    public IRepository<Customer> Customers { get; }

    public async Task<int> SaveChangesAsync()
        => await _context.SaveChangesAsync();

    public void Dispose()
        => _context.Dispose();
}

// Usage
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;

    public OrderService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task CreateOrderAsync(OrderDto dto)
    {
        var customer = await _unitOfWork.Customers.GetByIdAsync(dto.CustomerId);
        var products = await _unitOfWork.Products.FindAsync(p => dto.ProductIds.Contains(p.Id));

        var order = new Order(customer, products);
        await _unitOfWork.Orders.AddAsync(order);

        // Single transaction for all changes
        await _unitOfWork.SaveChangesAsync();
    }
}
```

---

## Circuit Breaker (Polly)

### Purpose
Prevent repeated calls to a failing service. After a threshold of failures, "open" the circuit and fail fast without calling the service.

### C# Implementation

```bash
dotnet add package Polly
dotnet add package Microsoft.Extensions.Http.Polly
```

```csharp
// Basic circuit breaker
var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .OrResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
    .CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 3,  // Open after 3 failures
        durationOfBreak: TimeSpan.FromSeconds(30), // Stay open for 30s
        onBreak: (result, breakDelay) =>
        {
            Console.WriteLine($"Circuit opened for {breakDelay.TotalSeconds}s");
        },
        onReset: () =>
        {
            Console.WriteLine("Circuit reset");
        },
        onHalfOpen: () =>
        {
            Console.WriteLine("Circuit half-open, testing...");
        });

// With HttpClientFactory (recommended)
services.AddHttpClient("PaymentGateway")
    .AddPolicyHandler(GetCircuitBreakerPolicy());

private static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30));
}

// Advanced: Combined policies
services.AddHttpClient("PaymentGateway")
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());

private static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: retryAttempt =>
                TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))); // Exponential backoff
}
```

---

## Retry with Exponential Backoff

### C# Implementation

```csharp
// Manual implementation
public async Task<T> ExecuteWithRetryAsync<T>(
    Func<Task<T>> operation,
    int maxRetries = 3)
{
    for (int attempt = 0; attempt < maxRetries; attempt++)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex) when (attempt < maxRetries - 1 && IsTransient(ex))
        {
            var delay = TimeSpan.FromSeconds(Math.Pow(2, attempt)); // 1s, 2s, 4s
            await Task.Delay(delay);
        }
    }

    throw new InvalidOperationException("Should not reach here");
}

private bool IsTransient(Exception ex)
{
    return ex is HttpRequestException ||
           ex is TimeoutException ||
           (ex is TaskCanceledException tce && !tce.CancellationToken.IsCancellationRequested);
}

// Using Polly (recommended)
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .Or<TimeoutException>()
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: retryAttempt =>
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
        onRetry: (exception, timeSpan, retryCount, context) =>
        {
            Console.WriteLine($"Retry {retryCount} after {timeSpan.TotalSeconds}s due to {exception.Message}");
        });

// With jitter (prevents thundering herd)
var retryWithJitter = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: retryAttempt =>
        {
            var baseDelay = TimeSpan.FromSeconds(Math.Pow(2, retryAttempt));
            var jitter = TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000));
            return baseDelay + jitter;
        });
```

---

## Quick Reference

| Pattern | Purpose | Use When |
|---------|---------|----------|
| **Singleton** | Single instance | Config, logging, connection pools |
| **Factory** | Create objects | Unknown type at compile time |
| **Builder** | Complex construction | Many constructor parameters |
| **Adapter** | Interface conversion | Integrating legacy/third-party code |
| **Decorator** | Add behavior | Features that can be combined |
| **Facade** | Simplify subsystem | Complex subsystem access |
| **Proxy** | Control access | Caching, security, lazy loading |
| **Strategy** | Interchangeable algorithms | Multiple approaches to same task |
| **Observer** | Notify changes | Events, subscriptions |
| **Command** | Encapsulate requests | Undo/redo, queuing |
| **Chain of Responsibility** | Pass request along chain | Middleware, validation |
| **Mediator** | Centralize communication | Reduce coupling between objects |
| **State** | State-dependent behavior | Objects with distinct states |
| **Repository** | Data access abstraction | Database operations |
| **Circuit Breaker** | Prevent cascade failures | External service calls |
