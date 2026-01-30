# Architecture Concepts

> Microservices, Clean Architecture, DDD, CQRS for senior-level discussions

---

## Microservices vs Monolith

### Monolith

```
┌─────────────────────────────────────────┐
│              Monolith                   │
│  ┌─────────────────────────────────┐   │
│  │         Presentation            │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │      Business Logic             │   │
│  │  Orders │ Users │ Inventory     │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │         Data Access             │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │       Single Database           │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**Advantages:**
- Simple to develop, test, deploy
- Easy debugging (single process)
- No network latency between components
- Transactions are straightforward (ACID)
- Single codebase to understand

**Disadvantages:**
- Scaling requires scaling everything
- One bug can crash the whole system
- Technology lock-in
- Large codebase becomes hard to maintain
- Long deployment cycles
- Team coupling

### Microservices

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ Order Svc   │   │ User Svc    │   │Inventory Svc│
│  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │
│  │ API   │  │   │  │ API   │  │   │  │ API   │  │
│  │Logic  │  │   │  │Logic  │  │   │  │Logic  │  │
│  │  DB   │  │   │  │  DB   │  │   │  │  DB   │  │
│  └───────┘  │   │  └───────┘  │   │  └───────┘  │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
              ┌──────────▼──────────┐
              │    Message Bus      │
              │   (Kafka/RabbitMQ)  │
              └─────────────────────┘
```

**Advantages:**
- Independent scaling per service
- Independent deployment
- Technology diversity
- Fault isolation
- Team autonomy
- Better for large organizations

**Disadvantages:**
- Distributed system complexity
- Network latency and failures
- Data consistency challenges
- Operational overhead
- Testing complexity
- Debugging across services

### When to Use Each

| Criteria | Monolith | Microservices |
|----------|----------|---------------|
| Team size | <20 developers | Large, multiple teams |
| Domain complexity | Well-understood | Complex, distinct bounded contexts |
| Scaling needs | Uniform | Different parts scale differently |
| Time to market | Fast initial development | Longer setup, faster iteration later |
| Organization | Single team | Multiple independent teams |

### Interview Answer

**Q: When would you recommend microservices over a monolith?**

> "I'd consider microservices when:
> 1. The domain has clear bounded contexts that can be owned by different teams
> 2. Different parts of the system have different scaling requirements
> 3. We need to deploy parts independently without affecting others
> 4. The team is large enough to justify the operational overhead
>
> However, I'd start with a well-structured monolith for a new project. It's much easier to identify service boundaries after you understand the domain. Premature decomposition often leads to distributed monolith—the worst of both worlds."

---

## Clean Architecture

### Overview

Clean Architecture (by Robert C. Martin) organizes code into concentric layers with dependencies pointing inward. The inner layers don't know about outer layers.

```
┌─────────────────────────────────────────────────────────────┐
│                    Infrastructure                            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Presentation                      │    │
│  │  ┌─────────────────────────────────────────────┐    │    │
│  │  │               Application                    │    │    │
│  │  │  ┌─────────────────────────────────────┐    │    │    │
│  │  │  │             Domain                   │    │    │    │
│  │  │  │  ┌─────────────────────────────┐    │    │    │    │
│  │  │  │  │         Entities            │    │    │    │    │
│  │  │  │  └─────────────────────────────┘    │    │    │    │
│  │  │  └─────────────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘

Dependencies flow INWARD →
```

### Layer Responsibilities

| Layer | Responsibility | Examples |
|-------|---------------|----------|
| **Domain (Entities)** | Enterprise business rules | `Order`, `Customer`, `Money` |
| **Application (Use Cases)** | Application business rules | `CreateOrderCommand`, `IOrderRepository` |
| **Presentation** | UI, API endpoints | Controllers, Views |
| **Infrastructure** | External concerns | Database, Email, APIs |

### Project Structure

