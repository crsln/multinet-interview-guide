# .NET Current Version - Interview Questions

> 5 real-world interview questions with detailed answers and communication tactics

---

## Question 1: .NET 8 Features for Production

### The Question
> "What .NET 8 features would you use in a new production project? Why are these important?"

### Key Points to Cover
- LTS status and stability
- Performance improvements
- Developer productivity features
- Practical use cases

### Detailed Answer

**Why .NET 8 for Production:**

```
.NET 8 is LTS (Long Term Support):
- 3 years of support (until November 2026)
- Stable for enterprise applications
- Regular security updates
```

**Key Features I'd Use:**

**1. Frozen Collections - For High-Read Scenarios:**

```csharp
// Scenario: Configuration lookup, static reference data
// FrozenDictionary is optimized for read-heavy scenarios

// Old way - regular Dictionary
private static readonly Dictionary<string, decimal> TaxRates = new()
{
    ["TR"] = 0.18m,
    ["DE"] = 0.19m,
    ["UK"] = 0.20m
};

// .NET 8 way - FrozenDictionary
private static readonly FrozenDictionary<string, decimal> TaxRates =
    new Dictionary<string, decimal>
    {
        ["TR"] = 0.18m,
        ["DE"] = 0.19m,
        ["UK"] = 0.20m
    }.ToFrozenDictionary();

// Why use it:
// - Faster lookups (optimized hash function)
// - Thread-safe without locks (immutable)
// - Perfect for app configuration, country codes, etc.

// Real usage
public decimal GetTaxRate(string countryCode)
{
    return TaxRates.TryGetValue(countryCode, out var rate) ? rate : 0m;
}
```

**2. Keyed Services - Multiple Implementations:**

```csharp
// Scenario: Multiple payment gateways, environment-specific services

// Registration
builder.Services.AddKeyedScoped<IPaymentGateway, StripeGateway>("stripe");
builder.Services.AddKeyedScoped<IPaymentGateway, PayPalGateway>("paypal");
builder.Services.AddKeyedScoped<IPaymentGateway, IyzicoGateway>("iyzico");

// Injection by key
public class PaymentController : ControllerBase
{
    private readonly IPaymentGateway _stripeGateway;
    private readonly IPaymentGateway _paypalGateway;

    public PaymentController(
        [FromKeyedServices("stripe")] IPaymentGateway stripeGateway,
        [FromKeyedServices("paypal")] IPaymentGateway paypalGateway)
    {
        _stripeGateway = stripeGateway;
        _paypalGateway = paypalGateway;
    }
}

// Dynamic resolution
public class PaymentService
{
    private readonly IServiceProvider _serviceProvider;

    public async Task<PaymentResult> ProcessAsync(string gateway, PaymentRequest request)
    {
        var paymentGateway = _serviceProvider.GetRequiredKeyedService<IPaymentGateway>(gateway);
        return await paymentGateway.ProcessAsync(request);
    }
}
```

**3. TimeProvider - Testable Time:**

```csharp
// Scenario: Any code that depends on current time
public class OrderService
{
    private readonly TimeProvider _timeProvider;

    public OrderService(TimeProvider timeProvider)
    {
        _timeProvider = timeProvider;
    }

    public Order CreateOrder(Cart cart)
    {
        return new Order
        {
            CreatedAt = _timeProvider.GetUtcNow(),
            ExpiresAt = _timeProvider.GetUtcNow().AddDays(30),
            Items = cart.Items.ToList()
        };
    }

    public bool IsOrderExpired(Order order)
    {
        return _timeProvider.GetUtcNow() > order.ExpiresAt;
    }
}

// Production registration
services.AddSingleton(TimeProvider.System);

// Test with fake time
[Fact]
public void IsOrderExpired_AfterExpiryDate_ReturnsTrue()
{
    var fakeTime = new FakeTimeProvider();
    fakeTime.SetUtcNow(new DateTimeOffset(2024, 6, 1, 0, 0, 0, TimeSpan.Zero));

    var service = new OrderService(fakeTime);
    var order = service.CreateOrder(cart);

    // Advance time by 31 days
    fakeTime.Advance(TimeSpan.FromDays(31));

    Assert.True(service.IsOrderExpired(order));
}
```

**4. Native AOT - For Containers/Serverless:**

```xml
<!-- In .csproj -->
<PropertyGroup>
    <PublishAot>true</PublishAot>
</PropertyGroup>
```

