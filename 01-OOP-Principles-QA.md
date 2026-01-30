# OOP Principles - Interview Questions

> 5 real-world interview questions with detailed answers and communication tactics

---

## Question 1: Explain the 4 OOP Pillars with a Payment System Example

### The Question
> "Can you explain the four pillars of Object-Oriented Programming? Please use a real-world example from a payment or e-commerce system to illustrate each concept."

### Key Points to Cover
- Define each pillar clearly and concisely
- Use a cohesive example (payment system) throughout
- Show how they work together
- Mention C# specific implementations

### Detailed Answer

**1. Encapsulation** - Bundling data and methods that operate on that data, hiding internal state.

```csharp
public class PaymentTransaction
{
    // Private fields - hidden from outside
    private decimal _amount;
    private string _status;
    private readonly DateTime _createdAt;

    // Public property with validation - controlled access
    public decimal Amount
    {
        get => _amount;
        set
        {
            if (value <= 0)
                throw new ArgumentException("Amount must be positive");
            _amount = value;
        }
    }

    // Read-only property - protects internal state
    public string Status => _status;

    // Method that controls state transitions
    public void Complete()
    {
        if (_status != "Pending")
            throw new InvalidOperationException("Can only complete pending transactions");
        _status = "Completed";
    }
}
```

**2. Inheritance** - Creating new classes based on existing ones (is-a relationship).

```csharp
public abstract class PaymentMethod
{
    public string Id { get; protected set; }
    public abstract Task<PaymentResult> ProcessAsync(decimal amount);
}

public class CreditCardPayment : PaymentMethod
{
    public string CardNumber { get; set; }
    public string ExpiryDate { get; set; }

    public override async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        // Credit card specific processing
        return await _gateway.ChargeCreditCardAsync(CardNumber, amount);
    }
}

public class WalletPayment : PaymentMethod
{
    public string WalletId { get; set; }

    public override async Task<PaymentResult> ProcessAsync(decimal amount)
    {
        // Wallet specific processing
        return await _walletService.DeductAsync(WalletId, amount);
    }
}
```

**3. Abstraction** - Hiding complexity and showing only necessary details.

```csharp
// Interface defines WHAT, not HOW
public interface IPaymentGateway
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request);
    Task<RefundResult> RefundAsync(string transactionId);
}

// Implementation hides the complexity
public class StripePaymentGateway : IPaymentGateway
{
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // All Stripe-specific complexity hidden here:
        // - API authentication
        // - Request formatting
        // - Error handling
        // - Retry logic
        // Caller just sees: gateway.ProcessPaymentAsync(request)
    }
}
```

**4. Polymorphism** - Objects of different types responding to the same interface differently.

```csharp
public class PaymentProcessor
{
    public async Task<PaymentResult> ProcessAsync(PaymentMethod paymentMethod, decimal amount)
    {
        // Same method call, different behavior based on actual type
        return await paymentMethod.ProcessAsync(amount);
    }
}

// Usage - polymorphism in action
var processor = new PaymentProcessor();

PaymentMethod creditCard = new CreditCardPayment { CardNumber = "4111..." };
PaymentMethod wallet = new WalletPayment { WalletId = "W123" };

// Same interface, different implementations
await processor.ProcessAsync(creditCard, 100);  // Uses credit card logic
await processor.ProcessAsync(wallet, 100);      // Uses wallet logic
```

### Communication Tactics

🎯 **Structure your answer**: Start with "The four pillars are..." then go through each one systematically. Use the same domain example throughout to show cohesion.

💡 **Emphasize**: How these pillars work TOGETHER. Encapsulation protects data, inheritance enables code reuse, abstraction hides complexity, polymorphism enables flexibility.

⚠️ **Avoid**: Don't just give textbook definitions. Show you understand WHY these matter in real code - maintainability, testability, flexibility.

---

## Question 2: Abstract Class vs Interface - When to Use Each?

### The Question
> "What's the difference between an abstract class and an interface in C#? When would you choose one over the other? Can you give me a real example?"

