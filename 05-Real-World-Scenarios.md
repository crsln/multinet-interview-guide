# Real-World Scenario Questions

> ⚠️ **HIGH PRIORITY**: For 11 years experience, expect architecture and design questions

---

## Architecture & Design Scenarios

---

### "How would you design a high-availability payment processing system?"

#### Context
This is a classic fintech architecture question. They want to see if you understand:
- Distributed systems concepts
- Fault tolerance and reliability
- Transaction integrity
- Scalability

#### Approach Framework

**1. Clarify Requirements**
- What's the expected transaction volume? (TPS - transactions per second)
- What's the acceptable latency? (sub-second typically)
- What's the SLA for uptime? (99.9%, 99.99%?)
- Is it real-time or batch processing?

**2. High-Level Architecture**

```
                    ┌─────────────┐
                    │ Load Balancer│
                    │ (Active-Active)│
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
      ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
      │   API     │  │   API     │  │   API     │
      │ Gateway 1 │  │ Gateway 2 │  │ Gateway 3 │
      └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
            │              │              │
            └──────────────┼──────────────┘
                           │
                    ┌──────▼──────┐
                    │  Message    │
                    │  Queue      │
                    │  (Kafka)    │
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
      ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
      │ Payment   │  │ Payment   │  │ Payment   │
      │ Processor1│  │ Processor2│  │ Processor3│
      └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
            │              │              │
            └──────────────┼──────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────▼─────┐    ┌─────▼─────┐    ┌─────▼─────┐
    │Primary DB │    │ Secondary │    │  Read     │
    │ (Write)   │────│ (Standby) │    │ Replicas  │
    └───────────┘    └───────────┘    └───────────┘
```

**3. Key Points to Mention**

- **Idempotency**: Every payment request has a unique idempotency key to prevent double-charging
- **Circuit Breakers**: Polly for external payment gateway calls
- **Event Sourcing**: Store all payment events for audit trail
- **Eventual Consistency**: Accept that some operations will be eventually consistent
- **SAGA Pattern**: For distributed transactions across services
- **Dead Letter Queues**: For failed message processing
- **Health Checks**: Active monitoring and auto-healing

**4. Sample Code - Idempotent Payment Processing**

```csharp
public class PaymentService
{
    private readonly IDistributedCache _cache;
    private readonly IPaymentRepository _repository;
    private readonly IPaymentGateway _gateway;

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Check idempotency - prevent duplicate processing
        var existingResult = await _cache.GetAsync<PaymentResult>(request.IdempotencyKey);
        if (existingResult != null)
        {
            return existingResult; // Return cached result for duplicate request
        }

        // Acquire distributed lock
        await using var lockHandle = await _distributedLock.AcquireAsync(
            $"payment:{request.IdempotencyKey}",
            TimeSpan.FromSeconds(30));

        if (lockHandle == null)
            throw new ConcurrencyException("Could not acquire lock");

        // Double-check after acquiring lock
        existingResult = await _repository.GetByIdempotencyKeyAsync(request.IdempotencyKey);
        if (existingResult != null)
            return existingResult;

        // Process with circuit breaker
        var result = await _circuitBreakerPolicy.ExecuteAsync(async () =>
        {
            return await _gateway.ProcessAsync(request);
        });

        // Store result for idempotency
        await _repository.SaveAsync(result);
        await _cache.SetAsync(request.IdempotencyKey, result, TimeSpan.FromHours(24));

        // Publish event for other services
        await _eventBus.PublishAsync(new PaymentProcessedEvent(result));

        return result;
    }
}
```

---

### "We need to improve logging across our microservices - how would you approach this?"

#### Approach Framework

**1. Understand Current State**
- What logging exists today?
- What problems are you trying to solve? (debugging? compliance? monitoring?)
- What's the scale? (log volume, number of services)

**2. Propose Structured Logging with Correlation**