```csharp
// Benefits:
// - Faster startup (no JIT compilation)
// - Smaller memory footprint
// - Smaller container images

// Limitations to mention:
// - No reflection at runtime
// - Some libraries don't support it
// - Larger binary size

// Good for: AWS Lambda, Azure Functions, CLI tools
// Not for: Complex apps with heavy reflection (EF Core migrations)
```

**5. Minimal API Improvements:**

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Route groups for organization
var api = app.MapGroup("/api/v1");

var orders = api.MapGroup("/orders")
    .RequireAuthorization()
    .AddEndpointFilter<ValidationFilter>();

orders.MapGet("/", GetOrders);
orders.MapGet("/{id}", GetOrderById);
orders.MapPost("/", CreateOrder);

// Typed results for OpenAPI
static Results<Ok<Order>, NotFound> GetOrderById(int id, OrderService service)
{
    var order = service.GetById(id);
    return order is not null
        ? TypedResults.Ok(order)
        : TypedResults.NotFound();
}
```

### Communication Tactics

🎯 **Structure your answer**: Focus on 3-4 features with real use cases. Don't just list features - explain WHEN you'd use them.

💡 **Emphasize**: .NET 8 is LTS - safe for production. Mention specific scenarios like "Frozen collections for configuration lookup" or "TimeProvider for testable code."

⚠️ **Avoid**: Don't list every new feature. Show you understand which ones matter for real applications.

---

## Question 2: Primary Constructors in C# 12

### The Question
> "Explain primary constructors in C# 12. Show me before and after code and explain when you'd use them."

### Key Points to Cover
- Syntax difference
- Parameter capture semantics
- When to use (and when not to)
- Comparison with records

### Detailed Answer

**Before C# 12 - Verbose Constructor:**

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IPaymentGateway _paymentGateway;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderRepository repository,
        IPaymentGateway paymentGateway,
        ILogger<OrderService> logger)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _paymentGateway = paymentGateway ?? throw new ArgumentNullException(nameof(paymentGateway));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public async Task<Order> GetOrderAsync(int id)
    {
        _logger.LogInformation("Getting order {Id}", id);
        return await _repository.GetByIdAsync(id);
    }
}
```

**After C# 12 - Primary Constructor:**

```csharp
public class OrderService(
    IOrderRepository repository,
    IPaymentGateway paymentGateway,
    ILogger<OrderService> logger)
{
    public async Task<Order> GetOrderAsync(int id)
    {
        logger.LogInformation("Getting order {Id}", id);
        return await repository.GetByIdAsync(id);
    }

    public async Task<PaymentResult> ProcessPaymentAsync(int orderId)
    {
        var order = await repository.GetByIdAsync(orderId);
        return await paymentGateway.ChargeAsync(order.Total);
    }
}
```

**Key Semantics:**

```csharp
// Primary constructor parameters are CAPTURED, not automatically fields
public class Example(IService service)
{
    // 'service' is accessible throughout the class
    // but it's not a field - it's a captured parameter

    public void DoWork()
    {
        service.Execute(); // ✅ Works
    }
}

// If you NEED a field (for readonly, or to modify):
public class Example(IService service)
{
    private readonly IService _service = service; // Explicit field

    public void DoWork()
    {
        _service.Execute();
    }
}

// Mixing primary constructor with regular constructors:
public class Order(int id, string customerId)
{
    public int Id { get; } = id;
    public string CustomerId { get; } = customerId;
    public List<OrderItem> Items { get; } = new();

    // Secondary constructor must call primary
    public Order(int id, string customerId, List<OrderItem> items)
        : this(id, customerId)
    {
        Items = items;
    }
}
```

**When to Use Primary Constructors:**

```csharp
// ✅ GOOD USE CASES:

// 1. Dependency injection in services
public class PaymentService(IPaymentGateway gateway, ILogger<PaymentService> logger)
{
    public async Task ProcessAsync(Payment payment)
    {
        logger.LogInformation("Processing payment");
        await gateway.ChargeAsync(payment.Amount);
    }
}

// 2. Simple DTOs and view models
public class OrderSummary(int id, decimal total, string status)
{
    public int Id { get; } = id;
    public decimal Total { get; } = total;
    public string Status { get; } = status;
}

// 3. Wrapper/Decorator classes
public class CachingRepository(IRepository inner, ICache cache) : IRepository
{
    public async Task<Order?> GetByIdAsync(int id)
    {
        var cached = await cache.GetAsync<Order>($"order:{id}");
        if (cached != null) return cached;

        var order = await inner.GetByIdAsync(id);
        if (order != null) await cache.SetAsync($"order:{id}", order);

        return order;
    }
}
```