### Key Points to Cover
- Technical differences (inheritance, implementation, fields)
- C# 8+ interface default implementations
- Design intent differences
- Practical decision criteria

### Detailed Answer

**Technical Differences:**

| Aspect | Abstract Class | Interface |
|--------|---------------|-----------|
| Inheritance | Single (one base class) | Multiple (many interfaces) |
| Implementation | Can have implemented methods | Default methods since C# 8 |
| Fields | Can have fields and state | No instance fields |
| Constructors | Yes | No |
| Access modifiers | Any | Public by default |

**When to Use Abstract Class:**

```csharp
// Use when you have SHARED STATE or SHARED IMPLEMENTATION
public abstract class PaymentProcessor
{
    // Shared state
    protected readonly ILogger _logger;
    protected readonly IMetrics _metrics;

    // Shared constructor logic
    protected PaymentProcessor(ILogger logger, IMetrics metrics)
    {
        _logger = logger;
        _metrics = metrics;
    }

    // Shared implementation (template method pattern)
    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        _logger.LogInformation("Processing payment {Id}", request.Id);
        var stopwatch = Stopwatch.StartNew();

        try
        {
            // Abstract - each subclass implements differently
            var result = await ProcessCoreAsync(request);

            _metrics.RecordPaymentTime(stopwatch.Elapsed);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Payment failed");
            throw;
        }
    }

    // Force subclasses to implement
    protected abstract Task<PaymentResult> ProcessCoreAsync(PaymentRequest request);
}

public class StripeProcessor : PaymentProcessor
{
    public StripeProcessor(ILogger logger, IMetrics metrics) : base(logger, metrics) { }

    protected override async Task<PaymentResult> ProcessCoreAsync(PaymentRequest request)
    {
        // Stripe-specific implementation
    }
}
```

**When to Use Interface:**

```csharp
// Use when you define a CONTRACT or CAPABILITY
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(decimal amount, string token);
    Task<RefundResult> RefundAsync(string transactionId);
}

public interface IAuditable
{
    DateTime CreatedAt { get; }
    string CreatedBy { get; }
}

public interface INotifiable
{
    Task NotifyAsync(string message);
}

// A class can implement multiple interfaces (multiple capabilities)
public class Order : IAuditable, INotifiable
{
    public DateTime CreatedAt { get; private set; }
    public string CreatedBy { get; private set; }

    public async Task NotifyAsync(string message)
    {
        // Send notification
    }
}
```

**Decision Criteria:**

```csharp
// INTERFACE when:
// 1. You need multiple inheritance of behavior
// 2. You're defining a contract/capability
// 3. Unrelated classes need same behavior
// 4. You want to support dependency injection easily

// ABSTRACT CLASS when:
// 1. You have shared code to reuse
// 2. You need shared state (fields)
// 3. Classes are closely related (is-a relationship)
// 4. You want to use template method pattern
```

### Communication Tactics

🎯 **Structure your answer**: Start with technical differences (table format helps), then move to "when to use which" with concrete examples.

💡 **Emphasize**: The DESIGN INTENT - interfaces are about contracts/capabilities, abstract classes are about shared behavior in related classes. Mention C# 8 default interface methods but note they're for API evolution, not replacing abstract classes.

⚠️ **Avoid**: Don't memorize without understanding. If asked a follow-up like "Can you have fields in interface?", you should know WHY not (interfaces don't have instance state).

---

## Question 3: Compile-time vs Runtime Polymorphism

### The Question
> "Can you explain the difference between compile-time and runtime polymorphism in C#? Give me examples of each and explain when you'd use them."

### Key Points to Cover
- Method overloading (compile-time)
- Method overriding (runtime)
- How the compiler/runtime resolves calls
- Practical use cases

### Detailed Answer

**Compile-time Polymorphism (Static/Early Binding):**

The compiler decides which method to call based on the method signature at compile time.