```csharp
// Install: Serilog.AspNetCore, Serilog.Sinks.Elasticsearch

public class Program
{
    public static void Main(string[] args)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Information()
            .Enrich.FromLogContext()
            .Enrich.WithProperty("Application", "PaymentService")
            .Enrich.WithProperty("Environment", Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT"))
            .WriteTo.Console(new JsonFormatter())
            .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://elasticsearch:9200"))
            {
                IndexFormat = "logs-{0:yyyy.MM.dd}",
                AutoRegisterTemplate = true
            })
            .CreateLogger();

        CreateHostBuilder(args).Build().Run();
    }
}
```

**3. Correlation ID Middleware**

```csharp
public class CorrelationIdMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-ID";
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context)
    {
        // Get or generate correlation ID
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        // Add to response headers
        context.Response.Headers[CorrelationIdHeader] = correlationId;

        // Add to logging context
        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}

// HTTP client that propagates correlation ID
public class CorrelationIdHandler : DelegatingHandler
{
    private readonly IHttpContextAccessor _contextAccessor;

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        var correlationId = _contextAccessor.HttpContext?
            .Request.Headers["X-Correlation-ID"].FirstOrDefault();

        if (!string.IsNullOrEmpty(correlationId))
        {
            request.Headers.Add("X-Correlation-ID", correlationId);
        }

        return await base.SendAsync(request, cancellationToken);
    }
}
```

**4. Key Points to Mention**
- **Structured logging** (JSON format for searchability)
- **Correlation IDs** to trace requests across services
- **Centralized aggregation** (ELK stack, Seq, or cloud-native like Azure App Insights)
- **Log levels** appropriately (not everything is ERROR)
- **Sensitive data** - don't log PII, card numbers, etc.
- **Performance** - async logging, sampling for high-volume services

---

### "How would you migrate a monolith to microservices?"

#### Approach Framework

**1. Don't Start with Microservices**
> "I'd first ask: should we migrate? Microservices add complexity. For some systems, a well-structured monolith is better."

**2. If Migration is Right, Use Strangler Fig Pattern**

```
Phase 1: Identify Boundaries
┌────────────────────────────────────┐
│           Monolith                 │
│  ┌──────┐ ┌──────┐ ┌──────────┐   │
│  │Orders│ │Users │ │Inventory │   │
│  └──────┘ └──────┘ └──────────┘   │
└────────────────────────────────────┘

Phase 2: Extract First Service (lowest risk)
┌────────────────────────────────────┐
│           Monolith                 │
│  ┌──────┐ ┌──────┐                │
│  │Orders│ │Users │ ──────────────────► New Inventory Service
│  └──────┘ └──────┘                │
└────────────────────────────────────┘

Phase 3: Gradually Extract More
┌────────────────────────────────────┐
│           Monolith                 │
│  ┌──────┐                         │
│  │Orders│──────────────────────────────► New User Service
│  └──────┘                         │
└────────────────────────────────────┘
                                          ▼
                                    New Inventory Service

Phase 4: Monolith Becomes a Service
       ┌──────────┐
       │Order Svc │
       └────┬─────┘
            │
     ┌──────┼──────┐
     ▼      ▼      ▼
┌──────┐ ┌──────┐ ┌──────┐
│User  │ │Invent│ │Notif │
│Svc   │ │Svc   │ │Svc   │
└──────┘ └──────┘ └──────┘
```

**3. Key Steps**

1. **Define bounded contexts** using DDD principles
2. **Start with data** - separate databases first
3. **Use Anti-Corruption Layer** between old and new
4. **Feature flags** for gradual rollout
5. **Parallel running** - keep monolith running until confident

**4. Key Points to Mention**
- Start with the **least risky, most independent** service
- **Database per service** is crucial but hardest
- **Event-driven communication** for loose coupling
- **Shared nothing** architecture goal
- **Monitor and observe** before cutting over completely

---

### "Design a distributed caching strategy for a high-traffic API"

#### Approach Framework

**1. Multi-Level Caching**