```csharp
// ❌ AVOID WHEN:

// 1. You need validation in constructor
// Bad - validation not obvious
public class Money(decimal amount, string currency)
{
    public decimal Amount { get; } = amount >= 0 ? amount
        : throw new ArgumentException("Amount cannot be negative");
}

// Better - explicit constructor with validation
public class Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative", nameof(amount));
        if (string.IsNullOrEmpty(currency))
            throw new ArgumentException("Currency is required", nameof(currency));

        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }
}

// 2. Complex initialization logic
// 3. When you need to modify parameters
```

**Comparison with Records:**

```csharp
// Record - for immutable data objects with value equality
public record OrderDto(int Id, decimal Total, string Status);

// Primary constructor class - for services with behavior
public class OrderService(IOrderRepository repo)
{
    public async Task<Order> GetAsync(int id) => await repo.GetByIdAsync(id);
}

// When to use which:
// - Record: DTOs, value objects, immutable data
// - Primary constructor class: Services, handlers, decorators
```

### Communication Tactics

🎯 **Structure your answer**: Show clear before/after, explain the semantics (captured, not fields), give use cases.

💡 **Emphasize**: Primary constructors reduce boilerplate for DI scenarios. But for complex validation, traditional constructors are clearer.

⚠️ **Avoid**: Don't suggest using them everywhere. Show you understand when explicit constructors are better.

---

## Question 3: Native AOT Trade-offs

### The Question
> "When would you use Native AOT and what are the trade-offs? Give me a scenario where it's a good fit."

### Key Points to Cover
- What Native AOT is
- Benefits and limitations
- Good use cases
- What doesn't work with AOT

### Detailed Answer

**What is Native AOT:**

```
Traditional .NET:
Source Code → IL Code → JIT Compile at Runtime → Machine Code

Native AOT:
Source Code → IL Code → AOT Compile at Build → Native Binary

Result: No JIT needed at runtime, faster startup
```

**Benefits:**

```csharp
// 1. FASTER STARTUP
// Traditional: 500ms-2s to start (JIT compilation)
// Native AOT: 50-100ms (already compiled)

// 2. SMALLER MEMORY FOOTPRINT
// No JIT compiler loaded
// Trimmed unused code

// 3. SMALLER CONTAINER IMAGES
// Can use smaller base images (no .NET runtime needed)
// Alpine-based: ~50MB vs ~200MB

// 4. PREDICTABLE PERFORMANCE
// No JIT compilation pauses
// Consistent response times from first request
```

**Limitations:**

```csharp
// 1. NO RUNTIME REFLECTION (biggest limitation)
// This won't work:
var type = Type.GetType("MyClass");
var instance = Activator.CreateInstance(type);

// 2. NO DYNAMIC CODE GENERATION
// System.Reflection.Emit doesn't work
// Some serializers need workarounds

// 3. LARGER BINARY SIZE
// All code compiled upfront
// Typical: 20-50MB for a simple API

// 4. SOME LIBRARIES DON'T SUPPORT IT
// Must check compatibility
// Many popular libraries now support it
```

**Good Use Cases:**

```csharp
// ✅ PERFECT FOR:

// 1. AWS Lambda / Azure Functions (Serverless)
// Cold start matters - AOT reduces it dramatically
[assembly: LambdaSerializer(typeof(SourceGeneratorLambdaJsonSerializer<LambdaContext>))]

public class Function
{
    public string Handler(string input, ILambdaContext context)
    {
        return $"Hello, {input}!";
    }
}

// 2. CLI Tools
// Users expect instant response
dotnet publish -c Release -r win-x64 --self-contained -p:PublishAot=true

// 3. Microservices with Many Instances
// Faster scaling, lower memory per instance

// 4. Sidecar Containers
// Small footprint, quick startup
```

```csharp
// ❌ NOT SUITABLE FOR:

// 1. EF Core with Migrations
// Migrations use reflection heavily

// 2. Applications using MVC with Views
// Razor compilation needs runtime

// 3. Plugin Systems
// Can't load assemblies dynamically

// 4. Heavy Reflection Usage
// ASP.NET Identity, some DI containers
```

**Making Code AOT-Compatible:**

