# System Design Interview Questions

> 7 system design questions with structured approaches for senior software engineers

---

## How to Approach System Design Questions

### The Framework (5 Steps)

```
1. CLARIFY REQUIREMENTS (2-3 minutes)
   - Functional: What should the system do?
   - Non-functional: Scale, latency, availability
   - Constraints: Budget, timeline, existing systems

2. ESTIMATE SCALE (2-3 minutes)
   - Users, requests per second, data volume
   - Read vs write ratio
   - Peak vs average load

3. HIGH-LEVEL DESIGN (5-8 minutes)
   - Draw main components
   - Show data flow
   - Identify APIs

4. DEEP DIVE (10-15 minutes)
   - Database schema and choices
   - Caching strategy
   - Key algorithms
   - Handle bottlenecks

5. WRAP UP (2-3 minutes)
   - Address trade-offs
   - Discuss monitoring and operations
   - Mention what you'd improve with more time
```

---

## Question 1: Design a Payment Gateway System

### The Question
> "Design a payment gateway system like Stripe or Iyzico. How would you handle the core transaction flow?"

### Clarifying Questions to Ask
- What payment methods? (Card, bank transfer, digital wallet)
- Transaction volume? (1000/sec? 10,000/sec?)
- What regions/currencies?
- PCI-DSS compliance level needed?
- Real-time or batch settlement?

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              PAYMENT GATEWAY                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────┐     ┌──────────────┐     ┌─────────────────┐                │
│  │  Merchant  │────→│  API Gateway │────→│  Auth Service   │                │
│  │   (Client) │     │  (Kong/YARP) │     │  (OAuth/JWT)    │                │
│  └────────────┘     └──────┬───────┘     └─────────────────┘                │
│                            │                                                 │
│                            ▼                                                 │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                        Payment Orchestrator                           │   │
│  │  ┌──────────────┐  ┌───────────────┐  ┌────────────────────────┐    │   │
│  │  │ Idempotency  │  │    Routing    │  │   Saga Coordinator     │    │   │
│  │  │   Service    │  │    Engine     │  │   (Transaction Mgmt)   │    │   │
│  │  └──────────────┘  └───────────────┘  └────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                            │                                                 │
│           ┌────────────────┼────────────────┐                               │
│           ▼                ▼                ▼                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                       │
│  │  Card Proc   │  │  Bank Proc   │  │  Wallet Proc │                       │
│  │  Adapter     │  │  Adapter     │  │  Adapter     │                       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                       │
│         │                 │                 │                                │
└─────────┼─────────────────┼─────────────────┼────────────────────────────────┘
          ▼                 ▼                 ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Visa/MC/etc  │  │ Bank APIs    │  │ Apple Pay,   │
  │ Networks     │  │              │  │ Google Pay   │
  └──────────────┘  └──────────────┘  └──────────────┘
```

### Core Components

**1. Idempotency Service (Critical for Payments)**

```csharp
public class IdempotencyService
{
    private readonly IDistributedCache _cache;
    private readonly IDatabase _db;

    public async Task<PaymentResult?> CheckIdempotency(string key)
    {
        // Check cache first (fast path)
        var cached = await _cache.GetAsync<PaymentResult>(key);
        if (cached != null) return cached;

        // Check database (durable)
        return await _db.GetAsync<PaymentResult>(key);
    }

    public async Task StoreResult(string key, PaymentResult result)
    {
        // Store in both for durability + speed
        await Task.WhenAll(
            _cache.SetAsync(key, result, TimeSpan.FromHours(24)),
            _db.UpsertAsync(key, result)
        );
    }
}
```

**2. Transaction State Machine**

```
Payment States:
┌─────────────┐
│  INITIATED  │
└──────┬──────┘
       │ Validate
       ▼
┌─────────────┐     ┌─────────────┐
│ AUTHORIZED  │────→│   FAILED    │
└──────┬──────┘     └─────────────┘
       │ Capture            ▲
       ▼                    │
┌─────────────┐     ┌───────┴─────┐
│  CAPTURED   │────→│  REVERSED   │
└──────┬──────┘     └─────────────┘
       │ Settle
       ▼