```
📁 src/
├── 📁 MyApp.Domain/
│   ├── 📁 Entities/
│   │   ├── Order.cs
│   │   └── Customer.cs
│   ├── 📁 ValueObjects/
│   │   ├── Money.cs
│   │   └── Address.cs
│   ├── 📁 Enums/
│   │   └── OrderStatus.cs
│   └── 📁 Exceptions/
│       └── DomainException.cs
│
├── 📁 MyApp.Application/
│   ├── 📁 Common/
│   │   ├── 📁 Interfaces/
│   │   │   ├── IOrderRepository.cs
│   │   │   └── IUnitOfWork.cs
│   │   └── 📁 Behaviors/
│   │       └── ValidationBehavior.cs
│   ├── 📁 Orders/
│   │   ├── 📁 Commands/
│   │   │   ├── CreateOrderCommand.cs
│   │   │   └── CreateOrderCommandHandler.cs
│   │   └── 📁 Queries/
│   │       ├── GetOrderQuery.cs
│   │       └── GetOrderQueryHandler.cs
│   └── 📁 DTOs/
│       └── OrderDto.cs
│
├── 📁 MyApp.Infrastructure/
│   ├── 📁 Persistence/
│   │   ├── AppDbContext.cs
│   │   └── OrderRepository.cs
│   ├── 📁 Services/
│   │   └── EmailService.cs
│   └── DependencyInjection.cs
│
└── 📁 MyApp.Api/
    ├── 📁 Controllers/
    │   └── OrdersController.cs
    ├── Program.cs
    └── DependencyInjection.cs
```

### Implementation Example

```csharp
// === DOMAIN LAYER ===

// Entity
public class Order
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public Money Total => Money.FromItems(_items);

    private Order() { } // For EF

    public static Order Create(Guid customerId)
    {
        return new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Pending
        };
    }

    public void AddItem(Guid productId, int quantity, Money price)
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Cannot modify non-pending order");

        _items.Add(new OrderItem(productId, quantity, price));
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Only pending orders can be confirmed");
        if (!_items.Any())
            throw new DomainException("Cannot confirm empty order");

        Status = OrderStatus.Confirmed;
    }
}

// === APPLICATION LAYER ===

// Interface (defined in Application, implemented in Infrastructure)
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id);
    Task AddAsync(Order order);
}

// Command
public record CreateOrderCommand(Guid CustomerId, List<OrderItemDto> Items)
    : IRequest<Guid>;

// Handler
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public CreateOrderCommandHandler(IOrderRepository repository, IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
    }

    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId);

        foreach (var item in request.Items)
        {
            order.AddItem(item.ProductId, item.Quantity, new Money(item.Price, "TRY"));
        }

        await _repository.AddAsync(order);
        await _unitOfWork.SaveChangesAsync(ct);

        return order.Id;
    }
}

// === INFRASTRUCTURE LAYER ===

// Repository implementation
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Order?> GetByIdAsync(Guid id)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id);
    }

    public async Task AddAsync(Order order)
    {
        await _context.Orders.AddAsync(order);
    }
}

// === PRESENTATION LAYER ===

[ApiController]
[Route("api/[controller]")]
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
        var orderId = await _mediator.Send(command);
        return CreatedAtAction(nameof(Get), new { id = orderId }, new { id = orderId });
    }
}
```

### Key Principles

1. **Dependency Rule**: Dependencies point inward. Inner layers don't know outer layers.
2. **Entities are pure**: No framework dependencies, no persistence concerns.
3. **Use Cases orchestrate**: Application layer contains business workflows.
4. **Interfaces in Application**: Infrastructure implements interfaces defined in Application.
5. **Framework independence**: Business logic doesn't depend on ASP.NET, EF, etc.

---

## Domain-Driven Design (DDD) Basics

### Key Concepts

| Concept | Description | Example |
|---------|-------------|---------|
| **Entity** | Object with identity | Order, Customer |
| **Value Object** | Immutable, no identity | Money, Address |
| **Aggregate** | Cluster of entities/VOs with one root | Order (root) + OrderItems |
| **Aggregate Root** | Entry point for aggregate | Order |
| **Domain Event** | Something that happened | OrderPlaced, PaymentReceived |
| **Repository** | Persistence abstraction | IOrderRepository |
| **Domain Service** | Logic that doesn't fit entities | PricingService |
| **Bounded Context** | Boundary around a domain model | Orders, Inventory, Shipping |

