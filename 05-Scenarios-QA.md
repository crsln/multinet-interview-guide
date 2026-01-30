# Real-World Scenarios - Interview Questions

> 5 real-world scenario questions with detailed answers and communication tactics

---

## Question 1: Design an Idempotent Payment API

### The Question
> "Design an idempotent payment API. How would you ensure that a payment isn't processed twice even if the client retries the request?"

### Key Points to Cover
- What idempotency means and why it matters
- Idempotency key pattern
- Distributed locking
- Response caching

### Detailed Answer

**Why Idempotency Matters in Payments:**

```
Scenario without idempotency:
1. Client sends payment request
2. Server processes payment successfully
3. Network timeout before client receives response
4. Client retries (thinking it failed)
5. Server processes payment AGAIN
6. Customer charged TWICE! 💸

With idempotency:
1. Client sends payment request with Idempotency-Key: "abc123"
2. Server processes payment, stores result with key "abc123"
3. Network timeout before client receives response
4. Client retries with same Idempotency-Key: "abc123"
5. Server finds cached result for "abc123"
6. Returns cached result - customer charged ONCE ✅
```

**Implementation:**

```csharp
[ApiController]
[Route("api/payments")]
public class PaymentsController : ControllerBase
{
    private readonly IPaymentService _paymentService;
    private readonly IIdempotencyService _idempotencyService;
    private readonly IDistributedLock _distributedLock;

    [HttpPost]
    public async Task<IActionResult> ProcessPayment(
        [FromBody] PaymentRequest request,
        [FromHeader(Name = "Idempotency-Key")] string? idempotencyKey)
    {
        // 1. Require idempotency key
        if (string.IsNullOrEmpty(idempotencyKey))
        {
            return BadRequest(new ProblemDetails
            {
                Title = "Missing Idempotency Key",
                Detail = "Idempotency-Key header is required for payment requests"
            });
        }

        // 2. Check if we've already processed this request
        var cachedResult = await _idempotencyService.GetAsync<PaymentResponse>(idempotencyKey);
        if (cachedResult != null)
        {
            Response.Headers["X-Idempotent-Replay"] = "true";
            return Ok(cachedResult);
        }

        // 3. Acquire distributed lock to prevent concurrent processing
        await using var lockHandle = await _distributedLock.AcquireAsync(
            $"payment:{idempotencyKey}",
            timeout: TimeSpan.FromSeconds(30));

        if (lockHandle == null)
        {
            return Conflict(new ProblemDetails
            {
                Title = "Request In Progress",
                Detail = "This payment request is currently being processed"
            });
        }

        // 4. Double-check after acquiring lock (another request might have completed)
        cachedResult = await _idempotencyService.GetAsync<PaymentResponse>(idempotencyKey);
        if (cachedResult != null)
        {
            Response.Headers["X-Idempotent-Replay"] = "true";
            return Ok(cachedResult);
        }

        // 5. Process the payment
        var result = await _paymentService.ProcessAsync(request);

        // 6. Cache the result for future retries (24 hours)
        await _idempotencyService.SetAsync(
            idempotencyKey,
            result,
            TimeSpan.FromHours(24));

        return CreatedAtAction(
            nameof(GetPayment),
            new { id = result.TransactionId },
            result);
    }
}
```

**Idempotency Service Implementation:**

```csharp
public interface IIdempotencyService
{
    Task<T?> GetAsync<T>(string key) where T : class;
    Task SetAsync<T>(string key, T value, TimeSpan expiry) where T : class;
}

public class RedisIdempotencyService : IIdempotencyService
{
    private readonly IDatabase _redis;
    private const string Prefix = "idempotency:";

    public RedisIdempotencyService(IConnectionMultiplexer redis)
    {
        _redis = redis.GetDatabase();
    }

    public async Task<T?> GetAsync<T>(string key) where T : class
    {
        var value = await _redis.StringGetAsync(Prefix + key);
        if (value.IsNullOrEmpty)
            return null;

        return JsonSerializer.Deserialize<T>(value!);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan expiry) where T : class
    {
        var json = JsonSerializer.Serialize(value);
        await _redis.StringSetAsync(Prefix + key, json, expiry);
    }
}
```

