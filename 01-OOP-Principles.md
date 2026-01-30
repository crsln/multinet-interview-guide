# OOP Principles - The 4 Pillars

> ⚠️ **HIGH PRIORITY**: Insider info confirms "they will definitely ask about this"

## Overview

Object-Oriented Programming (OOP) is built on four fundamental principles. Understanding these deeply—with practical C# examples—is essential for any senior .NET developer interview.

---

## 1. Encapsulation (Kapsülleme)

### Definition
Encapsulation is the bundling of data (fields) and methods that operate on that data within a single unit (class), while restricting direct access to some of the object's components.

### Key Concept
**Hide internal state and require all interaction to be performed through well-defined methods.**

### C# Implementation

```csharp
// ❌ BAD: No encapsulation - fields exposed directly
public class BankAccount
{
    public decimal Balance;  // Anyone can set this to any value!
    public string AccountNumber;
}

// ✅ GOOD: Proper encapsulation
public class BankAccount
{
    // Private fields - internal state is hidden
    private decimal _balance;
    private readonly string _accountNumber;

    // Constructor validates initial state
    public BankAccount(string accountNumber, decimal initialBalance)
    {
        if (string.IsNullOrEmpty(accountNumber))
            throw new ArgumentException("Account number required");
        if (initialBalance < 0)
            throw new ArgumentException("Initial balance cannot be negative");

        _accountNumber = accountNumber;
        _balance = initialBalance;
    }

    // Read-only property - external code can read but not modify directly
    public decimal Balance => _balance;
    public string AccountNumber => _accountNumber;

    // Controlled access through methods with business logic
    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Deposit amount must be positive");
        _balance += amount;
    }

    public bool Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Withdrawal amount must be positive");
        if (amount > _balance)
            return false; // Insufficient funds

        _balance -= amount;
        return true;
    }
}
```

### Access Modifiers in C#

| Modifier | Access Level |
|----------|-------------|
| `public` | Accessible from anywhere |
| `private` | Only within the same class |
| `protected` | Same class + derived classes |
| `internal` | Same assembly only |
| `protected internal` | Same assembly OR derived classes |
| `private protected` | Same class + derived classes in same assembly |

### Real-World Use Case (Fintech Context)
In a payment system, transaction amounts and account balances should never be directly modifiable. All changes must go through validated methods that enforce business rules (e.g., no negative balances, fraud checks).

### Interview Q&A

**Q: What is encapsulation and why is it important?**
> A: Encapsulation bundles data with the methods that operate on it while hiding internal implementation details. It's important because it:
> 1. Protects data integrity by controlling how state is modified
> 2. Reduces coupling—external code doesn't depend on internal structure
> 3. Makes code more maintainable—internal changes don't break external code
> 4. Enables validation and business rules at the point of modification

**Q: What's the difference between a field and a property?**
> A: A field is a variable declared directly in a class. A property is a member that provides a flexible mechanism to read, write, or compute a value. Properties can include logic (validation, computation) while fields cannot.

---

## 2. Inheritance (Kalıtım)

### Definition
Inheritance is a mechanism where a new class (derived/child) inherits properties and behaviors from an existing class (base/parent), enabling code reuse and establishing an "is-a" relationship.

### Key Concept
**Derive new classes from existing ones to reuse code and create hierarchies.**

### C# Implementation

```csharp
// Base class
public abstract class PaymentMethod
{
    public string MethodId { get; protected set; }
    public DateTime CreatedAt { get; } = DateTime.UtcNow;

    // Virtual method - can be overridden
    public virtual bool Validate()
    {
        return !string.IsNullOrEmpty(MethodId);
    }

    // Abstract method - MUST be implemented by derived classes
    public abstract Task<PaymentResult> ProcessPaymentAsync(decimal amount);
}

// Derived class - Credit Card
public class CreditCardPayment : PaymentMethod
{
    public string CardNumber { get; private set; }
    public string ExpiryDate { get; private set; }
    public string CVV { get; private set; }

    public CreditCardPayment(string cardNumber, string expiry, string cvv)
    {
        MethodId = Guid.NewGuid().ToString();
        CardNumber = cardNumber;
        ExpiryDate = expiry;
        CVV = cvv;
    }

    // Override base validation and extend it
    public override bool Validate()
    {
        if (!base.Validate()) return false;

        // Credit card specific validation
        return CardNumber?.Length == 16
            && !string.IsNullOrEmpty(ExpiryDate)
            && CVV?.Length == 3;
    }

    // Implement abstract method
    public override async Task<PaymentResult> ProcessPaymentAsync(decimal amount)
    {
        // Credit card processing logic
        await Task.Delay(100); // Simulate API call
        return new PaymentResult { Success = true, TransactionId = Guid.NewGuid().ToString() };
    }
}

// Derived class - Digital Wallet (like MultiPay!)
public class DigitalWalletPayment : PaymentMethod
{
    public string WalletId { get; private set; }
    public string PhoneNumber { get; private set; }

    public DigitalWalletPayment(string walletId, string phoneNumber)
    {
        MethodId = Guid.NewGuid().ToString();
        WalletId = walletId;
        PhoneNumber = phoneNumber;
    }

    public override async Task<PaymentResult> ProcessPaymentAsync(decimal amount)
    {
        // Wallet-specific processing
        await Task.Delay(50); // Typically faster than card
        return new PaymentResult { Success = true, TransactionId = Guid.NewGuid().ToString() };
    }
}
```