┌─────────────┐     ┌─────────────┐
│   SETTLED   │────→│  REFUNDED   │
└─────────────┘     └─────────────┘
```

**3. Database Design**

```sql
-- Payments table (immutable ledger)
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    merchant_id UUID NOT NULL,
    amount DECIMAL(19,4) NOT NULL,
    currency CHAR(3) NOT NULL,
    status VARCHAR(20) NOT NULL,
    payment_method_type VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,

    INDEX idx_merchant_created (merchant_id, created_at),
    INDEX idx_idempotency (idempotency_key)
);

-- Payment events (event sourcing for audit)
CREATE TABLE payment_events (
    id UUID PRIMARY KEY,
    payment_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_data JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL,

    INDEX idx_payment_events (payment_id, created_at)
);
```

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Database | PostgreSQL | ACID compliance, JSONB for flexibility |
| Caching | Redis | Idempotency keys, rate limiting |
| Messaging | Kafka | Event streaming, audit log |
| Consistency | Strong for payments | Money can't be eventually consistent |
| Scaling | Horizontal API, Read replicas | Handle traffic spikes |

### Communication Tactics

- **Mention first**: "Idempotency is critical - we cannot charge twice"
- **Emphasize**: PCI-DSS compliance, audit trails, failure handling
- **Show depth**: State machine, compensation/refunds, monitoring

---

## Question 2: Design a Digital Wallet Application

### The Question
> "Design a digital wallet like MultiPay or Apple Pay. Users can load money, pay merchants, and transfer to others."

### Clarifying Questions to Ask
- Closed loop (only our merchants) or open loop (Visa/MC network)?
- What's the expected user base and transaction volume?
- Real-time transfers or batched?
- Regulatory requirements (e-money license)?

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           DIGITAL WALLET                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Mobile App ───→ API Gateway ───→ Auth Service (OAuth2/JWT)              │
│                       │                                                   │
│                       ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐     │
│  │                    CORE SERVICES                                 │     │
│  │                                                                  │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │     │
│  │  │   Account    │  │ Transaction  │  │   Ledger     │          │     │
│  │  │   Service    │  │   Service    │  │   Service    │          │     │
│  │  │              │  │              │  │              │          │     │
│  │  │ - User accts │  │ - P2P xfer   │  │ - Double     │          │     │
│  │  │ - KYC status │  │ - Pay merch  │  │   entry      │          │     │
│  │  │ - Limits     │  │ - Top-up     │  │ - Balances   │          │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘          │     │
│  │                                                                  │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │     │
│  │  │ Notification │  │   Fraud      │  │  Settlement  │          │     │
│  │  │   Service    │  │   Detection  │  │   Service    │          │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘          │     │
│  └─────────────────────────────────────────────────────────────────┘     │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
    ┌──────────┐        ┌──────────┐         ┌──────────┐
    │ Bank API │        │ Card     │         │ Merchant │
    │ (Top-up) │        │ Network  │         │   POS    │
    └──────────┘        └──────────┘         └──────────┘
```

### Core Component: Double-Entry Ledger

```csharp
public class LedgerService
{
    // Every transaction affects at least 2 accounts
    // Total debits always equals total credits

    public async Task<TransactionResult> Transfer(
        Guid fromAccount,
        Guid toAccount,
        decimal amount,
        string currency)
    {
        var transactionId = Guid.NewGuid();

        using var transaction = await _db.BeginTransactionAsync();

        try
        {
            // Debit from sender
            await CreateEntry(new LedgerEntry
            {
                TransactionId = transactionId,
                AccountId = fromAccount,
                EntryType = EntryType.Debit,
                Amount = amount,
                Currency = currency
            });

            // Credit to receiver
            await CreateEntry(new LedgerEntry
            {
                TransactionId = transactionId,
                AccountId = toAccount,
                EntryType = EntryType.Credit,
                Amount = amount,
                Currency = currency
            });

            // Update balances atomically
            await UpdateBalance(fromAccount, -amount);
            await UpdateBalance(toAccount, amount);

            await transaction.CommitAsync();

            return TransactionResult.Success(transactionId);
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}

// Ledger Entry - Immutable
public class LedgerEntry
{
    public Guid Id { get; init; }
    public Guid TransactionId { get; init; }
    public Guid AccountId { get; init; }
    public EntryType EntryType { get; init; }
    public decimal Amount { get; init; }
    public string Currency { get; init; }
    public DateTime CreatedAt { get; init; }

    // Entries are NEVER updated or deleted - only new entries created
}
```