**Distributed Lock with Redis:**

```csharp
public interface IDistributedLock
{
    Task<IAsyncDisposable?> AcquireAsync(string resource, TimeSpan timeout);
}

public class RedisDistributedLock : IDistributedLock
{
    private readonly IDatabase _redis;

    public async Task<IAsyncDisposable?> AcquireAsync(string resource, TimeSpan timeout)
    {
        var lockKey = $"lock:{resource}";
        var lockValue = Guid.NewGuid().ToString();

        // Try to acquire lock with SET NX (set if not exists)
        var acquired = await _redis.StringSetAsync(
            lockKey,
            lockValue,
            timeout,
            When.NotExists);

        if (!acquired)
            return null;

        return new LockHandle(_redis, lockKey, lockValue);
    }

    private class LockHandle : IAsyncDisposable
    {
        private readonly IDatabase _redis;
        private readonly string _key;
        private readonly string _value;

        public LockHandle(IDatabase redis, string key, string value)
        {
            _redis = redis;
            _key = key;
            _value = value;
        }

        public async ValueTask DisposeAsync()
        {
            // Only delete if we still own the lock
            var script = @"
                if redis.call('get', KEYS[1]) == ARGV[1] then
                    return redis.call('del', KEYS[1])
                else
                    return 0
                end";

            await _redis.ScriptEvaluateAsync(script,
                new RedisKey[] { _key },
                new RedisValue[] { _value });
        }
    }
}
```

**Key Design Points:**

| Aspect | Implementation |
|--------|----------------|
| Idempotency Key | Client-generated UUID, passed in header |
| Storage | Redis with 24-hour TTL |
| Concurrency | Distributed lock prevents race conditions |
| Response | Return cached response for duplicate requests |
| Header | X-Idempotent-Replay indicates cached response |

### Communication Tactics

🎯 **Structure your answer**: Explain the problem (double-charge), show the solution flow, then implement key components.

💡 **Emphasize**: This is CRITICAL for fintech. Mention distributed locking, cache TTL choices, and the importance of returning identical responses.

⚠️ **Avoid**: Don't forget the double-check after acquiring lock. Another request might have completed while waiting.

---

## Question 2: Debug Production Performance Issue

### The Question
> "How would you debug a production performance issue that you can't reproduce locally? Walk me through your systematic approach."

### Key Points to Cover
- Systematic debugging approach
- Metrics and observability
- Common production-only causes
- Tools and techniques

### Detailed Answer

**Systematic Approach - The 4 Golden Signals:**

```
1. LATENCY    - How long do requests take?
2. TRAFFIC    - How many requests per second?
3. ERRORS     - What's the error rate?
4. SATURATION - How "full" are resources?
```

**Step-by-Step Debugging Process:**

```csharp
// Step 1: Gather Information
// - When did it start? (correlate with deployments)
// - Is it affecting all users or specific segments?
// - Is it consistent or intermittent?
// - What changed recently?

// Step 2: Check Metrics Dashboard
public class MetricsToCheck
{
    // Application metrics
    public double RequestLatencyP50 { get; set; }
    public double RequestLatencyP95 { get; set; }
    public double RequestLatencyP99 { get; set; }
    public double ErrorRate { get; set; }
    public double RequestsPerSecond { get; set; }

    // Infrastructure metrics
    public double CpuUsage { get; set; }
    public double MemoryUsage { get; set; }
    public double DiskIO { get; set; }
    public double NetworkLatency { get; set; }

    // Database metrics
    public double QueryLatency { get; set; }
    public int ActiveConnections { get; set; }
    public int WaitingConnections { get; set; }

    // External service metrics
    public double ExternalApiLatency { get; set; }
    public double ExternalApiErrorRate { get; set; }
}
```