### When to Use Inheritance vs Composition

| Inheritance | Composition |
|-------------|-------------|
| "Is-a" relationship | "Has-a" relationship |
| `CreditCardPayment` **is a** `PaymentMethod` | `Order` **has a** `PaymentMethod` |
| When you need to override behavior | When you need flexibility |
| When classes share significant code | When combining behaviors from multiple sources |

```csharp
// Composition example - Order HAS a PaymentMethod
public class Order
{
    public string OrderId { get; set; }
    public PaymentMethod Payment { get; set; }  // Composition
    public List<OrderItem> Items { get; set; }  // Composition

    public async Task<bool> CompleteOrderAsync()
    {
        if (!Payment.Validate())
            return false;

        var result = await Payment.ProcessPaymentAsync(CalculateTotal());
        return result.Success;
    }
}
```

### Interview Q&A

**Q: What is inheritance and when would you use it?**
> A: Inheritance allows a class to inherit properties and methods from a parent class, establishing an "is-a" relationship. I'd use it when:
> 1. There's a clear hierarchical relationship between classes
> 2. Multiple classes share common functionality
> 3. I want to leverage polymorphism for extensibility
> However, I favor composition over inheritance when the relationship isn't naturally hierarchical or when I need more flexibility.

**Q: What's the difference between `abstract` and `virtual`?**
> A: A `virtual` method has a default implementation that CAN be overridden. An `abstract` method has NO implementation and MUST be overridden in derived classes. Abstract methods can only exist in abstract classes.

---

## 3. Abstraction (Soyutlama)

### Definition
Abstraction is the concept of hiding complex implementation details and showing only the necessary features of an object. It focuses on WHAT an object does rather than HOW it does it.

### Key Concept
**Define interfaces and abstract classes that specify behavior without implementation details.**

### Abstract Classes vs Interfaces