### Entity vs Value Object

```csharp
// ENTITY - Has identity, mutable
public class Customer
{
    public Guid Id { get; private set; }  // Identity
    public string Email { get; private set; }
    public string Name { get; private set; }

    public void ChangeName(string newName)
    {
        // Can change, but still same customer
        Name = newName;
    }
}

// VALUE OBJECT - No identity, immutable
public record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative");
        if (string.IsNullOrEmpty(currency))
            throw new ArgumentException("Currency is required");

        Amount = amount;
        Currency = currency;
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");

        return new Money(Amount + other.Amount, Currency);
    }

    // Two Money objects with same amount and currency are EQUAL
    // (records provide this by default)
}

public record Address(string Street, string City, string Country, string PostalCode);
```

### Aggregate

```csharp
// Order is the Aggregate Root
// OrderItem can only be accessed through Order
public class Order  // AGGREGATE ROOT
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }

    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    private Order() { }

    public static Order Create(Guid customerId)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Pending
        };

        order._domainEvents.Add(new OrderCreatedEvent(order.Id, customerId));

        return order;
    }

    // Business logic lives in the aggregate
    public void AddItem(Product product, int quantity)
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Cannot modify non-pending order");

        var existingItem = _items.FirstOrDefault(i => i.ProductId == product.Id);
        if (existingItem != null)
        {
            existingItem.IncreaseQuantity(quantity);
        }
        else
        {
            _items.Add(new OrderItem(product.Id, product.Price, quantity));
        }
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Only pending orders can be confirmed");

        if (!_items.Any())
            throw new DomainException("Cannot confirm empty order");

        Status = OrderStatus.Confirmed;
        _domainEvents.Add(new OrderConfirmedEvent(Id));
    }

    public void ClearDomainEvents() => _domainEvents.Clear();
}

// OrderItem is part of Order aggregate, not accessible directly
public class OrderItem  // ENTITY within aggregate
{
    public Guid Id { get; private set; }
    public Guid ProductId { get; private set; }
    public Money Price { get; private set; }
    public int Quantity { get; private set; }

    internal OrderItem(Guid productId, Money price, int quantity)
    {
        Id = Guid.NewGuid();
        ProductId = productId;
        Price = price;
        Quantity = quantity;
    }

    internal void IncreaseQuantity(int amount)
    {
        Quantity += amount;
    }
}
```

### Bounded Contexts

```
┌─────────────────────────┐     ┌─────────────────────────┐
│    Orders Context       │     │   Inventory Context     │
│                         │     │                         │
│  Order                  │     │  Product                │
│  - Id                   │     │  - Id                   │
│  - CustomerId           │     │  - SKU                  │
│  - Items                │     │  - StockLevel           │
│  - Total                │     │  - ReorderPoint         │
│                         │     │                         │
│  Customer (simplified)  │     │  Warehouse              │
│  - Id                   │     │  - Location             │
│  - Name                 │     │  - Capacity             │
│                         │     │                         │
└──────────┬──────────────┘     └──────────┬──────────────┘
           │                               │
           │    Integration Events         │
           └───────────────────────────────┘

// Different contexts have different models of same concept
// In Orders: Customer is simple (just needs ID and name)
// In CRM: Customer is rich (preferences, history, segments)
```

---

## CQRS (Command Query Responsibility Segregation)

### Overview

CQRS separates read (Query) and write (Command) operations into different models. This allows optimizing each side independently.

```
                    ┌─────────────────┐
                    │     Client      │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
       ┌──────▼──────┐               ┌──────▼──────┐
       │   Command   │               │    Query    │
       │    Side     │               │    Side     │
       └──────┬──────┘               └──────┬──────┘
              │                             │
       ┌──────▼──────┐               ┌──────▼──────┐
       │   Domain    │               │  Read Model │
       │   Model     │               │  (DTOs)     │
       └──────┬──────┘               └──────┬──────┘
              │                             │
       ┌──────▼──────┐               ┌──────▼──────┐
       │   Write     │               │    Read     │
       │   Database  │───Sync───────▶│   Database  │
       └─────────────┘               └─────────────┘
```