**Step 3: Add Targeted Logging/Tracing:**

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;
    private readonly ActivitySource _activitySource = new("OrderService");

    public async Task<Order> GetOrderAsync(int id)
    {
        using var activity = _activitySource.StartActivity("GetOrder");
        activity?.SetTag("order.id", id);

        var sw = Stopwatch.StartNew();

        try
        {
            // Log entry
            _logger.LogDebug("Getting order {OrderId}", id);

            // Track database call
            using (var dbActivity = _activitySource.StartActivity("Database.GetOrder"))
            {
                var order = await _repository.GetByIdAsync(id);

                dbActivity?.SetTag("db.rows_returned", order != null ? 1 : 0);

                // Flag slow queries
                if (sw.ElapsedMilliseconds > 100)
                {
                    _logger.LogWarning(
                        "Slow order fetch: OrderId={OrderId}, Duration={Duration}ms",
                        id, sw.ElapsedMilliseconds);
                }

                return order;
            }
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            _logger.LogError(ex, "Failed to get order {OrderId}", id);
            throw;
        }
    }
}
```

**Common Production-Only Causes:**

```csharp
// 1. N+1 QUERIES - Works fine with 10 rows, kills performance with 10,000
// BAD:
var orders = await _context.Orders.ToListAsync();
foreach (var order in orders)
{
    // This executes a query for EACH order!
    var customer = await _context.Customers.FindAsync(order.CustomerId);
}

// GOOD:
var orders = await _context.Orders
    .Include(o => o.Customer)  // Single query with JOIN
    .ToListAsync();

// 2. CONNECTION POOL EXHAUSTION
// Symptom: Requests hang waiting for database connection
// Check: Active vs max connections in pool
// Fix: Ensure proper async/await, increase pool size, add connection timeout

// 3. MEMORY PRESSURE / GC PAUSES
// Symptom: Periodic slowdowns, high memory usage
// Check: GC pause times, LOH fragmentation
// Fix: Object pooling, avoid large allocations, use ArrayPool<T>

// 4. THREAD POOL STARVATION
// Symptom: Requests queue up, async not helping
// Check: ThreadPool.GetAvailableThreads()
// Fix: Ensure async all the way down, avoid .Result/.Wait()

// 5. EXTERNAL SERVICE LATENCY
// Symptom: Slow when calling external API
// Check: Distributed trace shows external call taking long
// Fix: Add timeouts, circuit breaker, caching
```

**Diagnostic Queries:**

```sql
-- SQL Server: Find slow queries
SELECT TOP 10
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_elapsed_time DESC;

-- Check for blocking
SELECT * FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;

-- Check connection pool
SELECT
    DB_NAME(dbid) AS DatabaseName,
    COUNT(*) AS NumberOfConnections,
    loginame AS LoginName
FROM sys.sysprocesses
GROUP BY dbid, loginame;
```

**Tools to Use:**

| Tool | Purpose |
|------|---------|
| Application Insights / Datadog | APM, distributed tracing |
| Seq / ELK | Centralized logging |
| dotTrace / dotMemory | Profiling |
| PerfView | .NET performance analysis |
| SQL Profiler | Database query analysis |

### Communication Tactics

🎯 **Structure your answer**: Start with the 4 Golden Signals framework, then walk through the systematic process, mention common causes.

💡 **Emphasize**: "I don't guess - I use metrics. I check latency percentiles, error rates, and correlate with deployments."

⚠️ **Avoid**: Don't suggest random fixes. Show a methodical, data-driven approach.

---

## Question 3: Monolith to Microservices Migration

### The Question
> "Walk me through how you would migrate a monolith to microservices. What's your strategy and what are the key considerations?"

### Key Points to Cover
- When to migrate (and when not to)
- Strangler Fig pattern
- Service boundary identification
- Database decomposition challenges

### Detailed Answer

**First Question: Should We Migrate?**

```
DON'T migrate if:
- Team is small (<10 developers)
- Domain isn't well understood yet
- Current monolith isn't causing real problems
- You're doing it just because it's "modern"

DO migrate when:
- Different parts need different scaling
- Teams are stepping on each other
- Deployment of one feature breaks another
- Parts have clearly different release cycles
```

**Migration Strategy - Strangler Fig Pattern:**

```
Phase 1: Identify Boundaries
┌────────────────────────────────────┐
│           Monolith                 │
│  ┌──────┐ ┌──────┐ ┌──────────┐   │
│  │Orders│ │Users │ │Inventory │   │
│  └──────┘ └──────┘ └──────────┘   │
└────────────────────────────────────┘