```csharp
// Use source generators for JSON serialization
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(List<Order>))]
public partial class AppJsonContext : JsonSerializerContext { }

// Registration
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolver = AppJsonContext.Default;
});

// Minimal API with AOT
var builder = WebApplication.CreateSlimBuilder(args);
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolver = AppJsonContext.Default;
});

var app = builder.Build();
app.MapGet("/orders/{id}", (int id) =>
    new Order { Id = id, Total = 100 });
app.Run();
```

### Communication Tactics

🎯 **Structure your answer**: Define AOT briefly, list benefits, list limitations, give clear use case (Lambda/CLI).

💡 **Emphasize**: "Native AOT is great for serverless and CLI tools where startup time matters. But for typical web APIs, the benefits may not justify the limitations."

⚠️ **Avoid**: Don't oversell it. Be clear about what doesn't work (reflection, EF migrations).

---

## Question 4: Keyed Services in DI

### The Question
> "How do Keyed Services work in .NET 8 dependency injection? Show me a practical example."

### Key Points to Cover
- Problem they solve
- Registration and resolution
- Practical use cases
- Comparison with alternatives

### Detailed Answer

**Problem Before Keyed Services:**

```csharp
// Before .NET 8 - Workarounds for multiple implementations

// Option 1: Factory pattern
public interface INotificationSenderFactory
{
    INotificationSender Create(string type);
}

// Option 2: Named registrations with wrapper
services.AddScoped<EmailSender>();
services.AddScoped<SmsSender>();
services.AddScoped<Func<string, INotificationSender>>(sp => key =>
{
    return key switch
    {
        "email" => sp.GetRequiredService<EmailSender>(),
        "sms" => sp.GetRequiredService<SmsSender>(),
        _ => throw new ArgumentException($"Unknown sender: {key}")
    };
});

// Option 3: Register all and filter
services.AddScoped<INotificationSender, EmailSender>();
services.AddScoped<INotificationSender, SmsSender>();
// Then inject IEnumerable<INotificationSender> and filter
```

**Keyed Services Solution (.NET 8):**

```csharp
// Clean registration with keys
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddKeyedScoped<INotificationSender, EmailSender>("email");
builder.Services.AddKeyedScoped<INotificationSender, SmsSender>("sms");
builder.Services.AddKeyedScoped<INotificationSender, PushSender>("push");

// Also works with singletons and transients
builder.Services.AddKeyedSingleton<ICache, RedisCache>("distributed");
builder.Services.AddKeyedSingleton<ICache, MemoryCache>("local");
```

**Injection Methods:**

```csharp
// Method 1: Constructor injection with attribute
public class NotificationService
{
    private readonly INotificationSender _emailSender;
    private readonly INotificationSender _smsSender;

    public NotificationService(
        [FromKeyedServices("email")] INotificationSender emailSender,
        [FromKeyedServices("sms")] INotificationSender smsSender)
    {
        _emailSender = emailSender;
        _smsSender = smsSender;
    }

    public async Task SendAllAsync(string message)
    {
        await _emailSender.SendAsync(message);
        await _smsSender.SendAsync(message);
    }
}

// Method 2: Dynamic resolution from IServiceProvider
public class DynamicNotificationService
{
    private readonly IServiceProvider _serviceProvider;

    public DynamicNotificationService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task SendAsync(string channel, string message)
    {
        var sender = _serviceProvider.GetRequiredKeyedService<INotificationSender>(channel);
        await sender.SendAsync(message);
    }
}

// Method 3: In Minimal APIs
app.MapPost("/notify/{channel}", async (
    string channel,
    string message,
    [FromKeyedServices("email")] INotificationSender emailSender) =>
{
    await emailSender.SendAsync(message);
    return Results.Ok();
});
```

**Practical Example - Multi-tenant Database:**

```csharp
// Scenario: Different database connections per tenant
builder.Services.AddKeyedScoped<IDbConnection>("tenant-a", sp =>
    new SqlConnection(configuration.GetConnectionString("TenantA")));

builder.Services.AddKeyedScoped<IDbConnection>("tenant-b", sp =>
    new SqlConnection(configuration.GetConnectionString("TenantB")));

// Middleware sets the tenant
public class TenantMiddleware
{
    public async Task InvokeAsync(HttpContext context, IServiceProvider sp)
    {
        var tenantId = context.Request.Headers["X-Tenant-Id"].ToString();
        var connection = sp.GetRequiredKeyedService<IDbConnection>(tenantId);

        context.Items["DbConnection"] = connection;
        await _next(context);
    }
}

// Or inject based on request context
public class TenantAwareRepository(
    [FromKeyedServices("tenant-a")] IDbConnection connectionA,
    [FromKeyedServices("tenant-b")] IDbConnection connectionB,
    IHttpContextAccessor httpContext)
{
    private IDbConnection GetConnection()
    {
        var tenant = httpContext.HttpContext?.Request.Headers["X-Tenant-Id"].ToString();
        return tenant == "tenant-a" ? connectionA : connectionB;
    }
}
```