```
Request → In-Memory Cache (L1) → Distributed Cache (L2) → Database
               │                         │
          10-100μs                    1-10ms
```

**2. Implementation**

```csharp
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, TimeSpan? expiry = null);
    Task RemoveAsync(string key);
}

public class HybridCacheService : ICacheService
{
    private readonly IMemoryCache _memoryCache;
    private readonly IDistributedCache _distributedCache;
    private readonly TimeSpan _defaultL1Expiry = TimeSpan.FromMinutes(1);
    private readonly TimeSpan _defaultL2Expiry = TimeSpan.FromMinutes(15);

    public async Task<T?> GetAsync<T>(string key)
    {
        // L1: Check in-memory cache first (fastest)
        if (_memoryCache.TryGetValue(key, out T? cachedValue))
        {
            return cachedValue;
        }

        // L2: Check distributed cache
        var distributedValue = await _distributedCache.GetStringAsync(key);
        if (distributedValue != null)
        {
            var value = JsonSerializer.Deserialize<T>(distributedValue);

            // Populate L1 for future requests
            _memoryCache.Set(key, value, _defaultL1Expiry);

            return value;
        }

        return default;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null)
    {
        var serialized = JsonSerializer.Serialize(value);

        // Set in L1
        _memoryCache.Set(key, value, expiry ?? _defaultL1Expiry);

        // Set in L2
        await _distributedCache.SetStringAsync(key, serialized,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = expiry ?? _defaultL2Expiry
            });
    }

    public async Task RemoveAsync(string key)
    {
        _memoryCache.Remove(key);
        await _distributedCache.RemoveAsync(key);
    }
}
```

**3. Cache-Aside Pattern with Repository**

```csharp
public class CachedProductRepository : IProductRepository
{
    private readonly IProductRepository _innerRepository;
    private readonly ICacheService _cache;

    public async Task<Product?> GetByIdAsync(int id)
    {
        var cacheKey = $"product:{id}";

        // Try cache first
        var cached = await _cache.GetAsync<Product>(cacheKey);
        if (cached != null)
            return cached;

        // Cache miss - go to database
        var product = await _innerRepository.GetByIdAsync(id);

        if (product != null)
        {
            await _cache.SetAsync(cacheKey, product, TimeSpan.FromMinutes(10));
        }

        return product;
    }

    public async Task UpdateAsync(Product product)
    {
        await _innerRepository.UpdateAsync(product);

        // Invalidate cache
        await _cache.RemoveAsync($"product:{product.Id}");

        // Or update cache
        // await _cache.SetAsync($"product:{product.Id}", product);
    }
}
```

**4. Key Points to Mention**
- **Cache invalidation** is the hardest part
- **TTL strategy** - different for different data types
- **Cache stampede protection** - use locking for expensive operations
- **Monitoring** - hit rates, latency
- **Serialization** - be careful with large objects
- **Redis Cluster** for HA distributed cache

---

## Problem-Solving Scenarios

---

### "How would you diagnose a production performance issue that can't be reproduced locally?"

#### Approach Framework

**1. Gather Information**
- When did it start? Any deployments?
- Is it affecting all users or specific segments?
- Is it consistent or intermittent?

**2. Check Monitoring Dashboards**
```
Look for:
- CPU/Memory spikes
- Response time percentiles (p50, p95, p99)
- Error rates
- Database query times
- External service latencies
```

**3. Systematic Investigation**

```csharp
// Add detailed logging for investigation
public async Task<Order> GetOrderAsync(int id)
{
    using var activity = ActivitySource.StartActivity("GetOrder");
    activity?.SetTag("order.id", id);

    var sw = Stopwatch.StartNew();

    try
    {
        // Database call
        var order = await _repository.GetByIdAsync(id);

        activity?.SetTag("db.duration_ms", sw.ElapsedMilliseconds);

        if (sw.ElapsedMilliseconds > 1000)
        {
            _logger.LogWarning(
                "Slow order fetch: OrderId={OrderId}, Duration={Duration}ms",
                id, sw.ElapsedMilliseconds);
        }

        return order;
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        throw;
    }
}
```