### Database Schema

```sql
-- Accounts
CREATE TABLE accounts (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    account_type VARCHAR(20) NOT NULL, -- 'USER', 'MERCHANT', 'SYSTEM'
    currency CHAR(3) NOT NULL,
    balance DECIMAL(19,4) NOT NULL DEFAULT 0,
    available_balance DECIMAL(19,4) NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL,

    CONSTRAINT positive_balance CHECK (balance >= 0)
);

-- Ledger Entries (immutable)
CREATE TABLE ledger_entries (
    id UUID PRIMARY KEY,
    transaction_id UUID NOT NULL,
    account_id UUID NOT NULL REFERENCES accounts(id),
    entry_type VARCHAR(10) NOT NULL, -- 'DEBIT', 'CREDIT'
    amount DECIMAL(19,4) NOT NULL,
    currency CHAR(3) NOT NULL,
    created_at TIMESTAMP NOT NULL,

    INDEX idx_transaction (transaction_id),
    INDEX idx_account_date (account_id, created_at)
);

-- Verify ledger integrity
-- SUM of all debits should equal SUM of all credits
```

### Key Features

| Feature | Implementation |
|---------|----------------|
| Balance Check | SELECT FOR UPDATE to prevent overdraft |
| Transaction Limits | Per-day, per-transaction limits in Account Service |
| Fraud Detection | ML model scoring transactions in real-time |
| Notifications | Event-driven push/SMS on transaction complete |

---

## Question 3: Design an HR Management System

### The Question
> "How would you design a scalable HR management system for a company with 10,000+ employees?"

### Clarifying Questions
- What modules? (Employee data, payroll, leave, performance, recruitment)
- Multi-tenant (SaaS) or single company?
- Integration needs? (Payroll providers, job boards)
- Compliance requirements? (GDPR, labor law variations)

### High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                        HR MANAGEMENT SYSTEM                             │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Web App / Mobile ───→ API Gateway ───→ Auth (SSO/SAML/OAuth)         │
│                             │                                           │
│                             ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        CORE MODULES                              │   │
│  │                                                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │  Employee   │  │    Leave    │  │   Payroll   │             │   │
│  │  │  Service    │  │   Service   │  │   Service   │             │   │
│  │  │             │  │             │  │             │             │   │
│  │  │ - Profiles  │  │ - Balances  │  │ - Calc      │             │   │
│  │  │ - Org chart │  │ - Requests  │  │ - Taxes     │             │   │
│  │  │ - Documents │  │ - Approvals │  │ - Reports   │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │ Performance │  │ Recruitment │  │  Reporting  │             │   │
│  │  │   Service   │  │   Service   │  │   Service   │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │    Background Jobs: Payroll calc, Report generation, Reminders  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### Module: Leave Management

```csharp
public class LeaveService
{
    public async Task<LeaveRequestResult> SubmitRequest(LeaveRequestDto request)
    {
        // 1. Validate leave balance
        var balance = await _balanceService.GetBalance(
            request.EmployeeId,
            request.LeaveType);

        var requestedDays = CalculateWorkingDays(request.StartDate, request.EndDate);

        if (balance.Available < requestedDays)
            return LeaveRequestResult.InsufficientBalance();

        // 2. Check for conflicts
        var conflicts = await _leaveRepo.GetConflicts(
            request.EmployeeId,
            request.StartDate,
            request.EndDate);

        if (conflicts.Any())
            return LeaveRequestResult.Conflict(conflicts);

        // 3. Create request
        var leaveRequest = new LeaveRequest
        {
            Id = Guid.NewGuid(),
            EmployeeId = request.EmployeeId,
            LeaveType = request.LeaveType,
            StartDate = request.StartDate,
            EndDate = request.EndDate,
            Status = LeaveStatus.Pending,
            ApproverId = await GetApproverId(request.EmployeeId)
        };

        await _leaveRepo.CreateAsync(leaveRequest);

        // 4. Reserve balance
        await _balanceService.Reserve(
            request.EmployeeId,
            request.LeaveType,
            requestedDays);

        // 5. Notify approver
        await _notificationService.NotifyApprover(leaveRequest);

        return LeaveRequestResult.Success(leaveRequest.Id);
    }
}
```