Phase 2: Add API Gateway / Facade
┌─────────────────┐
│   API Gateway   │
└────────┬────────┘
         │
┌────────▼────────────────────────────┐
│           Monolith                  │
│  ┌──────┐ ┌──────┐ ┌──────────┐    │
│  │Orders│ │Users │ │Inventory │    │
│  └──────┘ └──────┘ └──────────┘    │
└─────────────────────────────────────┘

Phase 3: Extract First Service (lowest risk)
┌─────────────────┐
│   API Gateway   │
└───────┬─────────┘
        │
   ┌────┴────┐
   │         │
   ▼         ▼
┌──────┐  ┌────────────────────────┐
│Notif │  │      Monolith          │
│ Svc  │  │  ┌──────┐ ┌──────┐    │
└──────┘  │  │Orders│ │Users │    │
          │  └──────┘ └──────┘    │
          └────────────────────────┘

Phase 4: Gradually Extract More
┌─────────────────┐
│   API Gateway   │
└───────┬─────────┘
        │
   ┌────┼────┬────┐
   ▼    ▼    ▼    ▼
┌────┐┌────┐┌────┐┌────────┐
│Notf││Inv ││User││ Orders │
│Svc ││Svc ││Svc ││(Mono)  │
└────┘└────┘└────┘└────────┘
```

**Implementation Steps:**

```csharp
// Step 1: Add Anti-Corruption Layer in Monolith
public interface IInventoryService
{
    Task<int> GetStockAsync(int productId);
    Task<bool> ReserveAsync(int productId, int quantity);
}

// Initially, this calls the monolith's internal code
public class MonolithInventoryService : IInventoryService
{
    private readonly InventoryRepository _repo;

    public async Task<int> GetStockAsync(int productId)
    {
        return await _repo.GetStockAsync(productId);
    }
}

// Step 2: Create the Microservice
[ApiController]
[Route("api/inventory")]
public class InventoryController : ControllerBase
{
    [HttpGet("{productId}/stock")]
    public async Task<int> GetStock(int productId)
    {
        return await _service.GetStockAsync(productId);
    }
}

// Step 3: Switch to calling the microservice
public class HttpInventoryService : IInventoryService
{
    private readonly HttpClient _client;

    public async Task<int> GetStockAsync(int productId)
    {
        var response = await _client.GetAsync($"/api/inventory/{productId}/stock");
        return await response.Content.ReadFromJsonAsync<int>();
    }
}

// Step 4: Use feature flag to switch between implementations
services.AddScoped<IInventoryService>(sp =>
{
    var featureFlags = sp.GetRequiredService<IFeatureFlags>();

    return featureFlags.IsEnabled("UseInventoryMicroservice")
        ? sp.GetRequiredService<HttpInventoryService>()
        : sp.GetRequiredService<MonolithInventoryService>();
});
```

**Database Decomposition (The Hardest Part):**

```csharp
// Phase 1: Shared Database (temporary)
// Both monolith and new service read/write same tables
// Risk: Coupling, but allows gradual migration

// Phase 2: Database View / Sync
// New service has own database
// Sync data from monolith using:
// - Change Data Capture (CDC)
// - Event publishing
// - Dual writes

// Phase 3: Full Separation
// Each service owns its data
// Communication via APIs or events