**4. Common Causes**
- **N+1 queries** - check EF Core logging
- **Missing indexes** - database slow query log
- **Connection pool exhaustion** - monitor connection counts
- **Memory pressure** - GC pauses, LOH fragmentation
- **Thread pool starvation** - async/await issues
- **Network latency** to external services

**5. Key Points**
> "I'd start with metrics, not guesses. Look at the 4 golden signals: latency, traffic, errors, saturation. Then narrow down systematically."

---

### "A database query is slow in production but fast locally - what would you check?"

#### Checklist

| Check | Why |
|-------|-----|
| **Data volume** | Prod has millions of rows, local has thousands |
| **Query plan** | Use `EXPLAIN ANALYZE` to compare plans |
| **Index usage** | Index might be missing or not used in prod |
| **Parameter sniffing** | First execution cached a bad plan |
| **Statistics** | Outdated statistics leading to poor plan |
| **Locks/blocking** | Concurrent transactions blocking |
| **Resource contention** | CPU/memory/disk pressure |
| **Network latency** | Distance between app server and DB |

#### Investigation Steps

```sql
-- Check query plan in production
EXPLAIN ANALYZE
SELECT * FROM Orders WHERE CustomerId = @customerId;

-- Check for missing indexes
SELECT * FROM sys.dm_db_missing_index_details
WHERE database_id = DB_ID('YourDatabase');

-- Check for blocking
SELECT * FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;

-- Check statistics freshness
DBCC SHOW_STATISTICS ('Orders', 'IX_Orders_CustomerId');
```

---

### "How would you handle a situation where frontend devs are blocked waiting for your API?"

#### Approach

**1. Immediate Solutions**
- **API Contract First**: Define OpenAPI/Swagger spec together upfront
- **Mock Server**: Use tools like WireMock or Postman mock servers
- **Stub Endpoints**: Deploy endpoints that return hardcoded data

```csharp
// Quick stub endpoint
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    // TODO: Implement actual logic
    if (_environment.IsDevelopment())
    {
        return Ok(new OrderDto
        {
            Id = id,
            Status = "Pending",
            Total = 100.00m,
            CreatedAt = DateTime.UtcNow
        });
    }

    return await _orderService.GetOrderAsync(id);
}
```

**2. Long-term Solutions**
- **BFF Pattern** (Backend for Frontend) - frontend team owns their API
- **Contract Testing** with Pact
- **Parallel Development** with agreed interfaces

**3. Key Points**
> "Communication is key. Daily standups to identify blockers early. Define contracts upfront. Use feature flags so backend can merge incomplete work."

---

### "How would you approach refactoring legacy code with no tests?"

#### Approach Framework

**1. Michael Feathers' Legacy Code Change Algorithm**

```
1. Identify change points
2. Find test points
3. Break dependencies
4. Write tests
5. Make changes and refactor
```

**2. Start with Characterization Tests**

```csharp
// Characterization test - documents CURRENT behavior (even if wrong)
[Fact]
public void ProcessOrder_WhenCalled_ReturnsWhatItCurrentlyReturns()
{
    // This test captures current behavior, not intended behavior
    var legacy = new LegacyOrderProcessor();

    var result = legacy.ProcessOrder(new Order { Total = 100 });

    // Whatever this returns now, that's what we test for
    Assert.Equal("PROC-100-OK", result.Code);
}
```

**3. Sprout Method/Class Technique**

```csharp
// Original legacy code
public class LegacyOrderProcessor
{
    public void ProcessOrder(Order order)
    {
        // 500 lines of untested code...

        // NEW REQUIREMENT: Add tax calculation
        // Instead of modifying, SPROUT a new tested method
        var tax = CalculateTax(order);

        // ... rest of legacy code
    }

    // New, tested method
    public decimal CalculateTax(Order order)
    {
        return order.Total * 0.18m; // This is fully tested
    }
}
```

