# Architecture Concepts - Interview Questions

> 5 real-world interview questions with detailed answers and communication tactics

---

## Question 1: Microservices vs Monolith Decision

### The Question
> "When would you recommend microservices over a monolith? Walk me through your decision criteria."

### Key Points to Cover
- Decision criteria (not just definitions)
- Trade-offs of each approach
- When to start with what
- Migration considerations

### Detailed Answer

**My Decision Framework:**

```
QUESTION 1: How big is the team?
┌────────────────────────────────────────────────┐
│ < 10 developers → Monolith (simpler coordination) │
│ > 30 developers → Consider microservices       │
│ 10-30 → Depends on other factors              │
└────────────────────────────────────────────────┘

QUESTION 2: Do different parts need different scaling?
┌────────────────────────────────────────────────┐
│ Search service: 1000 req/s                     │
│ Order service: 10 req/s                        │
│ → Different scaling needs = microservices help │
└────────────────────────────────────────────────┘

QUESTION 3: Are there clear bounded contexts?
┌────────────────────────────────────────────────┐
│ Payments - own team, own release cycle         │
│ Inventory - own team, own release cycle        │
│ → Clear boundaries = can separate              │
└────────────────────────────────────────────────┘

QUESTION 4: What's the operational maturity?
┌────────────────────────────────────────────────┐
│ CI/CD pipeline? ✓                              │
│ Container orchestration? ✓                     │
│ Distributed tracing? ✓                         │
│ Team experienced with distributed systems? ✓   │
│ → Ready for microservices                      │
└────────────────────────────────────────────────┘
```

**My Recommendation:**

```csharp
// START with a well-structured monolith
// - Faster to develop initially
// - Simpler to deploy and operate
// - Easier to refactor when you learn the domain

public class WellStructuredMonolith
{
    // Modular monolith - clean separation internally
    // ├── src/
    // │   ├── Modules/
    // │   │   ├── Orders/          ← Could become a service
    // │   │   │   ├── Application/
    // │   │   │   ├── Domain/
    // │   │   │   └── Infrastructure/
    // │   │   ├── Payments/        ← Could become a service
    // │   │   ├── Inventory/       ← Could become a service
    // │   │   └── Notifications/   ← Could become a service
    // │   └── SharedKernel/
}

// EXTRACT services when:
// 1. A module has very different scaling needs
// 2. A team wants to deploy independently
// 3. You need different technology for a part
// 4. A module causes most of the deployments
```

**Trade-offs I Consider:**

```
MONOLITH
┌──────────────────────┬──────────────────────┐
│ PROS                 │ CONS                 │
├──────────────────────┼──────────────────────┤
│ Simple deployment    │ Single point of      │
│ Easy debugging       │ failure              │
│ No network latency   │ Tech stack lock-in   │
│ Simple transactions  │ Scaling is all-or-   │
│ Easy refactoring     │ nothing              │
│ One codebase         │ Large codebase grows │
└──────────────────────┴──────────────────────┘

MICROSERVICES
┌──────────────────────┬──────────────────────┐
│ PROS                 │ CONS                 │
├──────────────────────┼──────────────────────┤
│ Independent scaling  │ Network complexity   │
│ Independent deploy   │ Distributed debugging│
│ Tech diversity       │ Data consistency     │
│ Team autonomy        │ Operational overhead │
│ Fault isolation      │ Testing complexity   │
│ Smaller codebases    │ Latency added        │
└──────────────────────┴──────────────────────┘
```

**My Answer to the Interviewer:**

> "I'd recommend starting with a modular monolith for most new projects. It's faster to build, simpler to operate, and you can still have clean internal boundaries.
>
> I'd consider microservices when:
> 1. The team is large enough (30+) that coordination becomes a bottleneck
> 2. Different parts have very different scaling requirements
> 3. Teams need to deploy independently on different schedules
> 4. The organization has the operational maturity (CI/CD, monitoring, container orchestration)
>
> The worst outcome is a 'distributed monolith' - where you have all the complexity of microservices but still can't deploy independently. That happens when you extract services too early before understanding the domain boundaries."

### Communication Tactics

🎯 **Structure your answer**: Give decision criteria as questions, show you understand trade-offs, recommend starting with monolith.

💡 **Emphasize**: "I'd start with a modular monolith and extract services when there's a clear reason - not because microservices are trendy."

⚠️ **Avoid**: Don't be dogmatic about either approach. Show nuanced thinking.