```csharp
public class PriceCalculator
{
    // Method overloading - same name, different parameters
    public decimal Calculate(decimal basePrice)
    {
        return basePrice;
    }

    public decimal Calculate(decimal basePrice, decimal discountPercent)
    {
        return basePrice * (1 - discountPercent / 100);
    }

    public decimal Calculate(decimal basePrice, decimal discountPercent, decimal taxRate)
    {
        var discounted = basePrice * (1 - discountPercent / 100);
        return discounted * (1 + taxRate / 100);
    }

    // Operator overloading is also compile-time polymorphism
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(a.Amount + b.Amount, a.Currency);
    }
}

// Usage - compiler knows which method to call
var calc = new PriceCalculator();
var price1 = calc.Calculate(100);           // Calls first overload
var price2 = calc.Calculate(100, 10);       // Calls second overload
var price3 = calc.Calculate(100, 10, 18);   // Calls third overload
```

**Runtime Polymorphism (Dynamic/Late Binding):**

The runtime decides which method to call based on the actual object type.

```csharp
public abstract class Notification
{
    public abstract Task SendAsync(string message, string recipient);
}

public class EmailNotification : Notification
{
    public override async Task SendAsync(string message, string recipient)
    {
        Console.WriteLine($"Sending EMAIL to {recipient}: {message}");
        await _emailService.SendAsync(recipient, message);
    }
}

public class SmsNotification : Notification
{
    public override async Task SendAsync(string message, string recipient)
    {
        Console.WriteLine($"Sending SMS to {recipient}: {message}");
        await _smsGateway.SendAsync(recipient, message);
    }
}

public class PushNotification : Notification
{
    public override async Task SendAsync(string message, string recipient)
    {
        Console.WriteLine($"Sending PUSH to {recipient}: {message}");
        await _pushService.SendAsync(recipient, message);
    }
}

// Runtime polymorphism in action
public class NotificationService
{
    public async Task NotifyAsync(Notification notification, string message, string recipient)
    {
        // At compile time: doesn't know which SendAsync to call
        // At runtime: calls the actual type's implementation
        await notification.SendAsync(message, recipient);
    }
}

// Usage
var service = new NotificationService();
Notification notification = GetNotificationPreference(userId); // Returns Email, SMS, or Push

// Same code, different behavior based on actual type
await service.NotifyAsync(notification, "Your order shipped!", "user@email.com");
```

**Key Differences:**

| Aspect | Compile-time | Runtime |
|--------|-------------|---------|
| Binding | Early (compile) | Late (runtime) |
| Mechanism | Overloading | Overriding |
| Keywords | None special | virtual/override/abstract |
| Performance | Slightly faster | Virtual call overhead |
| Flexibility | Less flexible | More flexible |

### Communication Tactics

🎯 **Structure your answer**: Define both terms clearly, then show concrete examples. Use a comparison table to make differences clear.

💡 **Emphasize**: Runtime polymorphism is key for extensibility and following Open/Closed Principle. Mention practical use cases like plugin systems, strategy pattern, notification systems.

⚠️ **Avoid**: Don't confuse overloading with overriding. Be ready for follow-up: "What happens if you forget the `override` keyword?" (It hides the base method instead of overriding).

---

## Question 4: Composition Over Inheritance

### The Question
> "You've probably heard 'favor composition over inheritance.' What does this mean and when would you choose composition? Can you show me an example where inheritance would be a bad choice?"

### Key Points to Cover
- What composition means
- Problems with deep inheritance hierarchies
- When inheritance IS appropriate
- Practical refactoring example

### Detailed Answer

**The Problem with Inheritance:**

```csharp
// BAD - Deep inheritance hierarchy
public class Animal { }
public class Bird : Animal { public virtual void Fly() { } }
public class Penguin : Bird
{
    public override void Fly()
    {
        throw new NotSupportedException("Penguins can't fly!"); // LSP violation!
    }
}

// BAD - "God class" through inheritance
public class Report { }
public class FormattedReport : Report { }
public class PrintableFormattedReport : FormattedReport { }
public class EmailablePrintableFormattedReport : PrintableFormattedReport { }
// This gets ridiculous quickly!
```

