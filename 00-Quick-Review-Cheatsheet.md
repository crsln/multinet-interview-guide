# Quick Review Cheatsheet

> Last-minute review of all key points for Multinet Up Senior .NET Developer interview

---

## OOP 4 Pillars (HIGH PRIORITY - They WILL ask)

### Encapsulation
- **What**: Bundle data + methods, hide internal state
- **How**: Private fields, public methods/properties
- **Why**: Protect data integrity, control access, reduce coupling

### Inheritance
- **What**: Derive new class from existing class (is-a relationship)
- **How**: `: BaseClass`, `virtual`/`override`, `abstract`
- **Why**: Code reuse, polymorphism
- **Tip**: Favor composition over inheritance when relationship isn't clear

### Abstraction
- **What**: Hide complexity, show only what's needed
- **How**: Interfaces (contract), Abstract classes (shared implementation)
- **Interface vs Abstract**:
  - Interface: Multiple inheritance, no implementation (before C# 8)
  - Abstract: Single inheritance, can have implementation

### Polymorphism
- **Compile-time**: Method overloading (same name, different params)
- **Runtime**: Method overriding (virtual/override, actual type determines behavior)
- **Example**: `List<Animal>` - each animal's `Speak()` calls its own implementation

---

## SOLID Principles (HIGH PRIORITY)

| Principle | Meaning | Remember |
|-----------|---------|----------|
| **S**RP | Single Responsibility | One class = one reason to change |
| **O**CP | Open/Closed | Open for extension, closed for modification |
| **L**SP | Liskov Substitution | Derived class must be substitutable for base |
| **I**SP | Interface Segregation | Many small interfaces > one big interface |
| **D**IP | Dependency Inversion | Depend on abstractions, not concretions |

---

## Unit Testing (HIGH PRIORITY)

### Why Test?
1. Catch bugs early (cheaper to fix)
2. Enable safe refactoring
3. Document expected behavior
4. Force better design (testable = loosely coupled)

### Key Concepts
- **xUnit**: `[Fact]` single test, `[Theory]` parameterized
- **Moq**: Mock dependencies with `.Setup()` and `.Verify()`
- **Pattern**: Arrange → Act → Assert
- **Naming**: `MethodName_Scenario_ExpectedResult`

### Quick Moq Example
```csharp
var mock = new Mock<IRepository>();
mock.Setup(r => r.GetAsync(1)).ReturnsAsync(new Order());
mock.Verify(r => r.GetAsync(1), Times.Once);
```

---

## Design Patterns (HIGH PRIORITY)

### Creational
| Pattern | Use | Remember |
|---------|-----|----------|
| **Singleton** | One instance | `Lazy<T>` for thread-safe |
| **Factory** | Create objects | Decouple creation logic |
| **Builder** | Complex objects | Fluent API with `WithX()` methods |

### Structural
| Pattern | Use | Remember |
|---------|-----|----------|
| **Adapter** | Convert interfaces | Legacy integration |
| **Decorator** | Add behavior | Stack behaviors dynamically |
| **Facade** | Simplify subsystem | One simple method, many internal calls |
| **Proxy** | Control access | Caching, auth, lazy loading |

### Behavioral
| Pattern | Use | Remember |
|---------|-----|----------|
| **Strategy** | Swap algorithms | Interface + multiple implementations |
| **Observer** | Notify changes | C# events |
| **Command** | Encapsulate requests | Undo/redo, CQRS |
| **Mediator** | Centralize communication | MediatR library |

### Enterprise
| Pattern | Use |
|---------|-----|
| **Repository** | Abstract data access |
| **Unit of Work** | Transaction coordination |
| **Circuit Breaker** | Handle failures (Polly) |
| **Outbox** | Reliable messaging |

---

## .NET 8/9 & C# 12/13

### .NET 8 Features (LTS - Know These)
- **Frozen Collections**: Read-optimized `ToFrozenDictionary()`
- **Native AOT**: Faster startup, smaller footprint
- **Keyed Services**: `[FromKeyedServices("key")]`
- **TimeProvider**: Testable time abstraction

### C# 12 Features
```csharp
// Primary constructors
public class Service(IRepo repo) { }

// Collection expressions
int[] arr = [1, 2, 3];
int[] combined = [..arr1, ..arr2];

// Type aliases
using Point = (int X, int Y);
```

---

## Architecture Concepts

### Monolith vs Microservices
- **Monolith**: Simple, fast to start, harder to scale
- **Microservices**: Complex, independent scaling/deployment
- **Decision**: Start monolith, extract when clear bounded contexts emerge

### Clean Architecture
- Dependencies point INWARD
- Domain at center (no dependencies)
- Application orchestrates (use cases)
- Infrastructure implements interfaces

### CQRS
- **Command**: Write side, rich domain model
- **Query**: Read side, optimized DTOs
- Use MediatR: `IRequest<T>` + `IRequestHandler<T>`

### DDD Basics
- **Entity**: Has identity (Order)
- **Value Object**: Immutable, no identity (Money, Address)
- **Aggregate**: Entity cluster with one root
- **Bounded Context**: Clear domain boundaries

---

## DevOps Technologies

### Docker
- **Image**: Template (class)
- **Container**: Running instance (object)
- **Multi-stage build**: SDK image → Runtime image (smaller)
- **Commands**: `docker build`, `docker run -p 8080:80`

### Kubernetes
- **Pod**: One or more containers
- **Deployment**: Manages pod replicas, rolling updates
- **Service**: Stable endpoint for pods
- **Commands**: `kubectl get pods`, `kubectl apply -f`

### Redis
- **What**: In-memory data store
- **Use**: Caching, sessions, queues, rate limiting
- **Types**: String, Hash, List, Set, Sorted Set
- **C#**: `StackExchange.Redis`

### Elasticsearch
- **What**: Search and analytics engine
- **Use**: Full-text search, log analysis
- **ELK**: Elasticsearch + Logstash + Kibana

---

## Common Scenario Answers

### "Design a payment system"
1. **Idempotency**: Unique key per request, return cached result
2. **Circuit breaker**: Polly for external calls
3. **SAGA pattern**: Compensating transactions for failures
4. **Outbox pattern**: Ensure message delivery

### "Diagnose production performance issue"
1. Check metrics (CPU, memory, response time)
2. Look at logs with correlation ID
3. Common causes: N+1 queries, connection pool, memory pressure
4. Use profiling tools (dotTrace, Application Insights)

### "Migrate monolith to microservices"
1. Don't start unless necessary
2. Use Strangler Fig pattern
3. Start with least risky service
4. Database per service is hardest
5. Keep monolith running until confident

---

## LINQ Quick Reference

```csharp
// Filtering
.Where(x => condition)
.DistinctBy(x => x.Key)

// Projection
.Select(x => new { x.Id })
.SelectMany(x => x.Items)  // Flatten

// Aggregation
.Sum(x => x.Price)
.GroupBy(x => x.Category)
.Aggregate(0, (acc, x) => acc + x)

// Ordering
.OrderBy(x => x.Name)
.ThenByDescending(x => x.Date)

// Element
.First() / .FirstOrDefault()
.Single() / .SingleOrDefault()

// Quantifiers
.Any(x => condition)
.All(x => condition)

// Partitioning
.Take(10).Skip(5)
.Chunk(100)  // .NET 6+
```

---

## Interview Quick Tips

### Before Answer
- Clarify requirements
- Ask about constraints
- Think aloud

### Good Patterns to Mention
- "I'd start by understanding the requirements..."
- "The key considerations are..."
- "The tradeoffs to consider..."
- "In my experience..."

### Red Flags to Avoid
- Jumping to code without understanding
- Not mentioning tradeoffs
- Over-engineering simple problems
- Not asking questions

---

## Last Minute Reminders

**OOP**: Encapsulation (hide), Inheritance (is-a), Abstraction (show interface), Polymorphism (many forms)

**SOLID**: Single responsibility, Open/closed, Liskov substitution, Interface segregation, Dependency inversion

**Testing**: Arrange-Act-Assert, Mock dependencies, Test behavior not implementation

**Patterns**: Know Singleton, Factory, Strategy, Repository at minimum

**Architecture**: Dependencies inward, Domain has no dependencies, Depend on abstractions

**Fintech Focus**: Idempotency, Transaction integrity, Audit trails, High availability

---

## Files in This Guide

| File | Priority | Topics |
|------|----------|--------|
| 01-OOP-Principles | HIGH | 4 pillars with C# examples |
| 02-Engineering-Principles-SOLID | HIGH | SOLID, DRY, KISS, YAGNI |
| 03-Unit-Testing | HIGH | Why test, xUnit, Moq |
| 04-Design-Patterns | HIGH | All patterns with examples |
| 05-Real-World-Scenarios | HIGH | Architecture & problem-solving |
| 06-DotNet-Current | MEDIUM | .NET 8/9, C# 12/13 |
| 07-Clean-Code | MEDIUM | Naming, methods, code smells |
| 08-DevOps-Technologies | MEDIUM | Docker, K8s, Redis, Elasticsearch |
| 09-Architecture-Concepts | HIGH | Microservices, Clean Architecture, DDD |
| 10-Coding-Practice | MEDIUM | LINQ, algorithms, practice |

---

**Good luck with your interview at Multinet Up!** 🚀