---

## Question 2: Clean Architecture Layers

### The Question
> "Explain Clean Architecture. Draw the layers and explain how dependencies should flow."

### Key Points to Cover
- The layers and their responsibilities
- Dependency rule (inward only)
- Where each type of code lives
- Practical implementation

### Detailed Answer

**The Layers:**

```
┌─────────────────────────────────────────────────────────────┐
│                     INFRASTRUCTURE                          │
│  (Database, External APIs, File System, Email, etc.)       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   PRESENTATION                       │   │
│  │  (Controllers, Views, API Endpoints)                │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │               APPLICATION                    │   │   │
│  │  │  (Use Cases, Commands, Queries, DTOs)       │   │   │
│  │  │  ┌─────────────────────────────────────┐   │   │   │
│  │  │  │             DOMAIN                   │   │   │   │
│  │  │  │  (Entities, Value Objects,          │   │   │   │
│  │  │  │   Domain Services, Interfaces)      │   │   │   │
│  │  │  └─────────────────────────────────────┘   │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

DEPENDENCY RULE: Dependencies point INWARD only
- Domain knows nothing about other layers
- Application knows Domain, nothing else
- Presentation knows Application and Domain
- Infrastructure implements interfaces from inner layers
```

**Layer Responsibilities:**

```csharp
// DOMAIN LAYER - The core (no dependencies)
namespace MyApp.Domain.Entities
{
    public class Order
    {
        public Guid Id { get; private set; }
        public Money Total { get; private set; }
        public OrderStatus Status { get; private set; }

        public void AddItem(Product product, int quantity)
        {
            if (Status != OrderStatus.Draft)
                throw new DomainException("Cannot modify confirmed order");
            // Domain logic here
        }
    }
}

namespace MyApp.Domain.Interfaces
{
    // Interface defined in Domain, implemented in Infrastructure
    public interface IOrderRepository
    {
        Task<Order?> GetByIdAsync(Guid id);
        Task SaveAsync(Order order);
    }
}
```

```csharp
// APPLICATION LAYER - Use cases/business workflows
namespace MyApp.Application.Orders.Commands
{
    public record CreateOrderCommand(Guid CustomerId, List<OrderItemDto> Items)
        : IRequest<Guid>;

    public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Guid>
    {
        private readonly IOrderRepository _orderRepo;  // Interface from Domain
        private readonly IUnitOfWork _unitOfWork;

        public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
        {
            var order = Order.Create(request.CustomerId);

            foreach (var item in request.Items)
                order.AddItem(item.ProductId, item.Quantity);

            await _orderRepo.SaveAsync(order);
            await _unitOfWork.CommitAsync(ct);

            return order.Id;
        }
    }
}
```

```csharp
// INFRASTRUCTURE LAYER - External concerns
namespace MyApp.Infrastructure.Persistence
{
    public class OrderRepository : IOrderRepository  // Implements Domain interface
    {
        private readonly AppDbContext _context;

        public async Task<Order?> GetByIdAsync(Guid id)
        {
            return await _context.Orders
                .Include(o => o.Items)
                .FirstOrDefaultAsync(o => o.Id == id);
        }

        public async Task SaveAsync(Order order)
        {
            _context.Orders.Add(order);
        }
    }
}
```

```csharp
// PRESENTATION LAYER - API endpoints
namespace MyApp.Api.Controllers
{
    [ApiController]
    [Route("api/orders")]
    public class OrdersController : ControllerBase
    {
        private readonly IMediator _mediator;

        [HttpPost]
        public async Task<IActionResult> Create(CreateOrderRequest request)
        {
            var command = new CreateOrderCommand(request.CustomerId, request.Items);
            var orderId = await _mediator.Send(command);
            return CreatedAtAction(nameof(Get), new { id = orderId }, null);
        }
    }
}
```

**Project Structure:**

```
📁 src/
├── 📁 MyApp.Domain/           ← No project references
│   ├── 📁 Entities/
│   ├── 📁 ValueObjects/
│   ├── 📁 Interfaces/
│   └── 📁 Exceptions/
│
├── 📁 MyApp.Application/      ← References: Domain only
│   ├── 📁 Common/
│   ├── 📁 Orders/
│   │   ├── 📁 Commands/
│   │   └── 📁 Queries/
│   └── 📁 DTOs/
│
├── 📁 MyApp.Infrastructure/   ← References: Domain, Application
│   ├── 📁 Persistence/
│   └── 📁 ExternalServices/
│
└── 📁 MyApp.Api/              ← References: All
    ├── 📁 Controllers/
    └── Program.cs
```

