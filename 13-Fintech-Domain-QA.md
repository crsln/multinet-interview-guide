# Fintech & Payments Domain Knowledge

> 7 fintech domain questions with detailed answers for payment system interviews

---

## About Multinet Up (Interview Context)

| Attribute | Details |
|-----------|---------|
| **Company** | Multinet Up - Turkey's largest meal and shopping voucher network |
| **Parent** | UP Group (Paris) - acquired in 2010 |
| **Scale** | 552 employees, 1.8M card users, 60,000 contracted points |
| **Products** | MultiNet (meal), MultiGift, MultiPetrol, MultiPay (wallet), MultiPOS |
| **Tech Subsidiary** | Inventiv - develops digital wallets, mPOS, loyalty programs |

**Why This Matters:** Understanding Multinet's business shows you've done your homework and can discuss domain-specific challenges.

---

## Question 1: PCI-DSS Compliance for Developers

### The Question
> "Explain PCI-DSS compliance. What should developers know when building payment systems?"

### Key Points to Cover
- What PCI-DSS is and why it matters
- Key requirements relevant to developers
- Secure coding practices
- Common compliance mistakes

### Detailed Answer

**What is PCI-DSS:**

```
PCI DSS = Payment Card Industry Data Security Standard

Who created it: Major card networks (Visa, Mastercard, Amex, Discover)
Why: To protect cardholder data and reduce fraud
Who must comply: Any organization that stores, processes, or transmits card data
```

**Key Requirements for Developers:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  PCI-DSS DEVELOPER REQUIREMENTS                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Requirement 3: Protect Stored Cardholder Data                          │
│  ────────────────────────────────────────────────────────────────────   │
│  DO:                                                                     │
│  - Encrypt stored card data (AES-256)                                   │
│  - Use tokenization instead of storing actual card numbers              │
│  - Mask PAN when displayed (show only last 4 digits)                    │
│                                                                          │
│  DON'T:                                                                  │
│  - NEVER store CVV/CVC after authorization                              │
│  - NEVER store full magnetic stripe data                                │
│  - NEVER store PIN or encrypted PIN block                               │
│                                                                          │
│  Requirement 4: Encrypt Data in Transit                                 │
│  ────────────────────────────────────────────────────────────────────   │
│  - Use TLS 1.2+ for all card data transmission                          │
│  - No card data over email, chat, or SMS                                │
│  - Validate SSL certificates                                            │
│                                                                          │
│  Requirement 6: Develop Secure Systems                                  │
│  ────────────────────────────────────────────────────────────────────   │
│  - Security training for developers (annual minimum)                    │
│  - Input validation (prevent SQL injection, XSS)                        │
│  - Code review before production                                         │
│  - Patch management for dependencies                                     │
│  - Follow OWASP Top 10 guidelines                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Tokenization in Practice:**

```csharp
// Instead of storing card numbers, store tokens
public class PaymentService
{
    public async Task<string> StoreCardForFutureUse(CardDetails card)
    {
        // Send card to PCI-compliant vault (Stripe, Adyen, etc.)
        var token = await _vaultService.Tokenize(card);

        // Store only the token in your database
        await _db.SaveAsync(new StoredPaymentMethod
        {
            UserId = userId,
            Token = token,                    // "tok_abc123" - safe to store
            Last4 = card.Number[^4..],        // "4242" - for display
            Brand = card.Brand,               // "Visa"
            ExpiryMonth = card.ExpiryMonth,
            ExpiryYear = card.ExpiryYear
        });

        // Original card number never touches your database
        return token;
    }
}
```

**PCI DSS 4.0 Changes (2024):**

| Change | Impact on Developers |
|--------|---------------------|
| Security in SDLC | Security reviews required at each development phase |
| Training | Structured annual training on secure coding |
| DevSecOps | Security testing integrated into CI/CD |
| Threat Modeling | Required before major changes |

### Communication Tactics

- **Emphasize**: "I understand that PCI compliance is about protecting cardholder data at every stage - storage, transit, and processing"
- **Show practical knowledge**: Mention tokenization, never storing CVV, TLS requirements
- **Reference**: OWASP Top 10, secure coding practices

---

## Question 2: Payment Transaction Flow

### The Question
> "How do payment transactions work end-to-end? Walk me through what happens when a customer swipes their card."

### Key Points to Cover
- Key participants in the ecosystem
- Authorization vs settlement
- The message flow
- Timeframes involved

### Detailed Answer

