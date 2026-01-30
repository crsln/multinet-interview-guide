# .NET Current Version Knowledge

> Knowledge of .NET 8 (LTS) and .NET 9, C# 12/13 features

---

## .NET Version Overview

| Version | Release | Support | Status |
|---------|---------|---------|--------|
| .NET 9 | Nov 2024 | STS (18 months) | Current |
| .NET 8 | Nov 2023 | LTS (3 years) | **Recommended for Production** |
| .NET 7 | Nov 2022 | Ended May 2024 | End of Life |
| .NET 6 | Nov 2021 | LTS - Nov 2024 | End of Life |

> **LTS** = Long Term Support (3 years), **STS** = Standard Term Support (18 months)

---

## .NET 8 Key Features (LTS - Know These Well)

### Performance Improvements

```csharp
// Frozen Collections - Optimized for read-heavy scenarios
var frozenDict = new Dictionary<string, int>
{
    ["one"] = 1,
    ["two"] = 2,
    ["three"] = 3
}.ToFrozenDictionary();

// Much faster lookups than regular Dictionary
var value = frozenDict["two"];

// FrozenSet for unique values
var frozenSet = new HashSet<string> { "a", "b", "c" }.ToFrozenSet();
```

### Native AOT (Ahead-of-Time) Compilation

```xml
<!-- In .csproj -->
<PropertyGroup>
    <PublishAot>true</PublishAot>
</PropertyGroup>
```

```bash
# Publish with AOT
dotnet publish -c Release -r win-x64
```

Benefits:
- Faster startup time
- Smaller memory footprint
- No JIT compilation at runtime
- Better for containers and serverless

Limitations:
- No dynamic code generation
- Reflection limitations
- Larger binary size

### Minimal APIs Enhancements

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Route groups for organization
var api = app.MapGroup("/api");
var v1 = api.MapGroup("/v1");

v1.MapGet("/products", GetProducts);
v1.MapGet("/products/{id}", GetProductById);
v1.MapPost("/products", CreateProduct);

// With filters and metadata
v1.MapGet("/products/{id}", GetProductById)
    .WithName("GetProduct")
    .WithOpenApi()
    .RequireAuthorization()
    .AddEndpointFilter<ValidationFilter>();

app.Run();

// Type-safe route parameters
static IResult GetProductById(int id) =>
    Results.Ok(new { Id = id, Name = "Product" });

// Typed results
static Results<Ok<Product>, NotFound> GetProduct(int id, ProductDb db)
{
    var product = db.Find(id);
    return product is not null
        ? TypedResults.Ok(product)
        : TypedResults.NotFound();
}
```

### Time Abstraction

```csharp
// New TimeProvider abstraction for testable time-dependent code
public class OrderService
{
    private readonly TimeProvider _timeProvider;

    public OrderService(TimeProvider timeProvider)
    {
        _timeProvider = timeProvider;
    }

    public Order CreateOrder()
    {
        return new Order
        {
            CreatedAt = _timeProvider.GetUtcNow(),
            ExpiresAt = _timeProvider.GetUtcNow().AddDays(30)
        };
    }
}

// In production
services.AddSingleton(TimeProvider.System);

// In tests
var fakeTime = new FakeTimeProvider(new DateTimeOffset(2024, 1, 15, 0, 0, 0, TimeSpan.Zero));
var service = new OrderService(fakeTime);

fakeTime.Advance(TimeSpan.FromDays(31)); // Simulate time passing
```

### Keyed Services in DI

```csharp
// Register keyed services
builder.Services.AddKeyedScoped<IPaymentGateway, StripeGateway>("stripe");
builder.Services.AddKeyedScoped<IPaymentGateway, PayPalGateway>("paypal");

// Inject by key
public class PaymentService
{
    public PaymentService(
        [FromKeyedServices("stripe")] IPaymentGateway stripeGateway,
        [FromKeyedServices("paypal")] IPaymentGateway paypalGateway)
    {
        // Use appropriate gateway
    }
}

// Or resolve dynamically
public class PaymentFactory
{
    private readonly IServiceProvider _serviceProvider;

    public IPaymentGateway GetGateway(string provider)
    {
        return _serviceProvider.GetRequiredKeyedService<IPaymentGateway>(provider);
    }
}
```

### Configuration Binding Source Generator

```csharp
// Faster configuration binding with source generators
builder.Services.Configure<MySettings>(
    builder.Configuration.GetSection("MySettings"));