### Communication Tactics

🎯 **Structure your answer**: Draw the layers, explain the dependency rule, show code for each layer.

💡 **Emphasize**: "Domain is the core with no dependencies. Everything else depends on it, not the other way around."

⚠️ **Avoid**: Don't confuse with N-tier architecture. Clean Architecture is about dependency direction, not just layers.

---

## Question 3: CQRS - When to Apply

### The Question
> "What is CQRS and when would you apply it? Is it always necessary?"

### Key Points to Cover
- What CQRS is
- When it adds value
- When it's overkill
- Implementation with MediatR

### Detailed Answer

**What is CQRS:**

```
CQRS = Command Query Responsibility Segregation

Traditional:
┌─────────────────────────────────────┐
│           Single Model              │
│  ┌─────────────────────────────┐   │
│  │     OrderService            │   │
│  │                             │   │
│  │  CreateOrder() ─┐           │   │
│  │  UpdateOrder()  │─→ Order   │   │
│  │  GetOrder()    ─┘   Entity  │   │
│  │  ListOrders()               │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘

With CQRS:
┌─────────────────────┐  ┌─────────────────────┐
│    Command Side     │  │     Query Side      │
│  ┌───────────────┐  │  │  ┌───────────────┐  │
│  │ CreateOrder   │  │  │  │ GetOrderQuery │  │
│  │ UpdateOrder   │  │  │  │ ListOrders    │  │
│  │     ↓         │  │  │  │     ↓         │  │
│  │ Rich Domain   │  │  │  │ Optimized     │  │
│  │ Model         │  │  │  │ DTOs          │  │
│  └───────────────┘  │  │  └───────────────┘  │
└─────────────────────┘  └─────────────────────┘
```

**Implementation with MediatR:**

```csharp
// COMMAND - Write side (rich domain model)
public record CreateOrderCommand(
    Guid CustomerId,
    List<OrderItemDto> Items) : IRequest<Guid>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _repository;

    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        // Use rich domain model
        var order = Order.Create(request.CustomerId);

        foreach (var item in request.Items)
        {
            order.AddItem(item.ProductId, item.Quantity, item.Price);
        }

        order.Validate();  // Domain validation

        await _repository.AddAsync(order);

        return order.Id;
    }
}

// QUERY - Read side (optimized DTOs)
public record GetOrderQuery(Guid OrderId) : IRequest<OrderDetailsDto?>;

public class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDetailsDto?>
{
    private readonly IDbConnection _connection;  // Direct DB access, no domain model

    public async Task<OrderDetailsDto?> Handle(GetOrderQuery request, CancellationToken ct)
    {
        // Optimized query - no domain model overhead
        const string sql = @"
            SELECT o.Id, o.CreatedAt, o.Status,
                   c.Name as CustomerName,
                   c.Email as CustomerEmail,
                   (SELECT SUM(oi.Price * oi.Quantity)
                    FROM OrderItems oi WHERE oi.OrderId = o.Id) as Total
            FROM Orders o
            JOIN Customers c ON o.CustomerId = c.Id
            WHERE o.Id = @OrderId";

        return await _connection.QueryFirstOrDefaultAsync<OrderDetailsDto>(
            sql, new { request.OrderId });
    }
}
```

**When to Use CQRS:**

```csharp
// ✅ USE CQRS WHEN:

// 1. Read and write patterns are very different
// Write: Complex validation, business rules
// Read: Simple flat DTOs with joins

// 2. Read-heavy workloads
// Can optimize queries independently
// Can use different data stores for reads

// 3. Complex domain with rich write model
// Commands use Aggregates, business logic
// Queries bypass domain for performance

// 4. Event Sourcing (natural fit)
// Commands generate events
// Queries read from projections

// 5. Different scaling requirements
// Scale read side independently
```

```csharp
// ❌ DON'T USE CQRS WHEN:

// 1. Simple CRUD application
// Same model works for both read/write
// Overhead isn't justified

// 2. Small, simple domain
// No complex business rules
// No performance issues

// 3. Team unfamiliar with the pattern
// Adds learning curve
// Can be misapplied

// Example: Simple blog system
// - CreatePost, UpdatePost, GetPost
// - No complex business rules
// - Same Post entity works everywhere
// → CQRS is overkill
```