public class OrderService
{
    // Don't call inventory database directly!
    // Call the inventory service
    public async Task<bool> CanFulfillOrderAsync(Order order)
    {
        foreach (var item in order.Items)
        {
            var stock = await _inventoryService.GetStockAsync(item.ProductId);
            if (stock < item.Quantity)
                return false;
        }
        return true;
    }
}
```

**Key Considerations:**

| Aspect | Consideration |
|--------|---------------|
| Order | Start with least risky, most independent service |
| Data | Database separation is the hardest - plan carefully |
| Communication | Prefer async events over sync calls |
| Monitoring | Distributed tracing is essential |
| Testing | Need integration tests across services |
| Rollback | Keep monolith running until confident |

### Communication Tactics

🎯 **Structure your answer**: Start with "should we migrate?", then show Strangler Fig pattern, discuss database challenges.

💡 **Emphasize**: "I'd start with a well-structured monolith. Microservices add complexity. I'd only extract when there's a clear bounded context and scaling need."

⚠️ **Avoid**: Don't be evangelical about microservices. Show you understand the trade-offs and complexity they add.

---

## Question 4: Technical Debt vs Feature Delivery

### The Question
> "How do you balance technical debt against feature delivery? How would you convince stakeholders to allocate time for debt reduction?"

### Key Points to Cover
- Making debt visible
- Quantifying impact
- Practical allocation strategies
- Stakeholder communication

### Detailed Answer

**Making Technical Debt Visible:**

```csharp
// Track debt in your backlog with impact metrics
public class TechnicalDebtItem
{
    public string Id { get; set; }
    public string Description { get; set; }
    public string AffectedArea { get; set; }

    // Quantifiable impact
    public int IncidentsLastQuarter { get; set; }
    public TimeSpan AverageIncidentResolutionTime { get; set; }
    public int DeveloperHoursLostPerMonth { get; set; }

    // Estimate to fix
    public int EstimatedHoursToFix { get; set; }

    // Calculate ROI
    public double MonthlyTimeSaved =>
        DeveloperHoursLostPerMonth;

    public double PaybackPeriodMonths =>
        EstimatedHoursToFix / (double)DeveloperHoursLostPerMonth;
}
```

**Quantifying Impact - Speak Business Language:**

```
❌ "We need to refactor the payment module"

✅ "The payment module has caused 3 production incidents in the last month,
   each taking 4 hours to resolve. That's 12 hours of developer time plus
   estimated $50K in lost transactions. A 20-hour refactoring effort would
   prevent these issues and pay for itself in under 2 months."
```

**Practical Strategies:**

```
Strategy 1: The "Tax" Approach
- Allocate 20% of each sprint to tech debt
- Non-negotiable, built into velocity
- "Every sprint, we spend 2 days on improvements"

Strategy 2: The "Boy Scout Rule"
- Leave code better than you found it
- Refactor while working on features
- Small improvements compound over time

Strategy 3: Tech Debt Sprints
- Every 4th sprint is dedicated to improvements
- Larger refactoring efforts
- Works for bigger architectural changes

Strategy 4: Incident-Driven Prioritization
- Track incidents caused by tech debt
- Prioritize debt that causes real problems
- "This debt caused 3 outages - it's now P1"
```

**Framework for Stakeholder Conversation:**

```
1. SHOW THE PROBLEM
   "Deployments take 4 hours because of our test suite.
   That's 20 developer-hours per week."

2. QUANTIFY THE COST
   "At $100/hour, that's $8,000/month in deployment overhead."

3. PROPOSE THE SOLUTION
   "A 40-hour investment in test parallelization would cut
   deploy time to 30 minutes."

4. CALCULATE ROI
   "Investment: $4,000 (40 hours)
   Monthly savings: $7,000
   Payback period: 3 weeks"

5. SHOW THE RISK OF INACTION
   "Without this, our deploy frequency will drop as the
   codebase grows, slowing feature delivery."
```

**My Personal Approach:**

```csharp
// I follow these principles:

// 1. Never ask for "tech debt time" in isolation
// Instead: "This feature will take 5 days. 4 for the feature,
// 1 to clean up the area we're touching."

// 2. Make debt part of feature estimates
public class FeatureEstimate
{
    public string Feature { get; set; }
    public int FeatureHours { get; set; }
    public int RefactoringHours { get; set; }  // Built in!
    public int TotalHours => FeatureHours + RefactoringHours;
}

// 3. Link debt to incidents
// Every post-mortem asks: "What tech debt contributed to this?"
// Creates natural prioritization