// Enable source generator in .csproj
// <EnableConfigurationBindingGenerator>true</EnableConfigurationBindingGenerator>
```

### Improved JSON Serialization

```csharp
// New source generator improvements
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(List<Product>))]
public partial class AppJsonContext : JsonSerializerContext
{
}

// Missing member handling
var options = new JsonSerializerOptions
{
    UnmappedMemberHandling = JsonUnmappedMemberHandling.Disallow
};

// Interface hierarchy support
var json = JsonSerializer.Serialize<IAnimal>(new Dog());
```

---

## .NET 9 Features (Latest)

### LINQ Improvements

```csharp
// New CountBy method
var counts = products.CountBy(p => p.Category);
// Returns IEnumerable<KeyValuePair<string, int>>

// New AggregateBy method
var totals = products.AggregateBy(
    p => p.Category,
    decimal.Zero,
    (total, product) => total + product.Price);

// Index method - enumerate with index
foreach (var (index, item) in items.Index())
{
    Console.WriteLine($"{index}: {item}");
}
```

### Performance & Optimizations

```csharp
// Improved Regex source generator
[GeneratedRegex(@"\d+", RegexOptions.Compiled)]
private static partial Regex NumbersRegex();

// New SearchValues for multi-value searching
private static readonly SearchValues<string> Keywords =
    SearchValues.Create(["async", "await", "Task"], StringComparison.OrdinalIgnoreCase);

bool containsKeyword = Keywords.ContainsAny(sourceCode.AsSpan());
```

### JSON Improvements

```csharp
// Indented JSON formatting options
var options = new JsonSerializerOptions
{
    WriteIndented = true,
    IndentCharacter = '\t',
    IndentSize = 1
};

// New JsonSchemaExporter
var schema = JsonSchemaExporter.GetJsonSchema<Person>();
```

### ASP.NET Core 9 Features

```csharp
// OpenAPI document generation built-in
builder.Services.AddOpenApi();

app.MapOpenApi(); // Serves OpenAPI document at /openapi/v1.json

// HybridCache (combines memory + distributed cache)
builder.Services.AddHybridCache(options =>
{
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        LocalCacheExpiration = TimeSpan.FromMinutes(5),
        Expiration = TimeSpan.FromMinutes(30)
    };
});

public class ProductService
{
    private readonly HybridCache _cache;

    public async Task<Product> GetProductAsync(int id, CancellationToken ct)
    {
        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async token => await _repository.GetByIdAsync(id, token),
            cancellationToken: ct);
    }
}
```

---

## C# 12 Features (with .NET 8)

### Primary Constructors for Classes

```csharp
// Old way
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IOrderRepository repository, ILogger<OrderService> logger)
    {
        _repository = repository;
        _logger = logger;
    }
}

// C# 12 way - Primary constructor
public class OrderService(IOrderRepository repository, ILogger<OrderService> logger)
{
    public async Task<Order> GetOrderAsync(int id)
    {
        logger.LogInformation("Getting order {Id}", id);
        return await repository.GetByIdAsync(id);
    }
}

// Parameters are captured, not fields
// If you need a field, assign it:
public class OrderService(IOrderRepository repository)
{
    private readonly IOrderRepository _repository = repository; // Now a field
}
```

### Collection Expressions

```csharp
// Old ways
var numbers1 = new int[] { 1, 2, 3 };
var numbers2 = new List<int> { 1, 2, 3 };
var numbers3 = new[] { 1, 2, 3 };

// C# 12 - Collection expressions
int[] numbers = [1, 2, 3];
List<int> list = [1, 2, 3];
Span<int> span = [1, 2, 3];
ImmutableArray<int> immutable = [1, 2, 3];

// Empty collection
int[] empty = [];

// Spread operator
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];
int[] combined = [..first, ..second]; // [1, 2, 3, 4, 5, 6]

// Works with any collection type
HashSet<int> set = [1, 2, 3];
Dictionary<string, int> dict = new() { ["one"] = 1 }; // Not yet supported
```

### Default Lambda Parameters

```csharp
// C# 12 - Default values in lambdas
var greet = (string name = "World") => $"Hello, {name}!";