**Spectrum of CQRS:**

```
Light CQRS:                             Heavy CQRS:
Same DB, different models ←───────────→ Different DBs, Event Sourcing

┌─────────────────────┐               ┌─────────────────────┐
│ Commands and Queries│               │ Commands write to   │
│ use same database   │               │ event store         │
│                     │               │                     │
│ Different models:   │               │ Projections read    │
│ - Commands: Entities│               │ from read DB        │
│ - Queries: DTOs     │               │                     │
└─────────────────────┘               └─────────────────────┘

Start here                             Only if needed
(usually sufficient)                   (adds complexity)
```

### Communication Tactics

🎯 **Structure your answer**: Define CQRS, show when it helps, show when it's overkill, mention the spectrum.

💡 **Emphasize**: "CQRS is a tool, not a requirement. I'd use it when read and write patterns are genuinely different."

⚠️ **Avoid**: Don't suggest always using CQRS. Show you understand it's situational.

---

## Question 4: DDD Core Concepts

### The Question
> "Explain the core DDD concepts: Entity, Value Object, Aggregate, and Bounded Context. Give me examples."

### Key Points to Cover
- Clear definitions with examples
- Entity vs Value Object distinction
- Aggregate rules
- Bounded Context boundaries

### Detailed Answer

**Entity - Has Identity:**

```csharp
// Entity = defined by its identity, not its attributes
// Two orders with same items are still DIFFERENT orders

public class Order  // ENTITY
{
    public Guid Id { get; private set; }  // Identity
    public List<OrderItem> Items { get; } = new();
    public OrderStatus Status { get; private set; }

    // Can change attributes, still same order
    public void AddItem(Product product, int quantity)
    {
        Items.Add(new OrderItem(product.Id, quantity, product.Price));
    }

    // Identity comparison
    public override bool Equals(object? obj)
    {
        return obj is Order other && Id == other.Id;
    }
}

// Customer is also an Entity
public class Customer
{
    public Guid Id { get; private set; }
    public string Email { get; private set; }
    public string Name { get; private set; }

    // Customer can change email, still same customer
    public void UpdateEmail(string newEmail) => Email = newEmail;
}
```

**Value Object - No Identity, Defined by Attributes:**

```csharp
// Value Object = defined by its values, immutable
// Two Money objects with same amount/currency ARE the same

public record Money  // VALUE OBJECT (using record for value equality)
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative");
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    // Operations return NEW instances (immutable)
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(Amount + other.Amount, Currency);
    }
}

// Address is also a Value Object
public record Address(string Street, string City, string Country, string PostalCode)
{
    // Two addresses with same values are equal
    // No identity - if address changes, it's a new address
}

// Date range is a Value Object
public record DateRange(DateTime Start, DateTime End)
{
    public DateRange
    {
        if (End < Start)
            throw new ArgumentException("End must be after Start");
    }

    public bool Contains(DateTime date) => date >= Start && date <= End;
    public bool Overlaps(DateRange other) => Start < other.End && End > other.Start;
}
```

**Aggregate - Cluster with Consistency Boundary:**

```csharp
// Aggregate = cluster of objects treated as a single unit
// One entity is the Aggregate Root - the entry point

public class Order  // AGGREGATE ROOT
{
    public Guid Id { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public Money Total { get; private set; }
    public OrderStatus Status { get; private set; }

    // All modifications go through the root
    public void AddItem(Guid productId, int quantity, Money price)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Cannot modify confirmed order");

        var item = _items.FirstOrDefault(i => i.ProductId == productId);
        if (item != null)
            item.IncreaseQuantity(quantity);
        else
            _items.Add(new OrderItem(productId, quantity, price));

        RecalculateTotal();
    }

    public void Confirm()
    {
        if (!_items.Any())
            throw new DomainException("Cannot confirm empty order");
        Status = OrderStatus.Confirmed;
    }

    private void RecalculateTotal()
    {
        Total = _items.Aggregate(
            new Money(0, "TRY"),
            (sum, item) => sum.Add(item.Subtotal));
    }
}

public class OrderItem  // Part of Order aggregate, NOT a root
{
    public Guid ProductId { get; private set; }
    public int Quantity { get; private set; }
    public Money Price { get; private set; }
    public Money Subtotal => new(Price.Amount * Quantity, Price.Currency);

    internal void IncreaseQuantity(int amount)  // Internal - only Order can call
    {
        Quantity += amount;
    }
}

// AGGREGATE RULES:
// 1. Reference aggregates by ID, not by object
// 2. Modify only through the root
// 3. One transaction = one aggregate
// 4. Keep aggregates small
```