**Comparison with Alternatives:**

| Approach | Pros | Cons |
|----------|------|------|
| Keyed Services | Clean, built-in, type-safe | .NET 8+ only |
| Factory Pattern | Works everywhere | More boilerplate |
| IEnumerable + filter | Simple | Must inject all implementations |
| Named Options | For configuration | Not for services |

### Communication Tactics

🎯 **Structure your answer**: Show the problem first, then the clean solution, then practical examples.

💡 **Emphasize**: "Keyed services solve the multiple-implementation problem elegantly. Great for payment gateways, notification channels, multi-tenant scenarios."

⚠️ **Avoid**: Don't suggest using keys for everything. Simple single-implementation scenarios don't need this.

---

## Question 5: FrozenDictionary vs Dictionary

### The Question
> "Compare FrozenDictionary to Dictionary. When would you choose each?"

### Key Points to Cover
- Performance characteristics
- Thread safety
- Use cases for each
- Memory considerations

### Detailed Answer

**Key Differences:**

```csharp
// Dictionary<TKey, TValue>
// - Mutable (can add/remove after creation)
// - Not thread-safe for writes
// - Optimized for general-purpose use

// FrozenDictionary<TKey, TValue>
// - Immutable (cannot change after creation)
// - Thread-safe (immutability guarantees this)
// - Optimized for read-heavy scenarios
```

**Performance Comparison:**

```csharp
// Creating
var dict = new Dictionary<string, int>
{
    ["one"] = 1,
    ["two"] = 2,
    ["three"] = 3
};

var frozen = dict.ToFrozenDictionary(); // Slower creation, but only once

// Lookup performance (the key difference)
// FrozenDictionary optimizes the hash function at creation time
// based on the actual keys - results in faster lookups

BenchmarkResults:
|          Method |      Mean |
|-----------------|-----------|
| Dictionary      |  15.32 ns |
| FrozenDictionary|   8.91 ns |  // ~40% faster lookups!
```

**When to Use Each:**

```csharp
// ✅ USE FrozenDictionary FOR:

// 1. Static configuration data
private static readonly FrozenDictionary<string, decimal> TaxRates =
    new Dictionary<string, decimal>
    {
        ["TR"] = 0.18m,
        ["DE"] = 0.19m,
        ["US-CA"] = 0.0725m,
        ["US-TX"] = 0.0625m
    }.ToFrozenDictionary();

// 2. Lookup tables loaded once at startup
public class CountryService
{
    private readonly FrozenDictionary<string, Country> _countries;

    public CountryService(ICountryRepository repo)
    {
        // Load once, never changes
        _countries = repo.GetAll().ToFrozenDictionary(c => c.Code);
    }

    public Country? GetByCode(string code) => _countries.GetValueOrDefault(code);
}

// 3. Parsing/validation with known values
private static readonly FrozenSet<string> ValidStatuses =
    new HashSet<string> { "pending", "processing", "completed", "cancelled" }
    .ToFrozenSet(StringComparer.OrdinalIgnoreCase);

public bool IsValidStatus(string status) => ValidStatuses.Contains(status);

// 4. High-throughput read scenarios
// API that looks up product info millions of times per day
```

```csharp
// ✅ USE Dictionary FOR:

// 1. Dynamic collections that change
public class ShoppingCart
{
    private readonly Dictionary<int, CartItem> _items = new();

    public void AddItem(int productId, int quantity)
    {
        if (_items.ContainsKey(productId))
            _items[productId].Quantity += quantity;
        else
            _items[productId] = new CartItem { ProductId = productId, Quantity = quantity };
    }

    public void RemoveItem(int productId)
    {
        _items.Remove(productId);
    }
}

// 2. Caches that need updating
public class SimpleCache<TKey, TValue>
{
    private readonly Dictionary<TKey, TValue> _cache = new();
    private readonly object _lock = new();

    public void Set(TKey key, TValue value)
    {
        lock (_lock)
        {
            _cache[key] = value;
        }
    }
}

// 3. Building data incrementally
public Dictionary<string, int> CountWords(string text)
{
    var counts = new Dictionary<string, int>();
    foreach (var word in text.Split(' '))
    {
        counts[word] = counts.GetValueOrDefault(word) + 1;
    }
    return counts;
}
```