Console.WriteLine(greet());        // "Hello, World!"
Console.WriteLine(greet("John"));  // "Hello, John!"

// With multiple parameters
var calculate = (int x, int y = 10, int z = 100) => x + y + z;

Console.WriteLine(calculate(1));       // 111
Console.WriteLine(calculate(1, 2));    // 103
Console.WriteLine(calculate(1, 2, 3)); // 6
```

### Alias Any Type

```csharp
// Old - only for namespaces and named types
using OrderList = System.Collections.Generic.List<MyApp.Order>;

// C# 12 - Alias any type including tuples, arrays, pointers
using Point = (int X, int Y);
using IntArray = int[];
using OrderDict = System.Collections.Generic.Dictionary<string, (int Id, string Name)>;

// Usage
Point origin = (0, 0);
IntArray numbers = [1, 2, 3];

// Useful for complex generics
using CustomerResult = System.Threading.Tasks.Task<
    MyApp.Result<System.Collections.Generic.List<MyApp.Customer>>>;

public async CustomerResult GetCustomersAsync() { ... }
```

### Inline Arrays

```csharp
// High-performance fixed-size arrays on the stack
[InlineArray(10)]
public struct TenIntegers
{
    private int _element0;
}

// Usage
TenIntegers buffer = default;
buffer[0] = 1;
buffer[9] = 10;

// Can iterate
foreach (var item in buffer)
{
    Console.WriteLine(item);
}

// Useful for performance-critical code
// Avoids heap allocation
```

### Interceptors (Experimental)

```csharp
// Allow source generators to intercept and replace method calls
// Useful for AOT scenarios

// Mark a method as interceptable
[InterceptsLocation("path/file.cs", line: 10, column: 5)]
public static void InterceptedMethod() { }
```

---

## C# 13 Features (with .NET 9)

### `params` Collections

```csharp
// Old - params only worked with arrays
public void Log(params string[] messages) { }

// C# 13 - params works with any collection type
public void Log(params IEnumerable<string> messages) { }
public void Log(params List<string> messages) { }
public void Log(params Span<string> messages) { }
public void Log(params ReadOnlySpan<string> messages) { }

// Usage
Log("one", "two", "three");
Log(["one", "two", "three"]); // Collection expression
```

### New Lock Type

```csharp
// Old way
private readonly object _lock = new();

lock (_lock)
{
    // Critical section
}

// C# 13 - New Lock type (more efficient)
private readonly Lock _lock = new();

lock (_lock)
{
    // Critical section
}

// Or with explicit scope
using (_lock.EnterScope())
{
    // Critical section
}
```

### Partial Properties

```csharp
// Partial properties in partial classes
public partial class Customer
{
    public partial string Name { get; set; }
}

public partial class Customer
{
    private string _name = "";

    public partial string Name
    {
        get => _name;
        set => _name = value ?? throw new ArgumentNullException(nameof(value));
    }
}
```

### `\e` Escape Sequence

```csharp
// New escape sequence for ESC character (0x1B)
// Useful for terminal colors/formatting

string red = "\e[31m";
string reset = "\e[0m";

Console.WriteLine($"{red}Error!{reset}");
```

### `ref struct` Interface Implementation

```csharp
// C# 13 allows ref structs to implement interfaces
public interface IAddable<T>
{
    T Add(T other);
}

public ref struct MyRefStruct : IAddable<MyRefStruct>
{
    public int Value;

    public MyRefStruct Add(MyRefStruct other)
    {
        return new MyRefStruct { Value = this.Value + other.Value };
    }
}
```

---

## ASP.NET Core Fundamentals

### Middleware Pipeline

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Middleware order matters!
app.UseExceptionHandler("/error");  // 1. Exception handling (first)
app.UseHsts();                       // 2. HSTS
app.UseHttpsRedirection();           // 3. HTTPS redirect
app.UseStaticFiles();                // 4. Static files
app.UseRouting();                    // 5. Routing
app.UseCors();                       // 6. CORS
app.UseAuthentication();             // 7. Authentication
app.UseAuthorization();              // 8. Authorization
app.UseResponseCaching();            // 9. Response caching
app.MapControllers();                // 10. Endpoints

app.Run();
```

### Dependency Injection Lifetimes