**Bounded Context - Clear Domain Boundaries:**

```csharp
// Different contexts have DIFFERENT models of same concept

// === ORDERING CONTEXT ===
namespace Ordering.Domain
{
    public class Customer  // Simple - just need ID and basic info
    {
        public Guid Id { get; }
        public string Name { get; }
        public string Email { get; }
    }

    public class Product  // Just need ID and price
    {
        public Guid Id { get; }
        public string Name { get; }
        public Money Price { get; }
    }
}

// === CRM CONTEXT ===
namespace CRM.Domain
{
    public class Customer  // Rich - full customer details
    {
        public Guid Id { get; }
        public string Name { get; }
        public string Email { get; }
        public CustomerSegment Segment { get; }
        public List<CustomerInteraction> Interactions { get; }
        public decimal LifetimeValue { get; }
        public DateTime LastContactDate { get; }
    }
}

// === INVENTORY CONTEXT ===
namespace Inventory.Domain
{
    public class Product  // Rich - stock and warehouse info
    {
        public Guid Id { get; }
        public string SKU { get; }
        public int StockLevel { get; }
        public int ReorderPoint { get; }
        public List<WarehouseStock> WarehouseStocks { get; }
    }
}

// Contexts communicate via:
// - Integration events (async)
// - Shared kernel (common types)
// - Anti-corruption layer (translate between contexts)
```

### Communication Tactics

🎯 **Structure your answer**: Define each concept, give clear examples, show the relationships.

💡 **Emphasize**: "Entity = identity matters, Value Object = values matter, Aggregate = consistency boundary, Bounded Context = model boundary."

⚠️ **Avoid**: Don't make it too abstract. Concrete examples (Order, Money, Address) make concepts clear.

---

## Question 5: Eventual Consistency

### The Question
> "How do you handle eventual consistency in distributed systems? What patterns help?"

### Key Points to Cover
- What eventual consistency means
- Why it's necessary in distributed systems
- Patterns to handle it
- Practical implementation

### Detailed Answer

**Why Eventual Consistency:**

```
In distributed systems, you often can't have:
- Strong consistency (all nodes see same data immediately)
- High availability (system always responds)
- Partition tolerance (works despite network issues)

(CAP Theorem - pick 2 of 3)

Most systems choose: Availability + Partition Tolerance
Result: Eventual Consistency

The system will become consistent... eventually
(usually within milliseconds to seconds)
```

**Example - Order Processing:**

```
EVENTUALLY CONSISTENT FLOW:

1. Customer places order
   ┌─────────────────┐
   │  Order Service  │───→ Order saved, "Pending"
   └────────┬────────┘
            │ Publishes OrderPlaced event
            ▼
   ┌─────────────────┐
   │  Message Queue  │
   └────────┬────────┘
            │
   ┌────────┼────────┬───────────────┐
   │        │        │               │
   ▼        ▼        ▼               ▼
┌──────┐ ┌──────┐ ┌──────┐      ┌──────┐
│Invent│ │Paymt │ │Notif │      │Loyalt│
│ory   │ │      │ │      │      │y     │
└──────┘ └──────┘ └──────┘      └──────┘
   │        │        │               │
   │ Eventually consistent          │
   │ (each updates in its own time) │
   └────────────────────────────────┘

At time T+0: Order exists, but inventory not yet reserved
At time T+100ms: Inventory reserved
At time T+200ms: Payment processed
At time T+500ms: Notification sent

EVENTUALLY: All systems consistent
```

**Patterns for Handling:**

**1. Saga Pattern - Distributed Transactions:**