**4. Key Points**
> "Never refactor without tests. If no tests exist, write characterization tests first. Make small, incremental changes. Don't try to fix everything at once."

---

## Transaction & Data Integrity (Fintech-Relevant)

---

### "How do you ensure transaction consistency in distributed systems?"

#### Approach Framework

**1. SAGA Pattern**

```csharp
// Orchestration-based SAGA
public class OrderSaga
{
    private readonly IOrderService _orderService;
    private readonly IPaymentService _paymentService;
    private readonly IInventoryService _inventoryService;

    public async Task<OrderResult> ExecuteAsync(CreateOrderCommand command)
    {
        Order order = null;
        PaymentResult payment = null;

        try
        {
            // Step 1: Create order
            order = await _orderService.CreateOrderAsync(command);

            // Step 2: Reserve inventory
            await _inventoryService.ReserveAsync(order.Items);

            // Step 3: Process payment
            payment = await _paymentService.ProcessAsync(order);

            // Step 4: Confirm order
            await _orderService.ConfirmAsync(order.Id);

            return OrderResult.Success(order.Id);
        }
        catch (Exception ex)
        {
            // Compensating transactions (in reverse order)
            if (payment?.Success == true)
                await _paymentService.RefundAsync(payment.TransactionId);

            if (order != null)
                await _inventoryService.ReleaseAsync(order.Items);

            if (order != null)
                await _orderService.CancelAsync(order.Id);

            throw;
        }
    }
}
```

**2. Outbox Pattern for Reliable Messaging**

```csharp
public async Task CreateOrderAsync(Order order)
{
    using var transaction = await _context.Database.BeginTransactionAsync();

    try
    {
        // Save order
        _context.Orders.Add(order);

        // Save outbox message (same transaction)
        _context.OutboxMessages.Add(new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = "OrderCreated",
            Payload = JsonSerializer.Serialize(new OrderCreatedEvent(order)),
            CreatedAt = DateTime.UtcNow
        });

        await _context.SaveChangesAsync();
        await transaction.CommitAsync();

        // Background processor will publish the message
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

**3. Key Points**
- **Eventual consistency** is often acceptable
- **Idempotency** is crucial for retries
- **Compensating transactions** for rollback
- **Event sourcing** for complete audit trail

---

### "What patterns would you use for idempotent API operations?"

#### Implementation

```csharp
[ApiController]
public class PaymentsController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> ProcessPayment(
        [FromBody] PaymentRequest request,
        [FromHeader(Name = "Idempotency-Key")] string idempotencyKey)
    {
        if (string.IsNullOrEmpty(idempotencyKey))
            return BadRequest("Idempotency-Key header is required");

        // Check if we've processed this before
        var existingResult = await _cache.GetAsync<PaymentResponse>(
            $"idempotency:{idempotencyKey}");

        if (existingResult != null)
        {
            // Return cached response with 200 (not 201)
            Response.Headers["X-Idempotent-Replay"] = "true";
            return Ok(existingResult);
        }

        // Acquire distributed lock to prevent race conditions
        await using var lockHandle = await _lockProvider.AcquireAsync(
            $"payment-lock:{idempotencyKey}",
            TimeSpan.FromSeconds(30));

        if (lockHandle == null)
            return StatusCode(409, "Request already being processed");

        // Double-check after acquiring lock
        existingResult = await _cache.GetAsync<PaymentResponse>(
            $"idempotency:{idempotencyKey}");
        if (existingResult != null)
            return Ok(existingResult);

        // Process the payment
        var result = await _paymentService.ProcessAsync(request);

        // Store result for idempotency (24 hour TTL)
        await _cache.SetAsync(
            $"idempotency:{idempotencyKey}",
            result,
            TimeSpan.FromHours(24));

        return CreatedAtAction(nameof(GetPayment), new { id = result.Id }, result);
    }
}
```

**Key Points**
- Client generates idempotency key (usually UUID)
- Store result for sufficient time (24h+)
- Use distributed locking for concurrent requests
- Return same response for duplicate requests

---

### "How would you handle partial failures in a multi-step payment process?"

#### Approach

```csharp
public class PaymentOrchestrator
{
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var state = new PaymentState { Request = request };