**Memory Considerations:**

```csharp
// FrozenDictionary uses more memory per entry
// because it stores optimized lookup structures

// For small dictionaries (< 10 items): Regular Dictionary is fine
// For large, read-heavy dictionaries: FrozenDictionary shines

// Example: Country codes (195 entries, read millions of times)
// - Dictionary: Fast enough, less setup
// - FrozenDictionary: Even faster, worth the upfront cost

// Example: User session cache (changes frequently)
// - Dictionary: Better choice (need mutability)
// - FrozenDictionary: Not applicable
```

**Thread Safety:**

```csharp
// Dictionary - NOT thread-safe for writes
// Must use locks or ConcurrentDictionary

private readonly Dictionary<string, int> _cache = new();
private readonly object _lock = new();

public int GetOrAdd(string key, Func<int> factory)
{
    lock (_lock)  // Required for thread safety
    {
        if (!_cache.TryGetValue(key, out var value))
        {
            value = factory();
            _cache[key] = value;
        }
        return value;
    }
}

// FrozenDictionary - Thread-safe by immutability
// No locks needed - can't be modified!

private static readonly FrozenDictionary<string, Config> _config = LoadConfig();

public Config GetConfig(string key)
{
    return _config[key];  // No lock needed, multiple threads safe
}
```

### Communication Tactics

🎯 **Structure your answer**: Compare key characteristics, show performance difference, give clear use cases for each.

💡 **Emphasize**: "FrozenDictionary is for static, read-heavy data. Regular Dictionary is for dynamic collections. Choose based on mutability needs."

⚠️ **Avoid**: Don't suggest replacing all dictionaries. The creation overhead of FrozenDictionary only pays off for truly static data with many reads.

---

## Question 6: What's New in .NET 9

### The Question
> "What's new in .NET 9 that excites you?"

### Key Points to Cover
- New features and improvements
- Practical use cases
- Backward compatibility considerations

### Detailed Answer

**.NET 9 Highlights (Released November 2024):**

```csharp
// 1. Task.WhenEach - Process tasks as they complete
// Instead of waiting for all, process results as they arrive

var tasks = new[]
{
    FetchDataFromServiceA(),
    FetchDataFromServiceB(),
    FetchDataFromServiceC()
};

// Old way - wait for all
// var results = await Task.WhenAll(tasks);

// .NET 9 way - process as each completes
await foreach (var task in Task.WhenEach(tasks))
{
    var result = await task;  // Already completed, no wait
    ProcessResult(result);     // Start processing immediately
}

// Great for: Dashboard loading, parallel API calls, search aggregation
```

```csharp
// 2. New Lock type - Better synchronization API
// More intuitive than Monitor-based locking

private readonly Lock _lock = new();

public void ThreadSafeMethod()
{
    lock (_lock)  // Compiler recognizes Lock type
    {
        // Critical section
    }

    // Or explicit scoping
    using (_lock.EnterScope())
    {
        // Critical section
    }
}

// Benefits: Cleaner API, better performance than object locking
```

```csharp
// 3. Base64Url class - URL-safe encoding
// No more manual replacement of +/= characters

// Old way
var base64 = Convert.ToBase64String(data)
    .Replace('+', '-')
    .Replace('/', '_')
    .TrimEnd('=');

// .NET 9 way
var urlSafe = Base64Url.EncodeToString(data);
var decoded = Base64Url.DecodeFromChars(urlSafe);

// Great for: JWT tokens, URL parameters, file names
```

```csharp
// 4. LINQ improvements
// CountBy - count occurrences by key

var words = new[] { "apple", "banana", "apple", "cherry", "banana", "apple" };

// Old way
var counts = words.GroupBy(w => w).ToDictionary(g => g.Key, g => g.Count());

// .NET 9 way
var counts = words.CountBy(w => w);
// { "apple": 3, "banana": 2, "cherry": 1 }

// AggregateBy - aggregate by key
var orders = GetOrders();
var totalsByCustomer = orders.AggregateBy(
    o => o.CustomerId,
    0m,
    (total, order) => total + order.Amount);
```

**What Excites Me Most:**