**Composition Solution:**

```csharp
// GOOD - Compose behaviors
public interface IFlyable
{
    void Fly();
}

public interface ISwimmable
{
    void Swim();
}

public class Bird
{
    private readonly IFlyable _flyBehavior;

    public Bird(IFlyable flyBehavior)
    {
        _flyBehavior = flyBehavior;
    }

    public void Fly() => _flyBehavior?.Fly();
}

public class StandardFlight : IFlyable
{
    public void Fly() => Console.WriteLine("Flying through the air!");
}

public class NoFlight : IFlyable
{
    public void Fly() => Console.WriteLine("I can't fly, but I can waddle!");
}

// Usage
var eagle = new Bird(new StandardFlight());
var penguin = new Bird(new NoFlight());
```

**Real-World Example - Report Generation:**

```csharp
// GOOD - Composition with Strategy pattern
public interface IReportFormatter
{
    string Format(ReportData data);
}

public interface IReportExporter
{
    Task ExportAsync(string content, string destination);
}

public class ReportGenerator
{
    private readonly IReportFormatter _formatter;
    private readonly IReportExporter _exporter;

    public ReportGenerator(IReportFormatter formatter, IReportExporter exporter)
    {
        _formatter = formatter;
        _exporter = exporter;
    }

    public async Task GenerateAsync(ReportData data, string destination)
    {
        var content = _formatter.Format(data);
        await _exporter.ExportAsync(content, destination);
    }
}

// Now we can mix and match!
var pdfEmailReport = new ReportGenerator(new PdfFormatter(), new EmailExporter());
var htmlFileReport = new ReportGenerator(new HtmlFormatter(), new FileExporter());
var jsonApiReport = new ReportGenerator(new JsonFormatter(), new ApiExporter());
```

**When Inheritance IS Appropriate:**

```csharp
// Inheritance is fine when there's a true IS-A relationship
// and the hierarchy is shallow
public abstract class Entity
{
    public Guid Id { get; protected set; }
    public DateTime CreatedAt { get; protected set; }
}

public class Order : Entity  // Order IS-A Entity - makes sense
{
    public List<OrderItem> Items { get; }
}

public class Customer : Entity  // Customer IS-A Entity - makes sense
{
    public string Email { get; }
}
```

**Decision Guide:**

```
Use INHERITANCE when:
- True "is-a" relationship (Order is-a Entity)
- Hierarchy is shallow (2-3 levels max)
- Base class has significant shared implementation
- Relationship is unlikely to change

Use COMPOSITION when:
- "Has-a" or "uses-a" relationship
- Need multiple "behaviors" (would need multiple inheritance)
- Behaviors might be swapped at runtime
- Following Strategy, Decorator, or similar patterns
```

### Communication Tactics

🎯 **Structure your answer**: Start with the PROBLEM inheritance creates, then show the composition solution. End with when inheritance IS okay.

💡 **Emphasize**: This principle enables flexibility, testability (easy to mock composed dependencies), and follows Open/Closed Principle. Mention it's a "favor, not forbid" - inheritance has its place.

⚠️ **Avoid**: Don't say "inheritance is bad." Be nuanced - show you understand when each is appropriate.

---

## Question 5: Encapsulation and Immutability - Value Objects

### The Question
> "How does encapsulation relate to immutability? Can you show me how you'd implement a Value Object in C# that demonstrates both concepts?"

### Key Points to Cover
- How encapsulation protects state
- Why immutability matters
- C# record types
- Value Object pattern from DDD

### Detailed Answer

**Encapsulation + Immutability Connection:**

Encapsulation controls access to state. Immutability is the ultimate form of protection - once created, state cannot change. Together, they create objects that are:
- Thread-safe (no race conditions)
- Easy to reason about (no surprise mutations)
- Safe to share (no defensive copying needed)

**Value Object Implementation:**