        try
        {
            // Step 1: Validate
            state.ValidationResult = await ValidateAsync(request);
            if (!state.ValidationResult.IsValid)
                return PaymentResult.ValidationFailed(state.ValidationResult.Errors);

            // Step 2: Fraud check
            state.FraudResult = await CheckFraudAsync(request);
            if (state.FraudResult.IsSuspicious)
            {
                await SaveForReviewAsync(state);
                return PaymentResult.PendingReview();
            }

            // Step 3: Reserve funds
            state.ReservationResult = await ReserveFundsAsync(request);
            if (!state.ReservationResult.Success)
                return PaymentResult.InsufficientFunds();

            // Step 4: Charge (point of no return)
            state.ChargeResult = await ChargeAsync(request);
            if (!state.ChargeResult.Success)
            {
                // Release reservation
                await ReleaseReservationAsync(state.ReservationResult.ReservationId);
                return PaymentResult.ChargeFailed(state.ChargeResult.Error);
            }

            // Step 5: Update records
            await UpdateOrderStatusAsync(request.OrderId, OrderStatus.Paid);

            // Step 6: Send notification (non-critical)
            _ = SendNotificationAsync(request); // Fire and forget

            return PaymentResult.Success(state.ChargeResult.TransactionId);
        }
        catch (Exception ex)
        {
            await HandlePartialFailureAsync(state, ex);
            throw;
        }
    }

    private async Task HandlePartialFailureAsync(PaymentState state, Exception ex)
    {
        _logger.LogError(ex, "Payment failed at state: {State}", state);

        // Compensating actions based on how far we got
        if (state.ChargeResult?.Success == true)
        {
            // Charge succeeded - need manual intervention
            await AlertOpsTeamAsync(state, ex);
        }
        else if (state.ReservationResult?.Success == true)
        {
            // Release the reservation
            await ReleaseReservationAsync(state.ReservationResult.ReservationId);
        }

        // Log full state for debugging
        await SaveFailedPaymentStateAsync(state, ex);
    }
}
```

**Key Points**
- Identify the **point of no return** (where you can't easily rollback)
- Use **compensating transactions** for steps before that point
- **Alert operations** for failures after point of no return
- **Log everything** for debugging and reconciliation

---

### "Explain how you'd implement retry logic with exponential backoff"

```csharp
public class RetryService
{
    public async Task<T> ExecuteWithRetryAsync<T>(
        Func<Task<T>> operation,
        int maxRetries = 3,
        int baseDelayMs = 1000)
    {
        var exceptions = new List<Exception>();

        for (int attempt = 0; attempt <= maxRetries; attempt++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (IsTransient(ex) && attempt < maxRetries)
            {
                exceptions.Add(ex);

                // Exponential backoff with jitter
                var delay = CalculateDelay(attempt, baseDelayMs);

                _logger.LogWarning(
                    ex,
                    "Attempt {Attempt} failed. Retrying in {Delay}ms",
                    attempt + 1,
                    delay);

                await Task.Delay(delay);
            }
        }

        throw new AggregateException(
            $"Operation failed after {maxRetries + 1} attempts",
            exceptions);
    }

    private int CalculateDelay(int attempt, int baseDelayMs)
    {
        // Exponential: 1s, 2s, 4s, 8s...
        var exponentialDelay = baseDelayMs * (int)Math.Pow(2, attempt);

        // Add jitter (0-1000ms) to prevent thundering herd
        var jitter = Random.Shared.Next(0, 1000);

        // Cap at 30 seconds
        return Math.Min(exponentialDelay + jitter, 30000);
    }