| Feature | Abstract Class | Interface |
|---------|---------------|-----------|
| Implementation | Can have implemented methods | No implementation (before C# 8) |
| Fields | Can have fields | Cannot have fields |
| Constructors | Can have constructors | Cannot have constructors |
| Multiple inheritance | Single inheritance only | Multiple interfaces allowed |
| Access modifiers | All access modifiers | Public by default |
| Use case | Shared base functionality | Contract/capability definition |

### C# Implementation

```csharp
// INTERFACE - Defines a contract (what capabilities exist)
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(PaymentRequest request);
    Task<RefundResult> RefundAsync(string transactionId, decimal amount);
}

public interface IAuditable
{
    DateTime CreatedAt { get; }
    DateTime? ModifiedAt { get; }
    string CreatedBy { get; }
}

// ABSTRACT CLASS - Shared base with some implementation
public abstract class BasePaymentProcessor : IPaymentProcessor, IAuditable
{
    // IAuditable implementation
    public DateTime CreatedAt { get; } = DateTime.UtcNow;
    public DateTime? ModifiedAt { get; protected set; }
    public string CreatedBy { get; protected set; }

    // Common functionality implemented once
    protected async Task LogTransactionAsync(string transactionId, string status)
    {
        // Shared logging logic
        await Task.CompletedTask;
    }

    protected bool ValidateRequest(PaymentRequest request)
    {
        return request != null
            && request.Amount > 0
            && !string.IsNullOrEmpty(request.Currency);
    }

    // Abstract - each processor must implement
    public abstract Task<PaymentResult> ProcessAsync(PaymentRequest request);
    public abstract Task<RefundResult> RefundAsync(string transactionId, decimal amount);
}

// CONCRETE IMPLEMENTATION - Stripe
public class StripePaymentProcessor : BasePaymentProcessor
{
    private readonly string _apiKey;

    public StripePaymentProcessor(string apiKey)
    {
        _apiKey = apiKey;
        CreatedBy = "System";
    }

    public override async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        if (!ValidateRequest(request))
            return new PaymentResult { Success = false, Error = "Invalid request" };

        // Stripe-specific implementation hidden from consumers
        var result = await CallStripeApiAsync(request);
        await LogTransactionAsync(result.TransactionId, "Processed");

        return result;
    }

    public override async Task<RefundResult> RefundAsync(string transactionId, decimal amount)
    {
        // Stripe refund logic
        await Task.Delay(100);
        return new RefundResult { Success = true };
    }

    private async Task<PaymentResult> CallStripeApiAsync(PaymentRequest request)
    {
        // Complex Stripe API interaction hidden behind abstraction
        await Task.Delay(200);
        return new PaymentResult { Success = true, TransactionId = Guid.NewGuid().ToString() };
    }
}

// CONSUMER CODE - Works with abstraction, not implementation
public class CheckoutService
{
    private readonly IPaymentProcessor _processor;

    // Depends on abstraction, not concrete implementation
    public CheckoutService(IPaymentProcessor processor)
    {
        _processor = processor;
    }

    public async Task<CheckoutResult> ProcessCheckoutAsync(Cart cart)
    {
        var request = new PaymentRequest
        {
            Amount = cart.Total,
            Currency = "TRY"
        };

        // Doesn't know or care if it's Stripe, PayPal, or ininal
        var result = await _processor.ProcessAsync(request);

        return new CheckoutResult { Success = result.Success };
    }
}
```

### C# 8+ Interface Default Implementations

```csharp
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(PaymentRequest request);

    // Default implementation - new in C# 8
    Task<bool> ValidateAsync(PaymentRequest request)
    {
        return Task.FromResult(request != null && request.Amount > 0);
    }
}
```

### Interview Q&A

**Q: What is abstraction and how does it differ from encapsulation?**
> A: Abstraction hides complexity by exposing only what's necessary—focusing on WHAT something does. Encapsulation hides internal data by bundling it with methods—focusing on HOW data is protected.
>
> Example: An `IPaymentProcessor` interface is abstraction (hides HOW payment is processed). Private fields with public methods is encapsulation (hides internal state).

**Q: When would you use an abstract class vs an interface?**
> A: I use an **abstract class** when:
> - Classes share common implementation code
> - I need to define fields or constructors
> - There's a clear "is-a" relationship
>
> I use an **interface** when:
> - I need multiple inheritance
> - I'm defining a capability/contract
> - Unrelated classes need to share behavior
> - I want to enable easy mocking in tests

---

## 4. Polymorphism (Çok Biçimlilik)

### Definition
Polymorphism allows objects of different types to be treated as objects of a common base type. The same method call can behave differently depending on the actual object type.

### Key Concept
**"Many forms" - One interface, multiple implementations.**

### Types of Polymorphism

1. **Compile-time (Static)**: Method overloading, operator overloading
2. **Runtime (Dynamic)**: Method overriding, interface implementation

### C# Implementation

```csharp
// ===== COMPILE-TIME POLYMORPHISM (Overloading) =====

public class PaymentCalculator
{
    // Same method name, different parameters
    public decimal CalculateFee(decimal amount)
    {
        return amount * 0.02m; // 2% default fee
    }

    public decimal CalculateFee(decimal amount, string cardType)
    {
        return cardType switch
        {
            "credit" => amount * 0.025m,
            "debit" => amount * 0.015m,
            _ => amount * 0.02m
        };
    }

    public decimal CalculateFee(decimal amount, string cardType, bool isInternational)
    {
        var baseFee = CalculateFee(amount, cardType);
        return isInternational ? baseFee + (amount * 0.01m) : baseFee;
    }
}

// Usage - Compiler determines which method at compile time
var calc = new PaymentCalculator();
var fee1 = calc.CalculateFee(100m);                        // First overload
var fee2 = calc.CalculateFee(100m, "credit");              // Second overload
var fee3 = calc.CalculateFee(100m, "credit", true);        // Third overload


// ===== RUNTIME POLYMORPHISM (Overriding) =====

public abstract class Notification
{
    public string Recipient { get; set; }
    public string Message { get; set; }

    // Virtual method with default implementation
    public virtual async Task SendAsync()
    {
        Console.WriteLine($"Sending notification to {Recipient}");
        await Task.CompletedTask;
    }

    // Abstract method - no default
    public abstract string GetDeliveryChannel();
}

public class EmailNotification : Notification
{
    public string Subject { get; set; }

    public override async Task SendAsync()
    {
        // Email-specific sending logic
        Console.WriteLine($"Sending email to {Recipient}: {Subject}");
        await Task.Delay(100);
    }

    public override string GetDeliveryChannel() => "Email";
}

public class SmsNotification : Notification
{
    public override async Task SendAsync()
    {
        // SMS-specific sending logic
        Console.WriteLine($"Sending SMS to {Recipient}: {Message}");
        await Task.Delay(50);
    }

    public override string GetDeliveryChannel() => "SMS";
}

public class PushNotification : Notification
{
    public string DeviceToken { get; set; }

    public override async Task SendAsync()
    {
        Console.WriteLine($"Sending push to device {DeviceToken}");
        await Task.Delay(30);
    }

    public override string GetDeliveryChannel() => "Push";
}

// ===== POLYMORPHISM IN ACTION =====

public class NotificationService
{
    public async Task SendAllNotificationsAsync(IEnumerable<Notification> notifications)
    {
        foreach (var notification in notifications)
        {
            // Runtime polymorphism - actual method called depends on object type
            Console.WriteLine($"Channel: {notification.GetDeliveryChannel()}");
            await notification.SendAsync();
        }
    }
}

// Usage
var notifications = new List<Notification>
{
    new EmailNotification { Recipient = "user@email.com", Subject = "Welcome" },
    new SmsNotification { Recipient = "+90555...", Message = "Your OTP is 1234" },
    new PushNotification { Recipient = "user123", DeviceToken = "abc123" }
};

var service = new NotificationService();
await service.SendAllNotificationsAsync(notifications);
// Each notification type uses its own SendAsync implementation!
```

### Interface Polymorphism

```csharp
public interface IExportable
{
    byte[] Export();
    string GetFileExtension();
}

public class PdfExporter : IExportable
{
    public byte[] Export() { /* PDF logic */ return new byte[0]; }
    public string GetFileExtension() => ".pdf";
}

public class ExcelExporter : IExportable
{
    public byte[] Export() { /* Excel logic */ return new byte[0]; }
    public string GetFileExtension() => ".xlsx";
}

public class CsvExporter : IExportable
{
    public byte[] Export() { /* CSV logic */ return new byte[0]; }
    public string GetFileExtension() => ".csv";
}

// Consumer code works with any IExportable
public class ReportService
{
    public void GenerateReport(IExportable exporter)
    {
        var data = exporter.Export();
        var filename = $"report{exporter.GetFileExtension()}";
        File.WriteAllBytes(filename, data);
    }
}
```

### Interview Q&A

**Q: Explain polymorphism with an example.**
> A: Polymorphism means "many forms"—the same method call behaves differently based on the object type.
>
> For example, if I have a `Notification` base class with `SendAsync()`, I can have `EmailNotification`, `SmsNotification`, and `PushNotification` that each override it. When I loop through a `List<Notification>` and call `SendAsync()`, each notification uses its own implementation. This lets me add new notification types without changing the calling code.

**Q: What's the difference between method overloading and overriding?**
> A: **Overloading** (compile-time polymorphism) is multiple methods with the same name but different parameters in the same class. The compiler chooses which one based on the arguments.
>
> **Overriding** (runtime polymorphism) is redefining a virtual/abstract method from a base class in a derived class. The runtime determines which implementation to use based on the actual object type.

---

## Quick Review Points

### Encapsulation
- ✅ Hide internal state (private fields)
- ✅ Expose controlled access (public methods/properties)
- ✅ Validate data at entry points
- ✅ Use appropriate access modifiers

### Inheritance
- ✅ "Is-a" relationship
- ✅ Code reuse through base classes
- ✅ Use `abstract` for required overrides
- ✅ Use `virtual` for optional overrides
- ✅ Favor composition when relationship isn't hierarchical

### Abstraction
- ✅ Hide complexity, show only what's needed
- ✅ Interfaces for contracts/capabilities
- ✅ Abstract classes for shared implementation
- ✅ Program to abstractions, not implementations

### Polymorphism
- ✅ Overloading = compile-time, same name different params
- ✅ Overriding = runtime, redefine inherited method
- ✅ Enables extensibility without modifying existing code
- ✅ Foundation for Open/Closed Principle

---

## Common Interview Trap Questions

**Q: Can you instantiate an abstract class?**
> No, abstract classes cannot be instantiated directly. They must be inherited by a concrete class.

**Q: Can an interface have a constructor?**
> No, interfaces cannot have constructors. If you need initialization logic, use an abstract class.

**Q: What happens if you don't use the `override` keyword?**
> The method will hide the base method (method hiding) instead of overriding it. The compiler will warn you to use `new` keyword to explicitly indicate hiding.

**Q: Can a class implement multiple interfaces with the same method signature?**
> Yes. You can use explicit interface implementation to provide different implementations:
```csharp
public class MyClass : IInterface1, IInterface2
{
    void IInterface1.DoSomething() { /* impl 1 */ }
    void IInterface2.DoSomething() { /* impl 2 */ }
}
```