**Payment Ecosystem Players:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                    PAYMENT ECOSYSTEM                                    │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CARDHOLDER                                                             │
│      │                                                                  │
│      │ 1. Swipes card                                                   │
│      ▼                                                                  │
│  MERCHANT (Point of Sale)                                               │
│      │                                                                  │
│      │ 2. Sends transaction data                                        │
│      ▼                                                                  │
│  ACQUIRER (Merchant's Bank)                                             │
│      │  - Processes transactions on behalf of merchant                  │
│      │  - Examples: Worldpay, First Data                                │
│      │                                                                  │
│      │ 3. Routes through network                                        │
│      ▼                                                                  │
│  CARD NETWORK (Visa, Mastercard)                                        │
│      │  - Routes transactions between banks                             │
│      │  - Sets interchange fees                                         │
│      │                                                                  │
│      │ 4. Forwards to issuer                                            │
│      ▼                                                                  │
│  ISSUER (Customer's Bank)                                               │
│      │  - Issued the card to customer                                   │
│      │  - Approves or declines based on funds/credit                   │
│      │                                                                  │
│      │ 5. Response flows back                                           │
│      ▼                                                                  │
│  BACK TO MERCHANT (approval/decline in ~2 seconds)                     │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

**Authorization vs Settlement:**

```
AUTHORIZATION (Real-time, ~2 seconds)
─────────────────────────────────────
1. Customer presents card
2. Merchant sends authorization request
3. Issuer checks:
   - Is card valid?
   - Are funds available?
   - Is it potentially fraud?
4. Issuer approves/declines
5. If approved: funds are HELD (not transferred yet)
   - Available balance decreases immediately
   - Posted balance unchanged

CAPTURE (Same day)
─────────────────────────────────────
1. Merchant signals intent to charge
2. Some merchants capture immediately (retail)
3. Others delay (hotels, gas stations - final amount may vary)

CLEARING & SETTLEMENT (1-3 days)
─────────────────────────────────────
1. End of day: Merchant sends batch of captured transactions
2. Acquirer processes batch
3. Card network facilitates clearing
4. Settlement: Actual money moves
   - Issuer pays card network
   - Card network pays acquirer
   - Acquirer pays merchant (minus fees)
5. Customer sees transaction on statement
```

**Message Format (ISO-8583):**

```
ISO-8583 is the standard for card payment messages.

Key fields:
- MTI (Message Type Indicator): 0100 = Auth request, 0110 = Auth response
- PAN (Primary Account Number): Card number
- Processing Code: Type of transaction (purchase, refund, etc.)
- Amount: Transaction amount
- Currency Code: 840 (USD), 949 (TRY), etc.
- Merchant Category Code (MCC): Type of business
- Response Code: 00 = Approved, 05 = Do not honor, etc.
```

**Timeline:**

| Phase | Duration | What Happens |
|-------|----------|--------------|
| Authorization | 1-3 seconds | Real-time approval |
| Capture | Same day | Merchant confirms charge |
| Clearing | End of day | Transaction data exchanged |
| Settlement | 1-3 days | Money moves between banks |
| Funding | 1-2 days | Merchant receives funds |

### Communication Tactics

- **Show the full picture**: Walk through all parties involved
- **Distinguish phases**: Authorization is instant; settlement takes days
- **Mention standards**: ISO-8583 shows deeper knowledge

---

## Question 3: Idempotency in Payments

### The Question
> "What is idempotency and why is it critical in payment systems?"

### Key Points to Cover
- Definition and importance
- Why payments need idempotency
- Implementation patterns
- Real-world scenarios

### Detailed Answer

**What is Idempotency:**

```
Idempotency: An operation that produces the same result
no matter how many times it's executed.

Mathematical: f(f(x)) = f(x)

In APIs: Calling the same request multiple times has the same
effect as calling it once.
```

**Why It's Critical for Payments:**

```
SCENARIO: Customer charged twice due to network retry

1. Customer clicks "Pay"
2. Payment request sent to server
3. Server processes payment successfully
4. Network timeout before response reaches client
5. Client retries (user clicks again, or auto-retry)
6. Server processes payment AGAIN
7. Customer charged TWICE!

With idempotency:
1. Customer clicks "Pay"
2. Payment request sent with Idempotency-Key: "order-123-attempt"
3. Server processes, stores result with key
4. Network timeout
5. Client retries with same Idempotency-Key
6. Server finds cached result, returns it
7. Customer charged ONCE, even with retries
```

**Implementation Pattern:**

```csharp
public class PaymentController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> ProcessPayment(
        [FromBody] PaymentRequest request,
        [FromHeader(Name = "Idempotency-Key")] string idempotencyKey)
    {
        // 1. Require idempotency key for payments
        if (string.IsNullOrEmpty(idempotencyKey))
            return BadRequest("Idempotency-Key header is required");

        // 2. Check if we've seen this key before
        var existingResult = await _idempotencyStore.GetAsync(idempotencyKey);
        if (existingResult != null)
        {
            // Return cached response
            Response.Headers["X-Idempotent-Replay"] = "true";
            return StatusCode(existingResult.StatusCode, existingResult.Body);
        }

        // 3. Acquire lock to prevent concurrent processing
        await using var lock = await _lockService.AcquireAsync(
            $"payment:{idempotencyKey}",
            TimeSpan.FromSeconds(30));

        if (lock == null)
            return Conflict("Request already in progress");

        // 4. Double-check after lock (race condition protection)
        existingResult = await _idempotencyStore.GetAsync(idempotencyKey);
        if (existingResult != null)
        {
            Response.Headers["X-Idempotent-Replay"] = "true";
            return StatusCode(existingResult.StatusCode, existingResult.Body);
        }

        // 5. Process the payment
        var result = await _paymentService.ProcessAsync(request);

        // 6. Store result for future retries (24-hour TTL typical)
        await _idempotencyStore.SetAsync(idempotencyKey, new IdempotencyRecord
        {
            StatusCode = 200,
            Body = result,
            CreatedAt = DateTime.UtcNow
        }, TimeSpan.FromHours(24));

        return Ok(result);
    }
}
```

**Key Design Decisions:**

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Key format | Client-generated UUID or `{orderId}-{attempt}` | Client controls uniqueness |
| Storage | Redis + Database | Fast lookup + durability |
| TTL | 24 hours | Balance between safety and storage |
| Scope | Per-API-key (merchant) | Same key from different merchants is different |

**Interview Answer Script:**

> "Idempotency ensures a payment is processed exactly once, even if the request is sent multiple times. In payments, this is critical because:
>
> 1. Network failures cause retries
> 2. Users double-click submit buttons
> 3. Mobile apps retry on timeout
>
> I implement it using an idempotency key that the client sends with each request. The server checks if we've seen this key before. If yes, return the cached response. If no, process and cache the result.
>
> The key implementation details are: require the key for all payment endpoints, use distributed locking to prevent race conditions, and store results long enough to handle delayed retries."

### Communication Tactics

- **Explain the problem first**: Why do we need this?
- **Give concrete scenario**: Network timeout causing double charge
- **Show implementation awareness**: Idempotency key, locking, caching

---

## Question 4: Card-Present vs Card-Not-Present

### The Question
> "Explain the difference between card-present and card-not-present transactions. Why does it matter?"

### Detailed Answer

**Definitions:**

```
CARD-PRESENT (CP)
─────────────────
Card is physically present during transaction
- In-store purchases
- ATM withdrawals
- Chip/PIN or contactless at POS

CARD-NOT-PRESENT (CNP)
─────────────────────
Card is NOT physically present
- Online/e-commerce purchases
- Phone orders
- Mail orders
- Recurring billing
```

**Why It Matters:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CP vs CNP COMPARISON                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  FRAUD RISK                                                              │
│  ───────────                                                             │
│  CP:  Lower - physical card + cardholder verification                   │
│  CNP: Higher - only card data required, easier to use stolen details    │
│                                                                          │
│  INTERCHANGE FEES                                                        │
│  ────────────────                                                        │
│  CP:  Lower fees (1.5-2.0% typical)                                     │
│  CNP: Higher fees (2.5-3.5% typical) - risk premium                     │
│                                                                          │
│  LIABILITY                                                               │
│  ─────────                                                               │
│  CP:  With EMV chip, liability shifts to issuer (if chip was used)      │
│  CNP: Merchant usually bears fraud liability (unless 3DS authenticated) │
│                                                                          │
│  AUTHENTICATION                                                          │
│  ──────────────                                                          │
│  CP:  Chip + PIN, signature, contactless with limits                    │
│  CNP: CVV, 3D Secure, address verification                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**CNP Security Measures:**

```csharp
// E-commerce checkout security layers
public class CheckoutService
{
    public async Task<PaymentResult> ProcessOnlinePayment(OnlinePaymentRequest request)
    {
        // 1. CVV Verification (Card Verification Value)
        // The 3-4 digit code on the card
        // Proves cardholder has physical card (mostly)

        // 2. AVS (Address Verification System)
        // Compare billing address with issuer's records
        var avsResult = await _processor.VerifyAddress(
            request.BillingAddress,
            request.CardNumber);

        if (avsResult.Code == "N") // No match
            return PaymentResult.DeclineForAVS();

        // 3. 3D Secure (Verified by Visa, Mastercard SecureCode)
        // Shifts liability to issuer if authenticated
        var threeDsResult = await _threeDsService.Authenticate(request);

        if (threeDsResult.Authenticated)
        {
            // Liability shifts to issuer
            // Include authentication data in authorization
        }

        // 4. Fraud scoring
        var fraudScore = await _fraudService.Score(request);
        if (fraudScore > 0.8)
            return PaymentResult.ReviewRequired();

        return await _processor.Authorize(request);
    }
}
```

### Communication Tactics

- **Frame as risk management**: CNP = higher risk = higher fees
- **Mention 3D Secure**: Shows you know the mitigation strategies
- **Connect to liability**: Who pays when fraud happens

---

## Question 5: Fraud Detection Patterns

### The Question
> "What fraud detection patterns should developers understand? How does ML help?"

### Detailed Answer

**Common Fraud Patterns:**

```
1. VELOCITY CHECKS
   - Many transactions in short time
   - Same card at multiple merchants
   - Many cards from same IP/device

2. GEOGRAPHIC ANOMALIES
   - Transaction in Turkey, then US within 1 hour
   - First-time country usage

3. BEHAVIORAL ANOMALIES
   - Unusual purchase amount
   - Purchase category mismatch (cardholder never buys jewelry)
   - Time of day (3 AM purchases)

4. CARD TESTING
   - Small transactions to verify card works
   - Followed by large purchase
   - Often from same IP

5. ACCOUNT TAKEOVER
   - Changed shipping address + immediate purchase
   - Password reset before purchase
```

**Rule-Based vs ML:**

```
RULE-BASED                          ML-BASED
─────────────                       ────────
- If amount > $5000 → flag         - Learn patterns from historical data
- If country changed → flag        - Detect subtle anomalies
- If 5+ txns in 1 hour → flag      - Adapt as fraud patterns change

Pros:                               Pros:
- Explainable                       - Catches complex patterns
- Easy to implement                 - Reduces false positives
- Fast to adjust                    - Scales with data

Cons:                               Cons:
- Rigid                             - Black box
- Many false positives              - Needs training data
- Can't catch new patterns          - More complex to implement
```

**Implementation Approach:**

```csharp
public class FraudDetectionService
{
    public async Task<FraudScore> ScoreTransaction(Transaction txn)
    {
        var signals = new List<FraudSignal>();

        // 1. Rule-based checks (fast, first line of defense)
        signals.Add(CheckVelocity(txn));
        signals.Add(CheckGeography(txn));
        signals.Add(CheckAmount(txn));

        // 2. ML model (more sophisticated)
        var mlScore = await _mlModel.PredictAsync(new FraudFeatures
        {
            Amount = txn.Amount,
            MerchantCategory = txn.MCC,
            TimeOfDay = txn.Timestamp.Hour,
            DayOfWeek = txn.Timestamp.DayOfWeek,
            CountryMatch = txn.Country == txn.Card.IssuingCountry,
            TransactionsLast24h = await GetRecentCount(txn.CardId, 24),
            AvgTransactionAmount = await GetAvgAmount(txn.CardId),
            // ... many more features
        });

        signals.Add(new FraudSignal("ML_MODEL", mlScore));

        // 3. Combine signals into final score
        var finalScore = CombineSignals(signals);

        return new FraudScore
        {
            Score = finalScore,
            Signals = signals,
            Recommendation = finalScore > 0.8 ? "DECLINE"
                           : finalScore > 0.5 ? "REVIEW"
                           : "APPROVE"
        };
    }
}
```

**Key Metrics:**

| Metric | Description | Goal |
|--------|-------------|------|
| True Positive Rate | Fraud caught / Total fraud | Maximize |
| False Positive Rate | Good txns blocked / Total good | Minimize |
| Precision | Correct flags / Total flags | Balance |
| Time to Detect | How quickly fraud is identified | Minimize |

### Communication Tactics

- **Show layered approach**: Rules + ML together
- **Mention trade-offs**: False positives hurt customer experience
- **Real examples**: Velocity checks, geographic anomalies

---

## Question 6: Digital Wallet Architecture

### The Question
> "How do digital wallets work technically? What are the key components?"

### Detailed Answer

**Core Components:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      DIGITAL WALLET ARCHITECTURE                          │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  MOBILE APP                                                               │
│  ──────────                                                               │
│  - Secure element / tokenized credentials                                │
│  - Biometric authentication                                              │
│  - QR code generation/scanning                                           │
│                                                                           │
│  ACCOUNT SERVICE                                                          │
│  ───────────────                                                          │
│  - User registration with KYC                                            │
│  - Account limits based on verification level                            │
│  - Multiple wallet types (personal, business)                            │
│                                                                           │
│  LEDGER SERVICE                                                           │
│  ──────────────                                                           │
│  - Double-entry bookkeeping                                              │
│  - Immutable transaction log                                             │
│  - Real-time balance calculation                                         │
│                                                                           │
│  FUNDING SERVICE                                                          │
│  ───────────────                                                          │
│  - Bank account linking                                                   │
│  - Card top-up                                                            │
│  - Cash loading points                                                    │
│                                                                           │
│  PAYMENT SERVICE                                                          │
│  ───────────────                                                          │
│  - P2P transfers                                                          │
│  - Merchant payments (QR, NFC)                                           │
│  - Online checkout integration                                            │
│                                                                           │
│  COMPLIANCE SERVICE                                                       │
│  ─────────────────                                                        │
│  - AML checks                                                             │
│  - Transaction monitoring                                                 │
│  - Regulatory reporting                                                   │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

**Wallet Types:**

```
CLOSED LOOP                        OPEN LOOP
───────────                        ─────────
- Only usable at specific          - Works like debit card
  merchants (like Multinet)        - Accepted anywhere on network

- Simpler compliance               - Full card network integration
- Lower fees                       - More regulatory requirements
- Limited utility                  - Greater flexibility

Example: Starbucks card            Example: Apple Pay, Venmo card
```

**Ledger Implementation:**

```csharp
// Double-entry ledger ensures money never appears or disappears
public class WalletLedger
{
    public async Task<TransferResult> Transfer(
        Guid fromWallet,
        Guid toWallet,
        decimal amount,
        string currency,
        string reference)
    {
        var transactionId = Guid.NewGuid();

        using var dbTxn = await _db.BeginTransactionAsync(
            IsolationLevel.Serializable);

        try
        {
            // Check balance
            var fromBalance = await GetBalanceForUpdate(fromWallet);
            if (fromBalance < amount)
                return TransferResult.InsufficientFunds();

            // Create ledger entries (double-entry)
            var entries = new[]
            {
                new LedgerEntry
                {
                    TransactionId = transactionId,
                    WalletId = fromWallet,
                    Type = EntryType.Debit,
                    Amount = amount,
                    Currency = currency,
                    Reference = reference,
                    CreatedAt = DateTime.UtcNow
                },
                new LedgerEntry
                {
                    TransactionId = transactionId,
                    WalletId = toWallet,
                    Type = EntryType.Credit,
                    Amount = amount,
                    Currency = currency,
                    Reference = reference,
                    CreatedAt = DateTime.UtcNow
                }
            };

            await _db.LedgerEntries.AddRangeAsync(entries);

            // Update cached balances
            await UpdateBalance(fromWallet, -amount);
            await UpdateBalance(toWallet, amount);

            await _db.SaveChangesAsync();
            await dbTxn.CommitAsync();

            return TransferResult.Success(transactionId);
        }
        catch
        {
            await dbTxn.RollbackAsync();
            throw;
        }
    }

    // Balance is always calculable from ledger
    public async Task<decimal> CalculateBalance(Guid walletId)
    {
        return await _db.LedgerEntries
            .Where(e => e.WalletId == walletId)
            .SumAsync(e => e.Type == EntryType.Credit ? e.Amount : -e.Amount);
    }
}
```

### Communication Tactics

- **Show architecture understanding**: Key components and their roles
- **Mention compliance**: KYC, AML are critical for wallets
- **Double-entry accounting**: Core to financial systems

---

## Question 7: 3D Secure Authentication

### The Question
> "What is 3D Secure authentication and how does it work? Why is it important?"

### Detailed Answer

**What is 3D Secure:**

```
3D Secure (3DS) = Three Domain Secure

The three domains:
1. ISSUER DOMAIN - Customer's bank
2. ACQUIRER DOMAIN - Merchant's bank
3. INTEROPERABILITY DOMAIN - Card network (Visa/MC)

Brand names:
- Visa: "Verified by Visa"
- Mastercard: "Mastercard SecureCode" / "Identity Check"
- Amex: "SafeKey"
```

**Why It Matters:**

```
LIABILITY SHIFT
───────────────
Without 3DS: Merchant liable for fraud chargebacks
With 3DS:    Issuer liable (if authentication succeeded)

This is HUGE for merchants - fraud losses can be devastating
```

**3DS Flow:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         3D SECURE FLOW                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Customer clicks "Pay" on merchant site                              │
│     │                                                                    │
│     ▼                                                                    │
│  2. Merchant sends payment data to 3DS Server                           │
│     │                                                                    │
│     ▼                                                                    │
│  3. 3DS Server contacts Directory Server (card network)                 │
│     │                                                                    │
│     ▼                                                                    │
│  4. Directory Server routes to Issuer's ACS (Access Control Server)     │
│     │                                                                    │
│     ▼                                                                    │
│  5. ACS performs risk assessment:                                        │
│     │                                                                    │
│     ├── Low risk  → FRICTIONLESS: Approve silently                      │
│     │               (Customer sees nothing extra)                        │
│     │                                                                    │
│     └── High risk → CHALLENGE: Show authentication                      │
│                     (OTP, biometric, app approval)                       │
│     │                                                                    │
│     ▼                                                                    │
│  6. Authentication result returned to merchant                          │
│     │                                                                    │
│     ▼                                                                    │
│  7. Merchant includes authentication data in authorization              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**3DS 1.0 vs 2.0:**

| Aspect | 3DS 1.0 | 3DS 2.0 |
|--------|---------|---------|
| User Experience | Always shows challenge (password) | Risk-based, often frictionless |
| Mobile Support | Poor (redirect-based) | Native SDK support |
| Data Shared | Minimal | 100+ data points for risk assessment |
| Success Rate | ~70% | ~90% (less friction) |
| Regulation | Optional | Required in EU (PSD2 SCA) |

**Implementation:**

```csharp
public class ThreeDSecureService
{
    public async Task<ThreeDSResult> Authenticate(PaymentRequest request)
    {
        // 1. Create authentication request
        var authRequest = new ThreeDSAuthRequest
        {
            CardNumber = request.CardNumber,
            Amount = request.Amount,
            Currency = request.Currency,
            MerchantName = _merchantConfig.Name,

            // Browser/device data for risk assessment
            BrowserUserAgent = request.UserAgent,
            BrowserLanguage = request.Language,
            DeviceChannel = "BROWSER",
            IpAddress = request.IpAddress,

            // Transaction data
            BillingAddress = request.BillingAddress,
            ShippingAddress = request.ShippingAddress,
            CustomerEmail = request.Email
        };

        // 2. Send to 3DS server
        var response = await _threeDSClient.AuthenticateAsync(authRequest);

        // 3. Handle response
        switch (response.TransactionStatus)
        {
            case "Y": // Fully authenticated
                return ThreeDSResult.Authenticated(response.AuthenticationValue);

            case "A": // Attempted authentication
                return ThreeDSResult.Attempted(response.AuthenticationValue);

            case "C": // Challenge required
                return ThreeDSResult.ChallengeRequired(
                    response.AcsUrl,
                    response.ChallengeRequest);

            case "N": // Not authenticated
                return ThreeDSResult.Failed(response.Reason);

            default:
                return ThreeDSResult.Unavailable();
        }
    }
}
```

### Communication Tactics

- **Explain liability shift**: This is the business value
- **3DS 2.0 improvements**: Frictionless flow, better UX
- **Regulatory context**: PSD2/SCA in Europe requires strong authentication

---

## Quick Review - Fintech Terminology

| Term | Definition |
|------|------------|
| **PAN** | Primary Account Number (card number) |
| **CVV/CVC** | Card Verification Value (3-4 digit security code) |
| **EMV** | Europay, Mastercard, Visa - chip card standard |
| **Acquirer** | Merchant's bank that processes payments |
| **Issuer** | Customer's bank that issued the card |
| **Interchange** | Fees paid between banks per transaction |
| **Authorization** | Real-time approval of transaction |
| **Settlement** | Actual movement of funds (1-3 days) |
| **Chargeback** | Customer disputes transaction, money returned |
| **3D Secure** | Additional authentication layer for online payments |
| **Tokenization** | Replacing card data with non-sensitive token |
| **PCI-DSS** | Security standard for handling card data |