### Database Optimization for Large Scale

```sql
-- Partition employee data by department for large organizations
CREATE TABLE employees (
    id UUID PRIMARY KEY,
    employee_number VARCHAR(20) UNIQUE NOT NULL,
    department_id UUID NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    hire_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL,
    -- ...
) PARTITION BY HASH (department_id);

-- Indexes for common queries
CREATE INDEX idx_emp_dept ON employees (department_id);
CREATE INDEX idx_emp_manager ON employees (manager_id);
CREATE INDEX idx_emp_status ON employees (status) WHERE status = 'ACTIVE';

-- Materialized view for org chart (refreshed nightly)
CREATE MATERIALIZED VIEW org_hierarchy AS
WITH RECURSIVE org AS (
    SELECT id, name, manager_id, 1 as level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, o.level + 1
    FROM employees e
    JOIN org o ON e.manager_id = o.id
)
SELECT * FROM org;
```

---

## Question 4: Design a Real-Time Notification System

### The Question
> "Design a notification system that can send millions of notifications across multiple channels (push, email, SMS, in-app)."

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      NOTIFICATION SYSTEM                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Services ───→ Notification API ───→ Message Queue (Kafka/RabbitMQ)      │
│                                              │                            │
│                    ┌─────────────────────────┼─────────────────────────┐  │
│                    ▼                         ▼                         ▼  │
│            ┌──────────────┐         ┌──────────────┐         ┌──────────┐│
│            │ Push Worker  │         │ Email Worker │         │SMS Worker││
│            │              │         │              │         │          ││
│            │ - FCM        │         │ - SendGrid   │         │ - Twilio ││
│            │ - APNS       │         │ - Template   │         │          ││
│            │ - Batching   │         │ - Batching   │         │          ││
│            └──────────────┘         └──────────────┘         └──────────┘│
│                    │                         │                         │  │
│                    └─────────────────────────┼─────────────────────────┘  │
│                                              ▼                            │
│                                    ┌────────────────┐                     │
│                                    │ Delivery Store │                     │
│                                    │ (Status, Logs) │                     │
│                                    └────────────────┘                     │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### Core Components

```csharp
// Notification Request
public record NotificationRequest
{
    public Guid UserId { get; init; }
    public string TemplateId { get; init; }
    public Dictionary<string, object> Data { get; init; }
    public NotificationPriority Priority { get; init; }
    public List<NotificationChannel> Channels { get; init; }
    public DateTime? ScheduledAt { get; init; }
}

// Notification Service
public class NotificationService
{
    private readonly IMessageQueue _queue;
    private readonly IUserPreferenceService _preferences;

    public async Task SendAsync(NotificationRequest request)
    {
        // 1. Get user preferences
        var prefs = await _preferences.GetAsync(request.UserId);

        // 2. Filter channels based on preferences and opt-outs
        var channels = request.Channels
            .Where(c => prefs.IsChannelEnabled(c))
            .ToList();

        // 3. Enqueue for each channel
        foreach (var channel in channels)
        {
            var message = new NotificationMessage
            {
                Id = Guid.NewGuid(),
                UserId = request.UserId,
                Channel = channel,
                TemplateId = request.TemplateId,
                Data = request.Data,
                Priority = request.Priority,
                ScheduledAt = request.ScheduledAt ?? DateTime.UtcNow
            };

            // Priority determines queue
            var queueName = request.Priority == NotificationPriority.High
                ? $"notifications.{channel}.high"
                : $"notifications.{channel}.normal";

            await _queue.PublishAsync(queueName, message);
        }
    }
}

// Push Notification Worker
public class PushNotificationWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var batch in _queue.ConsumeBatchAsync<NotificationMessage>(
            "notifications.push.*",
            batchSize: 1000,
            maxWaitTime: TimeSpan.FromSeconds(1),
            ct))
        {
            // FCM supports batch sending
            var fcmMessages = batch
                .Select(n => BuildFcmMessage(n))
                .ToList();

            var response = await _fcm.SendAllAsync(fcmMessages);

            // Track delivery status
            for (int i = 0; i < batch.Count; i++)
            {
                await _deliveryStore.RecordAsync(
                    batch[i].Id,
                    response.Responses[i].IsSuccess
                        ? DeliveryStatus.Sent
                        : DeliveryStatus.Failed,
                    response.Responses[i].Error?.Message);
            }
        }
    }
}
```