```csharp
// Value Object - Immutable, compared by value, self-validating
public sealed class Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    // Private constructor - controlled creation
    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    // Factory method with validation - encapsulates creation rules
    public static Money Create(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative", nameof(amount));

        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            throw new ArgumentException("Currency must be 3-letter code", nameof(currency));

        return new Money(amount, currency.ToUpperInvariant());
    }

    // Operations return NEW objects - immutability
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}");

        return new Money(Amount + other.Amount, Currency);
    }

    public Money MultiplyBy(decimal factor)
    {
        return new Money(Amount * factor, Currency);
    }

    // Value equality - compared by value, not reference
    public bool Equals(Money? other)
    {
        if (other is null) return false;
        return Amount == other.Amount && Currency == other.Currency;
    }

    public override bool Equals(object? obj) => Equals(obj as Money);

    public override int GetHashCode() => HashCode.Combine(Amount, Currency);

    public static bool operator ==(Money? left, Money? right) => Equals(left, right);
    public static bool operator !=(Money? left, Money? right) => !Equals(left, right);

    public override string ToString() => $"{Amount:N2} {Currency}";
}

// Usage
var price = Money.Create(100, "TRY");
var tax = price.MultiplyBy(0.18m);      // Returns new Money
var total = price.Add(tax);              // Returns new Money
// price is still 100 TRY - unchanged!

Console.WriteLine(price);  // 100.00 TRY
Console.WriteLine(total);  // 118.00 TRY
```

**C# Record Types (Simpler Approach):**

```csharp
// C# 9+ records are immutable by default with value semantics
public record Money(decimal Amount, string Currency)
{
    // Validation in constructor
    public Money
    {
        if (Amount < 0)
            throw new ArgumentException("Amount cannot be negative");
        if (string.IsNullOrWhiteSpace(Currency) || Currency.Length != 3)
            throw new ArgumentException("Currency must be 3-letter code");

        Currency = Currency.ToUpperInvariant();
    }

    // Operations still return new instances
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add different currencies");

        return this with { Amount = Amount + other.Amount };
    }
}

// Another common Value Object
public record Address(
    string Street,
    string City,
    string Country,
    string PostalCode)
{
    // Computed property
    public string FullAddress => $"{Street}, {City}, {Country} {PostalCode}";
}

// Records give you:
// - Immutability by default
// - Value equality
// - ToString()
// - with expressions for copies
```

**Why This Matters in Practice:**

```csharp
// Without Value Object - primitive obsession
public class Order
{
    public decimal Amount { get; set; }     // Can be negative?
    public string Currency { get; set; }    // Can be null? Invalid?

    // Validation scattered everywhere
    public void Process()
    {
        if (Amount <= 0) throw new Exception();
        if (Currency == null) throw new Exception();
        // ...
    }
}

// With Value Object - self-validating, impossible to create invalid state
public class Order
{
    public Money Total { get; }  // Always valid, always immutable

    public Order(Money total)
    {
        Total = total ?? throw new ArgumentNullException(nameof(total));
    }
}
```

### Communication Tactics

🎯 **Structure your answer**: Connect encapsulation and immutability conceptually first, then show the Value Object pattern. Finish with C# records as modern approach.

💡 **Emphasize**: Benefits - thread safety, no defensive copying, impossible invalid state, self-documenting code. Mention DDD context where Value Objects replace primitive obsession.

⚠️ **Avoid**: Don't just show code - explain WHY immutability matters. Be ready for follow-up: "When would you NOT use immutability?" (Performance-critical hot paths, large data structures with frequent updates).

---

## Quick Review - Key Takeaways

| Question | Key Point to Remember |
|----------|----------------------|
| 4 Pillars | Use same example throughout, show how they work together |
| Abstract vs Interface | Interfaces = contracts, Abstract = shared implementation |
| Polymorphism | Compile-time = overloading, Runtime = overriding |
| Composition | Favor for flexibility, inheritance for true is-a |
| Value Objects | Immutable, self-validating, use records in C# 9+ |
