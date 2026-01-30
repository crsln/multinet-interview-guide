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

## Behavioral & HR Questions

### STAR Method
- **S**ituation - Context and background
- **T**ask - Your specific responsibility
- **A**ction - Steps YOU took
- **R**esult - Outcome with metrics

### Key Tips
- Keep answers to 2-3 minutes
- Focus on YOUR contributions ("I" not "we")
- Quantify results when possible
- Prepare 5-7 versatile stories

### "Tell Me About Yourself" Structure
```
Present: Current role and key achievements
Past: Relevant experience that built your skills
Future: Why this company/role excites you
```

---

## Files in This Guide

### Technical Q&A Files

| File | Priority | Topics |
|------|----------|--------|
| 01-OOP-Principles-QA | HIGH | 4 pillars with C# examples |
| 02-SOLID-Principles-QA | HIGH | SOLID with code examples |
| 03-Unit-Testing-QA | HIGH | Why test, xUnit, Moq |
| 04-Design-Patterns-QA | HIGH | All patterns with examples |
| 05-Scenarios-QA | HIGH | Real-world problem-solving (8 questions) |
| 06-DotNet-Current-QA | MEDIUM | .NET 8/9, C# 12/13 (8 questions) |
| 07-Clean-Code-QA | MEDIUM | Naming, methods, code smells |
| 08-DevOps-QA | MEDIUM | Docker, K8s, Redis, Elasticsearch |
| 09-Architecture-QA | HIGH | Microservices, DDD, patterns (8 questions) |
| 10-Coding-Practice-QA | MEDIUM | LINQ, algorithms, practice |

### New Files (Phase 2)

| File | Priority | Topics |
|------|----------|--------|
| 11-Behavioral-Questions-QA | HIGH | 8 STAR method questions for senior devs |
| 12-System-Design-QA | HIGH | 7 system design questions with diagrams |
| 13-Fintech-Domain-QA | HIGH | Payments, PCI-DSS, 3D Secure, fraud |
| 14-HR-Interview-QA | HIGH | Tell me about yourself, salary, weaknesses |
| 15-Your-Experience-Stories | HIGH | Personalized STAR templates for your CV |

---

## Pre-Interview Checklist

### Day Before
- [ ] Review cheatsheet and priority files
- [ ] Prepare 5-7 STAR stories
- [ ] Research Multinet Up (products, scale, recent news)
- [ ] Know your salary expectations (range, not single number)
- [ ] Have 3-5 questions to ask them

### Day Of
- [ ] Review "Tell me about yourself" script
- [ ] Check tech setup (if remote)
- [ ] Have copies of CV (if in-person)
- [ ] Arrive 10 minutes early

### Questions to Ask Them
1. "What does success look like in this role after 6 months?"
2. "What are the biggest technical challenges the team faces?"
3. "How does the engineering team collaborate with product?"
4. "What's the career progression path for this role?"

---

**Good luck with your interview at Multinet Up!**