### Scaling Considerations

| Challenge | Solution |
|-----------|----------|
| High volume | Batch processing, multiple workers |
| Rate limits | Token bucket per channel, backoff |
| Failures | Dead letter queue, retry with exponential backoff |
| User prefs | Cache preferences, honor opt-outs |
| Tracking | Store delivery status, support callbacks |

---

## Question 5: Design a Rate Limiter

### The Question
> "Design a rate limiter for an API. How would you implement different limiting strategies?"

### Algorithms Comparison

```
1. TOKEN BUCKET
   - Bucket holds tokens (refilled at constant rate)
   - Request consumes token
   - Allows bursts up to bucket size
   - Most common for APIs

2. SLIDING WINDOW
   - Count requests in sliding time window
   - More memory than fixed window
   - Smoother limiting

3. FIXED WINDOW
   - Simple: count per time window
   - Edge case: burst at window boundary
   - Easy to implement

4. LEAKY BUCKET
   - Queue requests, process at fixed rate
   - Smooths traffic completely
   - Used for traffic shaping
```

### Implementation: Token Bucket with Redis

```csharp
public class RateLimiter
{
    private readonly IDatabase _redis;

    public async Task<RateLimitResult> CheckRateLimit(
        string clientId,
        RateLimitPolicy policy)
    {
        var key = $"ratelimit:{policy.Name}:{clientId}";

        // Lua script for atomic token bucket
        var script = @"
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local refillRate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            local requested = tonumber(ARGV[4])

            local bucket = redis.call('HMGET', key, 'tokens', 'lastRefill')
            local tokens = tonumber(bucket[1]) or capacity
            local lastRefill = tonumber(bucket[2]) or now

            -- Calculate token refill
            local elapsed = now - lastRefill
            local refillTokens = math.floor(elapsed * refillRate / 1000)
            tokens = math.min(capacity, tokens + refillTokens)

            -- Check if request can be served
            local allowed = tokens >= requested
            if allowed then
                tokens = tokens - requested
            end

            -- Update bucket
            redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
            redis.call('EXPIRE', key, 3600)

            return {allowed and 1 or 0, tokens}
        ";

        var result = await _redis.ScriptEvaluateAsync(script,
            keys: new RedisKey[] { key },
            values: new RedisValue[]
            {
                policy.Capacity,
                policy.RefillRatePerSecond,
                DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
                1
            });

        var values = (RedisValue[])result;

        return new RateLimitResult
        {
            Allowed = (int)values[0] == 1,
            RemainingTokens = (int)values[1],
            RetryAfterMs = (int)values[0] == 0
                ? (int)(1000.0 / policy.RefillRatePerSecond)
                : 0
        };
    }
}

// Rate Limit Middleware
public class RateLimitMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var clientId = GetClientId(context); // IP, API key, user ID
        var policy = GetPolicy(context.Request.Path);

        var result = await _rateLimiter.CheckRateLimit(clientId, policy);

        // Set standard headers
        context.Response.Headers["X-RateLimit-Limit"] = policy.Capacity.ToString();
        context.Response.Headers["X-RateLimit-Remaining"] = result.RemainingTokens.ToString();

        if (!result.Allowed)
        {
            context.Response.Headers["Retry-After"] =
                (result.RetryAfterMs / 1000).ToString();
            context.Response.StatusCode = 429; // Too Many Requests
            return;
        }

        await next(context);
    }
}
```