// 4. Track velocity over time
// If velocity drops, it's often debt slowing us down
// Data speaks louder than opinions
```

### Communication Tactics

🎯 **Structure your answer**: Show you make debt visible, quantify it in business terms, propose practical strategies.

💡 **Emphasize**: Speak business language - hours, dollars, incidents, risk. "Tech debt isn't a developer convenience issue - it's a business risk."

⚠️ **Avoid**: Don't complain about stakeholders not understanding. Show you know how to communicate effectively.

---

## Question 5: Code Review Approach

### The Question
> "Describe your ideal code review process. What do you look for and how do you give feedback?"

### Key Points to Cover
- What to review
- How to give feedback constructively
- Automation vs human review
- Speed vs thoroughness trade-off

### Detailed Answer

**What I Look For (Priority Order):**

```
1. CORRECTNESS
   - Does it do what it's supposed to?
   - Are edge cases handled?
   - Any logical errors?

2. SECURITY
   - SQL injection risks?
   - XSS vulnerabilities?
   - Proper authentication/authorization?
   - Secrets exposed?

3. PERFORMANCE
   - N+1 queries?
   - Unnecessary allocations?
   - Missing indexes?

4. MAINTAINABILITY
   - Is it readable?
   - Are names clear?
   - Is complexity appropriate?

5. TESTS
   - Are critical paths tested?
   - Are tests meaningful (not just coverage)?
```

**How I Give Feedback:**

```markdown
## ❌ BAD Feedback
"This is wrong. Use a dictionary."
"Why didn't you use LINQ here?"
"This code is messy."

## ✅ GOOD Feedback

### Questions Instead of Commands
"Have you considered using a dictionary here for O(1) lookup?
The current list iteration could be slow with large datasets."

### Explain the Why
"I'd suggest extracting this into a method because it appears
in 3 places. If the logic changes, we'd have to update all 3."

### Offer Alternatives
"This works! Another approach might be using `FirstOrDefault()`
instead of `Where().First()` - slightly more concise."

### Acknowledge Good Work
"Nice use of the null-coalescing operator here!"
"Good thinking on adding that index - this query would be slow without it."
```

**My Review Categories:**

```csharp
// I use prefixes to indicate severity

// 🔴 BLOCKER - Must fix before merge
// "🔴 This SQL query is vulnerable to injection.
// We need to use parameterized queries."

// 🟡 SUGGESTION - Should consider, but not blocking
// "🟡 This could be simplified using pattern matching.
// Optional improvement."

// 🟢 NIT - Minor style preference
// "🟢 Nit: I'd put the opening brace on a new line for consistency
// with the rest of the codebase."

// 💭 QUESTION - Genuinely curious, not a change request
// "💭 I'm curious why you chose this approach over X?
// Not a change request, just learning."
```

**Automation vs Human Review:**

```yaml
# Automate what you can
automated:
  - Formatting (dotnet format, Prettier)
  - Linting (StyleCop, ESLint)
  - Security scanning (Snyk, CodeQL)
  - Test coverage thresholds
  - Build success

# Humans focus on
human_review:
  - Business logic correctness
  - Architecture decisions
  - Code clarity and naming
  - Algorithm efficiency
  - Knowledge sharing