```csharp
// Transient - New instance every time
services.AddTransient<IEmailService, EmailService>();

// Scoped - One instance per HTTP request
services.AddScoped<IOrderService, OrderService>();

// Singleton - One instance for app lifetime
services.AddSingleton<ICacheService, CacheService>();

// When to use which:
// - Transient: Lightweight, stateless services
// - Scoped: Services with request-specific state (DbContext, Unit of Work)
// - Singleton: Heavy initialization, shared state (caches, configuration)
```

### Configuration

```csharp
// appsettings.json structure
{
    "ConnectionStrings": {
        "DefaultConnection": "Server=..."
    },
    "PaymentSettings": {
        "ApiKey": "xxx",
        "Timeout": 30
    }
}

// Strongly-typed configuration
public class PaymentSettings
{
    public string ApiKey { get; set; } = "";
    public int Timeout { get; set; } = 30;
}

// Registration
builder.Services.Configure<PaymentSettings>(
    builder.Configuration.GetSection("PaymentSettings"));

// Usage
public class PaymentService
{
    private readonly PaymentSettings _settings;

    public PaymentService(IOptions<PaymentSettings> options)
    {
        _settings = options.Value;
    }
}

// Environment-specific
// appsettings.Development.json
// appsettings.Production.json

// Environment variables (override)
// PaymentSettings__ApiKey=yyy
```

### Error Handling

```csharp
// Global exception handler
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/json";

        var error = context.Features.Get<IExceptionHandlerFeature>();
        if (error != null)
        {
            var response = new
            {
                Message = "An error occurred",
                TraceId = context.TraceIdentifier
            };

            await context.Response.WriteAsJsonAsync(response);
        }
    });
});

// Problem Details (RFC 7807)
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Extensions["traceId"] = context.HttpContext.TraceIdentifier;
    };
});
```

---

## Entity Framework Core Basics

### DbContext Setup

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}

// Entity configuration
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Total).HasPrecision(18, 2);
        builder.HasOne(o => o.Customer)
               .WithMany(c => c.Orders)
               .HasForeignKey(o => o.CustomerId);
    }
}

// Registration
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

### Common Operations

```csharp
// Query
var orders = await _context.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .Include(o => o.Items)
    .OrderByDescending(o => o.CreatedAt)
    .Take(10)
    .ToListAsync();

// Add
var order = new Order { /* ... */ };
_context.Orders.Add(order);
await _context.SaveChangesAsync();

// Update
var order = await _context.Orders.FindAsync(id);
order.Status = OrderStatus.Completed;
await _context.SaveChangesAsync();

// Delete
var order = await _context.Orders.FindAsync(id);
_context.Orders.Remove(order);
await _context.SaveChangesAsync();

// Raw SQL (when needed)
var orders = await _context.Orders
    .FromSqlRaw("SELECT * FROM Orders WHERE Total > {0}", 100)
    .ToListAsync();
```

---

## Quick Reference

### .NET 8 Must-Know

| Feature | Use Case |
|---------|----------|
| Frozen Collections | Read-heavy dictionaries/sets |
| Native AOT | Containers, serverless |
| TimeProvider | Testable time-dependent code |
| Keyed Services | Multiple implementations of same interface |
| Minimal APIs Groups | Organized endpoint routing |

### C# 12 Must-Know

| Feature | Example |
|---------|---------|
| Primary Constructors | `class Service(IRepo repo)` |
| Collection Expressions | `int[] arr = [1, 2, 3]` |
| Spread Operator | `[..first, ..second]` |
| Type Aliases | `using Point = (int X, int Y)` |

### Interview Tips

**Q: What's new in .NET 8?**
> "Key features include Native AOT for faster startup, Frozen Collections for read-heavy scenarios, keyed DI services, and the new TimeProvider for testable code. It's an LTS release, making it the recommended choice for production."

**Q: What's the difference between .NET 8 and .NET 9?**
> ".NET 9 builds on 8 with features like HybridCache, improved LINQ methods (CountBy, AggregateBy), and better OpenAPI support. However, .NET 8 is LTS with 3-year support, while .NET 9 is STS with 18-month support. For production stability, I'd recommend .NET 8."

**Q: When would you use Native AOT?**
> "Native AOT is ideal for containerized workloads, serverless functions, and CLI tools where startup time matters. It eliminates JIT compilation for faster cold starts. However, it has limitations with reflection and dynamic code, so it's not suitable for all applications."