---

## Question 6: Design a High-Availability Transaction System

### The Question
> "Design a transaction processing system with high availability. How do you ensure no data loss and minimal downtime?"

### Architecture for High Availability

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HIGH AVAILABILITY ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│              ┌────────────────────────────────────┐                     │
│              │         Load Balancer              │                     │
│              │    (Active-Active, Health Checks)  │                     │
│              └──────────────┬─────────────────────┘                     │
│                             │                                            │
│          ┌──────────────────┼──────────────────────┐                    │
│          ▼                  ▼                      ▼                    │
│  ┌──────────────┐   ┌──────────────┐      ┌──────────────┐             │
│  │   API Pod 1  │   │   API Pod 2  │ ...  │   API Pod N  │             │
│  │  (Stateless) │   │  (Stateless) │      │  (Stateless) │             │
│  └──────┬───────┘   └──────┬───────┘      └──────┬───────┘             │
│         │                  │                      │                     │
│         └──────────────────┼──────────────────────┘                     │
│                            ▼                                            │
│              ┌─────────────────────────────┐                            │
│              │     Message Queue Cluster   │                            │
│              │   (Kafka / RabbitMQ HA)     │                            │
│              └──────────────┬──────────────┘                            │
│                             │                                           │
│          ┌──────────────────┼──────────────────────┐                    │
│          ▼                  ▼                      ▼                    │
│  ┌──────────────┐   ┌──────────────┐      ┌──────────────┐             │
│  │   Worker 1   │   │   Worker 2   │ ...  │   Worker N   │             │
│  └──────────────┘   └──────────────┘      └──────────────┘             │
│                            │                                            │
│                            ▼                                            │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │                     Database Cluster                          │      │
│  │  ┌─────────┐       ┌─────────┐       ┌─────────┐             │      │
│  │  │ Primary │ ────→ │ Replica │       │ Replica │             │      │
│  │  │(Write)  │ sync  │ (Read)  │       │ (Read)  │             │      │
│  │  └─────────┘       └─────────┘       └─────────┘             │      │
│  │              (Synchronous replication for critical data)      │      │
│  └──────────────────────────────────────────────────────────────┘      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Patterns

**1. Outbox Pattern for Reliable Messaging**

```csharp
public class TransactionService
{
    public async Task ProcessTransaction(Transaction txn)
    {
        using var dbTransaction = await _context.Database.BeginTransactionAsync();

        try
        {
            // 1. Save transaction
            _context.Transactions.Add(txn);

            // 2. Save outbox message (same DB transaction)
            _context.OutboxMessages.Add(new OutboxMessage
            {
                Id = Guid.NewGuid(),
                Type = "TransactionProcessed",
                Payload = JsonSerializer.Serialize(txn),
                CreatedAt = DateTime.UtcNow
            });

            await _context.SaveChangesAsync();
            await dbTransaction.CommitAsync();

            // Transaction and message saved atomically
            // Background processor publishes to message queue
        }
        catch
        {
            await dbTransaction.RollbackAsync();
            throw;
        }
    }
}
```

**2. Circuit Breaker for External Services**

```csharp
// Using Polly
var circuitBreaker = Policy
    .Handle<HttpRequestException>()
    .Or<TimeoutException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (ex, duration) =>
            _logger.LogWarning("Circuit opened for {Duration}", duration),
        onReset: () =>
            _logger.LogInformation("Circuit closed")
    );
```

### Availability Metrics

| Metric | Target | How to Achieve |
|--------|--------|----------------|
| Uptime | 99.9% (8.76h downtime/year) | Multi-AZ deployment, auto-failover |
| RTO (Recovery Time) | < 5 minutes | Automated failover, health checks |
| RPO (Recovery Point) | 0 (no data loss) | Synchronous replication |
| Latency P99 | < 200ms | Connection pooling, caching, indexing |

---

## Question 7: Design a Caching Strategy for Banking Application

### The Question
> "Design a caching strategy for a banking application. What data can be cached and what cannot?"