```csharp
public class OrderSaga
{
    public async Task ExecuteAsync(CreateOrderCommand command)
    {
        var sagaState = new OrderSagaState();

        try
        {
            // Step 1: Create order
            sagaState.OrderId = await _orderService.CreateAsync(command);

            // Step 2: Reserve inventory
            sagaState.InventoryReserved = await _inventoryService.ReserveAsync(
                sagaState.OrderId, command.Items);

            // Step 3: Process payment
            sagaState.PaymentId = await _paymentService.ChargeAsync(
                command.CustomerId, command.Total);

            // Step 4: Confirm order
            await _orderService.ConfirmAsync(sagaState.OrderId);
        }
        catch (Exception)
        {
            // Compensating transactions (rollback)
            await CompensateAsync(sagaState);
            throw;
        }
    }

    private async Task CompensateAsync(OrderSagaState state)
    {
        if (state.PaymentId != null)
            await _paymentService.RefundAsync(state.PaymentId);

        if (state.InventoryReserved)
            await _inventoryService.ReleaseAsync(state.OrderId);

        if (state.OrderId != null)
            await _orderService.CancelAsync(state.OrderId);
    }
}
```

**2. Outbox Pattern - Reliable Event Publishing:**

```csharp
public class OrderService
{
    public async Task CreateOrderAsync(Order order)
    {
        using var transaction = await _context.Database.BeginTransactionAsync();

        try
        {
            // 1. Save the order
            _context.Orders.Add(order);

            // 2. Save the event to outbox (same transaction!)
            _context.OutboxMessages.Add(new OutboxMessage
            {
                Id = Guid.NewGuid(),
                Type = "OrderCreated",
                Payload = JsonSerializer.Serialize(new OrderCreatedEvent(order)),
                CreatedAt = DateTime.UtcNow,
                ProcessedAt = null
            });

            await _context.SaveChangesAsync();
            await transaction.CommitAsync();

            // Order and event saved atomically
            // Background worker will publish event to message queue
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}

// Background worker publishes outbox messages
public class OutboxProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var messages = await _context.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.CreatedAt)
                .Take(100)
                .ToListAsync();

            foreach (var message in messages)
            {
                await _messageBus.PublishAsync(message.Type, message.Payload);
                message.ProcessedAt = DateTime.UtcNow;
            }

            await _context.SaveChangesAsync();
            await Task.Delay(1000, ct);  // Poll every second
        }
    }
}
```

**3. Idempotent Event Handlers:**

```csharp
public class InventoryEventHandler
{
    public async Task HandleOrderCreatedAsync(OrderCreatedEvent @event)
    {
        // Idempotency check - already processed?
        if (await _processedEvents.ContainsAsync(@event.EventId))
            return;

        // Process the event
        foreach (var item in @event.Items)
        {
            await _inventoryService.ReserveAsync(item.ProductId, item.Quantity);
        }

        // Mark as processed
        await _processedEvents.AddAsync(@event.EventId);
    }
}
```

**Communicating Status to Users:**

```csharp
// Don't hide eventual consistency - communicate it!
public class OrderController
{
    [HttpPost]
    public async Task<IActionResult> CreateOrder(CreateOrderRequest request)
    {
        var orderId = await _orderService.CreateAsync(request);

        return Accepted(new
        {
            OrderId = orderId,
            Status = "Processing",
            Message = "Your order is being processed. You'll receive a confirmation email shortly.",
            CheckStatusUrl = $"/api/orders/{orderId}/status"
        });
    }

    [HttpGet("{id}/status")]
    public async Task<IActionResult> GetStatus(Guid id)
    {
        var status = await _orderService.GetStatusAsync(id);

        return Ok(new
        {
            OrderId = id,
            Status = status.CurrentStatus,
            Steps = new[]
            {
                new { Name = "Order Created", Completed = true },
                new { Name = "Inventory Reserved", Completed = status.InventoryReserved },
                new { Name = "Payment Processed", Completed = status.PaymentProcessed },
                new { Name = "Ready for Shipping", Completed = status.ReadyForShipping }
            }
        });
    }
}
```

### Communication Tactics

🎯 **Structure your answer**: Explain why eventual consistency is needed, show the patterns (Saga, Outbox), mention idempotency.

💡 **Emphasize**: "Eventual consistency is a trade-off for availability and scale. The key is making it explicit and handling failures gracefully."

⚠️ **Avoid**: Don't pretend you can avoid it in distributed systems. Show you understand the trade-offs.

---

## Quick Review - Key Takeaways

| Question | Key Point |
|----------|-----------|
| Microservices vs Monolith | Start with modular monolith, extract when needed |
| Clean Architecture | Dependencies point inward, Domain at center |
| CQRS | Use when read/write patterns differ, not always |
| DDD Concepts | Entity=identity, Value Object=values, Aggregate=boundary |
| Eventual Consistency | Saga for transactions, Outbox for reliable events |