> "Task.WhenEach is what excites me most. In payment systems, we often call multiple services in parallel - fraud check, balance check, rate lookup. With WhenEach, we can start processing results as soon as each completes, rather than waiting for the slowest one.
>
> The new Lock type is also great for code clarity. It's more intuitive than the old Monitor-based locking, and the performance improvements are nice for high-contention scenarios."

### Communication Tactics

- **Show practical application**: Not just features, but when you'd use them
- **Mention it's STS**: 18-month support, not LTS like .NET 8
- **Connect to real scenarios**: Parallel API calls, thread safety

---

## Question 7: IEnumerable vs ICollection vs IList vs IQueryable

### The Question
> "Explain the difference between IEnumerable, ICollection, IList, and IQueryable."

### Key Points to Cover
- Interface hierarchy
- When to use each
- Performance implications
- Common mistakes

### Detailed Answer

**Interface Hierarchy:**

```
IEnumerable<T>           ← Most basic: iteration only
    │
    └── ICollection<T>   ← Adds: Count, Add, Remove, Contains
            │
            └── IList<T> ← Adds: Index access, Insert, RemoveAt

IQueryable<T>            ← Separate branch: deferred execution with expression trees
    │
    └── (extends IEnumerable<T>)
```

**Comparison Table:**

| Interface | Capabilities | Best For |
|-----------|-------------|----------|
| `IEnumerable<T>` | Forward iteration only | Read-only sequences, streaming |
| `ICollection<T>` | + Count, Add, Remove | Modification without index |
| `IList<T>` | + Index access `[i]` | Random access by position |
| `IQueryable<T>` | Expression tree building | Database queries (EF Core) |

**When to Use Each:**

```csharp
// IEnumerable<T> - When you only need to iterate
public IEnumerable<Order> GetRecentOrders()
{
    return _orders.Where(o => o.Date > DateTime.Today.AddDays(-7));
    // Caller can foreach, but can't modify or count without iterating
}

// ICollection<T> - When you need Count or modification
public void ProcessOrders(ICollection<Order> orders)
{
    Console.WriteLine($"Processing {orders.Count} orders");  // O(1) Count
    orders.Add(new Order());  // Can modify
}

// IList<T> - When you need index access
public void UpdateOrder(IList<Order> orders, int index)
{
    orders[index].Status = OrderStatus.Processed;  // Direct access
    orders.Insert(0, priorityOrder);  // Insert at position
}

// IQueryable<T> - For database queries (expression trees)
public IQueryable<Order> GetOrdersQuery()
{
    return _context.Orders.Where(o => o.Amount > 100);
    // NOT executed yet - builds expression tree
    // SQL generated when enumerated
}
```

**Common Mistakes:**

```csharp
// MISTAKE 1: Using IQueryable when you mean IEnumerable
public IQueryable<Order> GetOrders()
{
    return _context.Orders.ToList().AsQueryable();  // BAD
    // Already materialized, AsQueryable is pointless
}

// MISTAKE 2: Multiple enumeration of IEnumerable
public void Process(IEnumerable<Order> orders)
{
    var count = orders.Count();      // Enumerates entire sequence
    foreach (var order in orders)    // Enumerates AGAIN
    {
        // ...
    }
    // If orders is a database query, this runs the query TWICE!
}

// BETTER: Materialize once if you need multiple operations
public void Process(IEnumerable<Order> orders)
{
    var orderList = orders.ToList();  // Single enumeration
    var count = orderList.Count;       // O(1)
    foreach (var order in orderList)   // Uses same list
    {
        // ...
    }
}

// MISTAKE 3: Returning IList when you don't need modification
public IList<Order> GetOrders()  // Overly permissive
public IEnumerable<Order> GetOrders()  // Better - no implied mutability
```

**Method Parameter Guidelines:**

```csharp
// Accept the most general type that works
public void ProcessOrders(IEnumerable<Order> orders)  // Accept any sequence

// Return the most specific type that's useful
public IReadOnlyList<Order> GetOrders()  // Caller knows they can index and count
```

### Communication Tactics

- **Show the hierarchy**: IEnumerable → ICollection → IList
- **Emphasize IQueryable**: Expression trees, database queries
- **Mention common mistakes**: Multiple enumeration, ToList().AsQueryable()

---

## Question 8: Garbage Collection in .NET

### The Question
> "How does garbage collection work in .NET? Explain generations and the Large Object Heap."