### Caching Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CACHING DECISION MATRIX                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  CAN CACHE (High Read, Low Write):                                      │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ - Exchange rates (refresh every minute)                         │    │
│  │ - Product catalog (credit cards, loan terms)                   │    │
│  │ - Branch/ATM locations                                          │    │
│  │ - User profile (name, preferences) - invalidate on update      │    │
│  │ - Session data                                                   │    │
│  │ - Reference data (country codes, currency codes)               │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  NEVER CACHE (Always real-time):                                        │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ - Account balances (critical for overdraft prevention)         │    │
│  │ - Transaction status (pending/completed)                        │    │
│  │ - Authentication tokens (security risk)                         │    │
│  │ - Fraud flags                                                    │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  CACHE WITH CAUTION (Short TTL + Invalidation):                        │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ - Recent transactions (display only, 5-second TTL)             │    │
│  │ - Daily limits remaining (invalidate on transaction)           │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Multi-Level Caching Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                      MULTI-LEVEL CACHE                                  │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Request ──→ L1: In-Memory (IMemoryCache)                             │
│                   - Fastest (microseconds)                              │
│                   - Per-instance (not shared)                           │
│                   - Small size (100MB)                                  │
│                   │                                                     │
│                   ▼ Cache Miss                                          │
│               L2: Distributed Cache (Redis)                             │
│                   - Fast (1-5ms)                                        │
│                   - Shared across instances                             │
│                   - Larger size (GB)                                    │
│                   │                                                     │
│                   ▼ Cache Miss                                          │
│               L3: Database                                              │
│                   - Slowest (10-100ms)                                  │
│                   - Source of truth                                     │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```csharp
public class CacheService
{
    private readonly IMemoryCache _memoryCache;
    private readonly IDistributedCache _redis;
    private readonly IDatabase _database;

    public async Task<T?> GetAsync<T>(
        string key,
        Func<Task<T>> factory,
        CacheOptions options) where T : class
    {
        // L1: Check in-memory cache
        if (_memoryCache.TryGetValue(key, out T? cached))
            return cached;

        // L2: Check Redis
        var redisValue = await _redis.GetStringAsync(key);
        if (redisValue != null)
        {
            var value = JsonSerializer.Deserialize<T>(redisValue);

            // Populate L1 cache
            _memoryCache.Set(key, value, options.MemoryCacheDuration);

            return value;
        }

        // L3: Get from database
        var dbValue = await factory();

        if (dbValue != null)
        {
            // Populate both caches
            var json = JsonSerializer.Serialize(dbValue);

            await _redis.SetStringAsync(
                key,
                json,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = options.RedisCacheDuration
                });

            _memoryCache.Set(key, dbValue, options.MemoryCacheDuration);
        }

        return dbValue;
    }

    public async Task InvalidateAsync(string key)
    {
        _memoryCache.Remove(key);
        await _redis.RemoveAsync(key);

        // Publish invalidation event for other instances
        await _redis.PublishAsync("cache:invalidate", key);
    }
}

// Cache invalidation subscriber (in each instance)
public class CacheInvalidationSubscriber : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var subscriber = _redis.GetSubscriber();

        await subscriber.SubscribeAsync("cache:invalidate", (channel, key) =>
        {
            _memoryCache.Remove(key.ToString());
        });
    }
}
```

### Banking-Specific Patterns

| Pattern | Use Case | Implementation |
|---------|----------|----------------|
| Read-Through | Product catalog | Cache fetches from DB on miss |
| Write-Through | User preferences | Update DB and cache together |
| Write-Behind | Audit logs | Write to cache, async persist |
| Cache-Aside | Most scenarios | App manages cache explicitly |

---

## Quick Review - System Design Principles

| Principle | Application |
|-----------|-------------|
| Start with requirements | Clarify functional and non-functional |
| Estimate scale | Users, RPS, data volume |
| Design incrementally | High-level first, then deep dive |
| Consider trade-offs | Consistency vs availability, cost vs performance |
| Think about failures | What happens when X fails? |
| Discuss operations | Monitoring, deployment, maintenance |