### Implementation with MediatR

```csharp
// === COMMAND (Write Side) ===

public record CreateOrderCommand(Guid CustomerId, List<OrderItemDto> Items)
    : IRequest<Guid>;

public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        // Uses rich domain model
        var order = Order.Create(request.CustomerId);

        foreach (var item in request.Items)
        {
            order.AddItem(item.ProductId, item.Quantity, new Money(item.Price, "TRY"));
        }

        await _repository.AddAsync(order);
        await _unitOfWork.SaveChangesAsync(ct);

        return order.Id;
    }
}

// === QUERY (Read Side) ===

public record GetOrderQuery(Guid OrderId) : IRequest<OrderDto?>;

public class GetOrderQueryHandler : IRequestHandler<GetOrderQuery, OrderDto?>
{
    private readonly IDbConnection _connection;  // Direct DB access, no domain model

    public async Task<OrderDto?> Handle(GetOrderQuery request, CancellationToken ct)
    {
        // Optimized read query - returns DTO directly
        const string sql = @"
            SELECT o.Id, o.Status, o.CreatedAt, c.Name as CustomerName,
                   SUM(oi.Quantity * oi.Price) as Total
            FROM Orders o
            JOIN Customers c ON o.CustomerId = c.Id
            JOIN OrderItems oi ON o.Id = oi.OrderId
            WHERE o.Id = @OrderId
            GROUP BY o.Id, o.Status, o.CreatedAt, c.Name";

        return await _connection.QueryFirstOrDefaultAsync<OrderDto>(sql,
            new { request.OrderId });
    }
}

// Or using EF Core for simpler queries
public class GetOrderQueryHandler : IRequestHandler<GetOrderQuery, OrderDto?>
{
    private readonly AppDbContext _context;

    public async Task<OrderDto?> Handle(GetOrderQuery request, CancellationToken ct)
    {
        return await _context.Orders
            .Where(o => o.Id == request.OrderId)
            .Select(o => new OrderDto
            {
                Id = o.Id,
                CustomerName = o.Customer.Name,
                Status = o.Status.ToString(),
                Total = o.Items.Sum(i => i.Price * i.Quantity),
                CreatedAt = o.CreatedAt
            })
            .FirstOrDefaultAsync(ct);
    }
}
```

### When to Use CQRS

**Good fit:**
- Read and write patterns are very different
- Need to scale reads and writes independently
- Complex domain with rich write model
- Different teams working on read/write

**Not needed:**
- Simple CRUD applications
- Same model works for both read/write
- Small applications

---

## Event-Driven Architecture

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Event** | Immutable record of something that happened |
| **Producer** | Service that publishes events |
| **Consumer** | Service that subscribes to events |
| **Message Broker** | Delivers events (Kafka, RabbitMQ, Azure Service Bus) |
| **Eventual Consistency** | System becomes consistent over time |

### Implementation

```csharp
// Domain Event
public record OrderPlacedEvent(
    Guid OrderId,
    Guid CustomerId,
    decimal Total,
    DateTime PlacedAt) : IDomainEvent;

// Publishing event (after saving aggregate)
public class OrderService
{
    private readonly IEventPublisher _eventPublisher;

    public async Task PlaceOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);

        // Publish domain events
        foreach (var @event in order.DomainEvents)
        {
            await _eventPublisher.PublishAsync(@event);
        }

        order.ClearDomainEvents();
    }
}

// Consumer in Inventory Service
public class InventoryEventHandler
{
    public async Task HandleOrderPlacedAsync(OrderPlacedEvent @event)
    {
        // Reserve inventory
        foreach (var item in @event.Items)
        {
            await _inventoryService.ReserveAsync(item.ProductId, item.Quantity);
        }
    }
}

// Consumer in Notification Service
public class NotificationEventHandler
{
    public async Task HandleOrderPlacedAsync(OrderPlacedEvent @event)
    {
        // Send confirmation email
        await _emailService.SendOrderConfirmationAsync(@event.CustomerId, @event.OrderId);
    }
}
```