    private bool IsTransient(Exception ex)
    {
        return ex is HttpRequestException
            || ex is TimeoutException
            || ex is SocketException
            || (ex is TaskCanceledException tce && !tce.CancellationToken.IsCancellationRequested);
    }
}

// Using Polly (recommended)
var policy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt =>
        {
            var exponential = TimeSpan.FromSeconds(Math.Pow(2, attempt));
            var jitter = TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000));
            return exponential + jitter;
        },
        onRetry: (exception, timeSpan, retryCount, context) =>
        {
            _logger.LogWarning(
                "Retry {RetryCount} after {Delay}s due to {Message}",
                retryCount, timeSpan.TotalSeconds, exception.Message);
        });
```

---

## Team & Process Scenarios

---

### "How do you approach code reviews?"

#### Framework

**1. What I Look For**
- **Correctness**: Does it do what it's supposed to?
- **Security**: SQL injection, XSS, auth issues?
- **Performance**: N+1 queries, unnecessary allocations?
- **Readability**: Can I understand it in 5 minutes?
- **Tests**: Are critical paths covered?
- **Edge cases**: Null checks, error handling?

**2. How I Give Feedback**
- Start with what's good
- Ask questions rather than demand changes
- Explain the "why" not just the "what"
- Offer suggestions, not prescriptions
- Pick battles - don't nitpick style if there's a linter

```markdown
✅ "Have you considered using a dictionary here for O(1) lookup?
   The current list iteration could be slow with large datasets."

❌ "This is wrong. Use a dictionary."
```

**3. Key Points**
> "Code reviews are about knowledge sharing, not gatekeeping. I aim to approve with suggestions when possible, rather than blocking for minor issues."

---

### "How would you mentor a junior developer struggling with a task?"

#### Approach

**1. Understand the Struggle**
- What specific part are they stuck on?
- Is it conceptual or implementation?
- Have they tried anything?

**2. Guide, Don't Solve**
```
Bad: "Here, let me write it for you"
Good: "What happens if you add a breakpoint here? What do you see?"
```

**3. Pair Programming When Appropriate**
- Take turns driving
- Think aloud to model problem-solving
- Let them do most of the typing

**4. Break Down the Problem**
- Help them decompose into smaller tasks
- Create a checklist they can follow
- Celebrate small wins

**5. Key Points**
> "My goal is to help them solve the next similar problem independently. I ask guiding questions, share relevant resources, and make myself available without taking over."

---

### "How do you balance technical debt against feature delivery?"

#### Framework

**1. Make Debt Visible**
- Track tech debt items in backlog
- Quantify impact when possible
- Link incidents to debt items

**2. Negotiate Budget**
- Propose 20% of sprint capacity for debt
- Tie debt reduction to business goals
- Show ROI: "Fixing this will reduce deployment time by 50%"

**3. Boy Scout Rule**
> "Leave the code better than you found it"
- Refactor while working on features
- Don't gold-plate, but improve what you touch

**4. Key Points**
> "I don't think of it as debt vs. features. Technical debt IS a feature - it affects velocity, reliability, and developer happiness. I advocate for sustainable pace and continuous improvement."

---

## Quick Scenario Response Templates

### For Architecture Questions
```
1. "First, I'd clarify requirements around [scale/availability/latency]..."
2. "The key considerations are [list 3-4 principles]..."
3. "A high-level approach would be [describe architecture]..."
4. "For implementation, I'd use [specific technologies/patterns]..."
5. "The tradeoffs to consider are [pros/cons]..."
```

### For Problem-Solving Questions
```
1. "I'd start by gathering data: [what metrics/logs to check]..."
2. "Common causes for this are [list possibilities]..."
3. "My debugging approach would be [systematic steps]..."
4. "To prevent this in future, I'd [preventive measures]..."
```

### For Process Questions
```
1. "In my experience, [share relevant story]..."
2. "The principles I follow are [list principles]..."
3. "A specific example was when [concrete example]..."
4. "The outcome was [measurable result]..."
```