### Key Points to Cover
- Generational GC concept
- Gen 0, 1, 2 behavior
- Large Object Heap (LOH)
- Performance implications

### Detailed Answer

**Generational Garbage Collection:**

```
WHY GENERATIONS?
────────────────
Observation: Most objects die young (short-lived)
Optimization: Check young objects frequently, old objects rarely

Generation 0 (Gen 0)
├── Newest objects
├── Collected most frequently (every few milliseconds)
├── Objects that survive → promoted to Gen 1
└── Size: ~256 KB - 4 MB

Generation 1 (Gen 1)
├── Objects that survived Gen 0 collection
├── Collected less frequently
├── Buffer between young and old
└── Objects that survive → promoted to Gen 2

Generation 2 (Gen 2)
├── Long-lived objects
├── Collected least frequently (can cause pauses)
├── Full GC includes Gen 2
└── Can grow large if objects aren't collected
```

**GC Process:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         GC PROCESS                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. MARK PHASE                                                           │
│     - Start from "roots" (stack variables, static fields)               │
│     - Traverse all reachable objects                                    │
│     - Mark them as "alive"                                              │
│                                                                          │
│  2. SWEEP PHASE                                                          │
│     - Unmarked objects = garbage                                        │
│     - Memory reclaimed                                                  │
│                                                                          │
│  3. COMPACT PHASE (sometimes)                                           │
│     - Move surviving objects together                                   │
│     - Eliminates fragmentation                                          │
│     - Updates references to new locations                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Large Object Heap (LOH):**

```csharp
// Objects >= 85,000 bytes go to LOH
// LOH is NOT compacted by default (expensive for large objects)

// This goes on LOH:
var largeArray = new byte[85000];  // >= 85KB

// LOH issues:
// 1. Collected only during Gen 2 collections (infrequent)
// 2. Fragmentation over time (holes between objects)
// 3. Can cause OutOfMemoryException even with "free" space

// Mitigation strategies:
// 1. Use ArrayPool<T> to reuse large arrays
var buffer = ArrayPool<byte>.Shared.Rent(100000);
try
{
    // Use buffer
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}

// 2. Enable LOH compaction (expensive, use sparingly)
GCSettings.LargeObjectHeapCompactionMode =
    GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect();
```

**Server vs Workstation GC:**

```csharp
// Workstation GC (default)
// - Single GC thread
// - Lower latency
// - For client apps

// Server GC
// - One GC thread per CPU core
// - Higher throughput
// - For server apps

// Enable in csproj:
<PropertyGroup>
    <ServerGarbageCollection>true</ServerGarbageCollection>
</PropertyGroup>

// Or in runtimeconfig.json:
{
    "runtimeOptions": {
        "configProperties": {
            "System.GC.Server": true
        }
    }
}
```

**Performance Tips:**

```csharp
// 1. Avoid allocations in hot paths
// BAD: Creates new string on each call
public string GetFullName() => $"{FirstName} {LastName}";

// BETTER: Reuse StringBuilder or cache result
private string _fullName;
public string GetFullName() => _fullName ??= $"{FirstName} {LastName}";

// 2. Use Span<T> and stackalloc for temporary buffers
Span<byte> buffer = stackalloc byte[256];  // Stack, not heap

// 3. Reduce Gen 2 collections
// - Object pooling for frequently created/destroyed objects
// - Avoid holding references longer than needed

// 4. Monitor GC metrics
// - % Time in GC (should be < 10%)
// - Gen 2 collections (should be rare)
// - LOH size growth
```

### Communication Tactics

- **Show understanding of generations**: Why they exist, how they work
- **Mention LOH**: Shows deeper knowledge
- **Give practical tips**: ArrayPool, Server GC, monitoring

---

## Quick Review - Key Takeaways

| Question | Key Point |
|----------|-----------|
| .NET 8 Features | FrozenCollections, KeyedServices, TimeProvider - know USE CASES |
| Primary Constructors | Great for DI, avoid for complex validation |
| Native AOT | Perfect for Lambda/CLI, not for EF migrations |
| Keyed Services | Solves multiple-implementation problem cleanly |
| Frozen vs Dictionary | Frozen = static, read-heavy; Dictionary = dynamic |
| .NET 9 Features | Task.WhenEach, Lock type, Base64Url, CountBy |
| Collection Interfaces | IEnumerable → ICollection → IList; IQueryable for EF |
| Garbage Collection | Generational GC, LOH for large objects, use pooling |