### Eventual Consistency

```
User places order
       │
       ▼
┌─────────────────┐
│  Order Service  │ ──── Saves order ────────────────┐
└────────┬────────┘                                  │
         │                                           │
         │ Publishes OrderPlaced                     ▼
         │                               ┌───────────────────┐
         ▼                               │ Orders Database   │
┌─────────────────┐                      └───────────────────┘
│  Message Broker │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐ ┌──────────┐
│Inv Svc│ │Notif Svc │ ──── Eventually consistent ───┐
└───┬───┘ └────┬─────┘                               │
    │          │                                     ▼
    ▼          ▼                         ┌───────────────────┐
Reserve    Send                          │ System is consistent│
Stock      Email                         │ (eventually)        │
                                         └───────────────────┘
```

---

## API Design Best Practices

### RESTful Conventions

```
GET     /api/orders         # List orders
GET     /api/orders/{id}    # Get single order
POST    /api/orders         # Create order
PUT     /api/orders/{id}    # Full update
PATCH   /api/orders/{id}    # Partial update
DELETE  /api/orders/{id}    # Delete order

# Nested resources
GET     /api/orders/{id}/items
POST    /api/orders/{id}/items

# Actions (when CRUD doesn't fit)
POST    /api/orders/{id}/cancel
POST    /api/orders/{id}/ship
```

### HTTP Status Codes

| Code | Meaning | Use Case |
|------|---------|----------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Validation errors |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Concurrency conflict |
| 422 | Unprocessable | Business rule violation |
| 500 | Server Error | Unexpected error |

### Error Responses (Problem Details)

```csharp
// Use Problem Details (RFC 7807)
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;

        var problemDetails = exception switch
        {
            ValidationException ve => new ProblemDetails
            {
                Status = 400,
                Title = "Validation Error",
                Detail = ve.Message,
                Extensions = { ["errors"] = ve.Errors }
            },
            NotFoundException nf => new ProblemDetails
            {
                Status = 404,
                Title = "Not Found",
                Detail = nf.Message
            },
            _ => new ProblemDetails
            {
                Status = 500,
                Title = "Server Error",
                Detail = "An unexpected error occurred"
            }
        };

        context.Response.StatusCode = problemDetails.Status ?? 500;
        await context.Response.WriteAsJsonAsync(problemDetails);
    });
});

// Response example
{
    "type": "https://tools.ietf.org/html/rfc7807",
    "title": "Validation Error",
    "status": 400,
    "detail": "One or more validation errors occurred",
    "errors": {
        "email": ["Email is required"],
        "amount": ["Amount must be positive"]
    }
}
```

### API Versioning

```csharp
// URL versioning (most explicit)
app.MapGroup("/api/v1/orders").MapOrderEndpointsV1();
app.MapGroup("/api/v2/orders").MapOrderEndpointsV2();

// Header versioning
// X-Api-Version: 2

// Query string versioning
// /api/orders?api-version=2

// With Asp.Versioning.Http
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});

[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetV1() { }

    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetV2() { }
}
```

---

## Quick Reference

| Concept | Key Point |
|---------|-----------|
| **Monolith** | Simple, good for small teams, harder to scale |
| **Microservices** | Complex, good for large orgs, independent scaling |
| **Clean Architecture** | Dependencies point inward, domain at center |
| **DDD** | Model the domain, use aggregates, bounded contexts |
| **CQRS** | Separate read/write models when needed |
| **Event-Driven** | Loose coupling, eventual consistency |

### Interview Tips

**Q: How do you decide between monolith and microservices?**
> "I consider team size, domain complexity, and scaling needs. I'd start with a well-structured monolith (using Clean Architecture) and extract services only when there's a clear bounded context that benefits from independent deployment or scaling. Premature microservices often create unnecessary complexity."

**Q: What's the difference between Clean Architecture and DDD?**
> "They're complementary. **Clean Architecture** is about organizing code layers with dependencies pointing inward—it's structural. **DDD** is about modeling the domain with entities, value objects, and aggregates—it's about the domain model itself. You can use DDD concepts within a Clean Architecture structure."