```

**Speed vs Thoroughness:**

```
My guidelines:
- Review within 24 hours (don't block teammates)
- First pass: 15-30 minutes for most PRs
- Second pass if needed for complex changes
- Approve with suggestions when possible
  (don't block for minor issues)
- Use "Approve with comments" liberally
```

**The Review as Learning:**

```csharp
// Code reviews are for:
// 1. Catching bugs (obvious)
// 2. Knowledge sharing (less obvious but equally important)

// I learn something from every review I do
// I teach something in every review I give

// Example comment that teaches:
// "This works! FYI, in C# 12 you could also write this as:
//
// public class UserService(IUserRepo repo) { }
//
// Primary constructors - no need for field declaration.
// Not a change request, just sharing a new feature!"
```

### Communication Tactics

🎯 **Structure your answer**: Cover what you look for, how you give feedback, the automation balance, and the learning aspect.

💡 **Emphasize**: Reviews are about knowledge sharing, not gatekeeping. Mention that you approve with suggestions rather than blocking for minor issues.

⚠️ **Avoid**: Don't sound like a harsh reviewer. Balance thoroughness with being a good team player.

---

## Question 6: Handling Changing Requirements Mid-Sprint

### The Question
> "How do you handle a situation where requirements keep changing mid-sprint?"

### Key Points to Cover
- Process for managing change
- Stakeholder communication
- Protecting team focus
- Trade-off discussions

### Detailed Answer

**My Approach:**

```
1. ACKNOWLEDGE THE REALITY
   - Requirements DO change - that's normal
   - The question is HOW we handle it, not IF

2. ESTABLISH A PROCESS
   - All changes go through Product Owner (single point)
   - Impact assessment before accepting
   - Explicit trade-off discussion
   - Sprint scope renegotiation if needed

3. PROTECT THE TEAM
   - Shield developers from constant context switching
   - Batch small changes for discussion
   - Reserve capacity for "interrupt" work (10-15%)
```

**Framework for Change Requests:**

```csharp
// Decision matrix for mid-sprint changes
public enum ChangeResponse
{
    // Critical bug or security issue
    AcceptImmediately,    // Drop current work, fix now

    // Important but not urgent
    SwapForExistingWork,  // "We can do this IF we drop X"

    // Nice to have
    AddToBacklog,         // "Great idea, let's plan for next sprint"

    // Scope creep
    PushBack              // "This changes the goal, needs PM discussion"
}
```

**Sample Response:**

> "When requirements change mid-sprint, I first assess the impact - is this a critical fix or a nice-to-have? For critical issues, we adapt immediately.
>
> For other changes, I facilitate a trade-off discussion: 'We can add feature X, but we'll need to drop or defer feature Y.' This makes the cost visible.
>
> I also try to protect the team from constant interrupts. Small changes get batched for discussion. I reserve some sprint capacity for unplanned work so we're not constantly failing commitments.
>
> The key is transparent communication - stakeholders understand the trade-offs, and the team has predictable work."

### Communication Tactics

- **Emphasize**: "I manage change through process, not resistance"
- **Show pragmatism**: Some changes are urgent, others can wait
- **Mention trade-offs**: Make the cost visible

---

## Question 7: Application Under Heavy Load - Incident Response

### The Question
> "Your application is under heavy load and starting to fail - walk me through your response."

### Key Points to Cover
- Incident response process
- Immediate vs. long-term actions
- Communication during incidents
- Post-incident learning

### Detailed Answer

**Incident Response Framework:**

```
PHASE 1: STABILIZE (First 5 minutes)
────────────────────────────────────
Goal: Stop the bleeding, buy time

Actions:
1. Acknowledge the incident (set status page, notify stakeholders)
2. Quick assessment: What's failing? What percentage of users?
3. Immediate mitigation options:
   - Scale up (add more instances)
   - Shed load (enable circuit breakers, rate limiting)
   - Disable non-critical features
   - Rollback if recent deployment

PHASE 2: DIAGNOSE (5-30 minutes)
────────────────────────────────
Goal: Find the root cause

Actions:
1. Check the 4 Golden Signals: Latency, Traffic, Errors, Saturation
2. Correlate with recent changes (deployment, config, traffic spike)
3. Check dependencies (database, external services, cache)
4. Look at resource utilization (CPU, memory, connections)

PHASE 3: RESOLVE (30+ minutes)
──────────────────────────────
Goal: Fix the underlying issue

Actions:
1. Implement targeted fix based on diagnosis
2. Gradual rollout (don't go from 0 to 100 instantly)
3. Verify fix with metrics
4. Stand down when stable for 15+ minutes
```

**Common Causes & Quick Fixes:**

```csharp
// Connection pool exhaustion
// Symptom: Requests hanging, "no available connection" errors
// Quick fix: Increase pool size, check for connection leaks
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sql =>
        sql.MaxPoolSize(200)));  // Increase from default 100

// Memory pressure
// Symptom: GC pauses, slow responses, OOM errors
// Quick fix: Restart instances (buys time), scale horizontally

// Database locks
// Symptom: Transactions timing out, specific operations slow
// Quick fix: Kill blocking queries, review transaction isolation

// External service slowdown
// Symptom: Requests to one service taking too long
// Quick fix: Enable circuit breaker, serve cached/default responses
```

**Post-Incident:**

```
Within 24 hours:
- Document timeline of events
- Identify root cause and contributing factors
- Create action items for permanent fix

Within 1 week:
- Post-mortem meeting (blameless)
- Update runbooks and monitoring
- Implement preventive measures
```

### Communication Tactics

- **Show structure**: Have a framework, don't panic
- **Emphasize stability first**: Don't debug while burning
- **Mention post-mortem**: Learning from incidents is key

---

## Question 8: Estimating Complex Features

### The Question
> "How do you approach estimating a complex feature you've never built before?"

### Key Points to Cover
- Estimation techniques
- Handling uncertainty
- Communication with stakeholders
- Iterative refinement

### Detailed Answer

**Estimation Framework:**

```
1. BREAK DOWN THE UNKNOWN
   - Identify what you know vs. don't know
   - List assumptions that affect the estimate
   - Find comparable past work (even if not identical)

2. USE RANGES, NOT POINTS
   - Best case: Everything goes smoothly
   - Expected: Normal friction and surprises
   - Worst case: Significant unknowns revealed

3. SPIKE FIRST IF POSSIBLE
   - Invest 1-2 days in a proof of concept
   - Validate key assumptions
   - Re-estimate with better information

4. COMMUNICATE UNCERTAINTY
   - "I estimate 2-3 weeks, but I'd like to spike the X integration first"
   - "Based on our Y experience, plus 50% for unknowns"
```

**Practical Techniques:**

```csharp
// Technique 1: Decomposition
// Break feature into smallest estimable pieces
public class FeatureEstimate
{
    public string Feature { get; set; } = "Payment Gateway Integration";

    public List<Task> Tasks { get; set; } = new()
    {
        new Task("Research gateway API", hours: 4, uncertainty: Uncertainty.Low),
        new Task("Design data model", hours: 8, uncertainty: Uncertainty.Medium),
        new Task("Implement core flow", hours: 24, uncertainty: Uncertainty.Medium),
        new Task("Handle edge cases", hours: 16, uncertainty: Uncertainty.High),
        new Task("Testing & integration", hours: 16, uncertainty: Uncertainty.Medium),
        new Task("Documentation", hours: 4, uncertainty: Uncertainty.Low)
    };

    // Total: 72 hours base
    // With uncertainty buffer: 90-100 hours
    // = 2.5 - 3 weeks
}

// Technique 2: Reference class forecasting
// "Our last API integration took 3 weeks. This is similar complexity,
// but we don't know the external API well. Estimate: 3-4 weeks."

// Technique 3: Three-point estimation
// Optimistic + (4 × Most Likely) + Pessimistic / 6
// (2 weeks + 4×3 weeks + 5 weeks) / 6 = 3.2 weeks
```

**Communication Script:**

> "For this feature, I'd break it down like this: [list components]. Based on similar work we've done, I estimate [X] weeks with medium confidence.
>
> There are a few unknowns that could affect this: [list]. I'd suggest a 2-day spike on [key unknown] before committing to a deadline.
>
> I'd rather give a range of [X-Y weeks] than a single number. If we discover [risk], it could push toward the higher end."

### Communication Tactics

- **Never give a single number**: Ranges communicate uncertainty
- **Identify unknowns explicitly**: Shows you've thought it through
- **Suggest spikes**: Reduces risk before big commitments

---

## Quick Review - Key Takeaways

| Question | Key Point |
|----------|-----------|
| Idempotent Payments | Idempotency key + distributed lock + response cache |
| Debug Production | 4 Golden Signals, systematic approach, don't guess |
| Monolith Migration | Strangler Fig, start small, database is hardest |
| Tech Debt Balance | Quantify in business terms, build into estimates |
| Code Review | Knowledge sharing, approve with suggestions |
| Changing Requirements | Trade-off discussions, protect team focus |
| Heavy Load Incident | Stabilize first, diagnose second, learn after |
| Complex Estimation | Break down, use ranges, spike unknowns |
