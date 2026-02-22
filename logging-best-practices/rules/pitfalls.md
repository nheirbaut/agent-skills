---
title: Common Pitfalls
impact: MEDIUM
tags: logging, anti-patterns, pitfalls, dotnet, csharp
---

## Common Pitfalls

**Impact: MEDIUM**

Avoid these anti-patterns that undermine your logging effectiveness.

### Pitfall 1: Unnecessary Scattered Logs

Prefer a single summary event per request. The anti-pattern is scattering `_logger.LogInformation()` calls throughout a handler when all that context belongs in one wide event emitted at completion.

This does **not** mean "never emit more than one log entry." Long-running operations, streaming responses, background jobs with multiple phases, and retry loops may legitimately need intermediate log entries to provide visibility into progress. The problem is unnecessary fragmentation -- five log lines that each carry one field, when a single wide event could carry all five fields together.

**Anti-pattern -- scattered logs that each carry one piece of context:**

```csharp
app.MapPost("/checkout", async (HttpContext context, ILogger<Program> logger) =>
{
    logger.LogInformation("Received checkout request");                      // Line 1
    logger.LogInformation("User ID: {UserId}", context.User.FindFirst("sub")?.Value); // Line 2
    var user = await GetUser(context.User);
    logger.LogInformation("User fetched: {Email}", user.Email);             // Line 3
    var cart = await GetCart(user.Id);
    logger.LogInformation("Cart fetched: {ItemCount} items", cart.Items.Count); // Line 4
    var payment = await ProcessPayment(cart);
    logger.LogInformation("Payment processed: {Status}", payment.Status);   // Line 5
    logger.LogInformation("Checkout completed successfully");               // Line 6
    return Results.Ok(new { payment.OrderId });
});
// 6 log lines per request = noise. No single event has the full picture.
// Cannot query: "show me all failed payments for premium users with carts > $100"
```

**Correct -- single wide event with everything:**

```csharp
// Single wide event with everything (see wide-events.md for the full pattern)
var wideEvent = new Dictionary<string, object?>
{
    ["Method"] = "POST",
    ["Path"] = "/checkout",
    ["User"] = new { user.Id, user.Subscription },
    ["Cart"] = new { ItemCount = cart.Items.Count, TotalCents = cart.Total },
    ["Payment"] = new { payment.Status, payment.OrderId },
    ["StatusCode"] = 200,
    ["DurationMs"] = 1247.0,
};
logger.LogInformation("{@WideEvent}", wideEvent);
```

**When multiple log entries are legitimate:**

- A background job that processes 10,000 records over 30 minutes should log progress (e.g., per-batch summaries) so operators can see it is still alive and how far along it is.
- A retry loop should log each attempt so you can see how many retries occurred and what changed between them.
- A streaming response should log when the stream opens and when it closes.
- A saga or multi-phase workflow may log per-phase events when individual phases can fail independently.

In all these cases, the wide event remains the **primary** summary. Step-level events are supplementary, not replacements. See `wide-events.md` for details on when additional step-level events are appropriate.

### Pitfall 2: Not Designing for Unknown Unknowns

Traditional logging captures "known unknowns" -- issues you anticipated. But production bugs are often "unknown unknowns" -- issues you never predicted. Wide events with rich context enable investigating issues you didn't anticipate.

**Anti-pattern -- logging only for anticipated issues:**

```csharp
// Logging only for anticipated issues
app.MapPost("/articles", async (CreateArticleRequest body, HttpContext context, ILogger<Program> logger) =>
{
    var article = await CreateArticle(body, user);
    if (!article.Published)
    {
        logger.LogInformation("Article created but not published"); // Anticipated issue
    }
    return Results.Ok(new { article });
});

// Bug: "Users on free trial can't see their articles"
// Your logs say: "Article created successfully"
// But you have NO visibility into:
// - Which users are affected (free trial? all?)
// - What subscription plans see this issue
// - When it started
```

**Correct -- wide event captures everything:**

```csharp
// Wide event captures everything -- even fields you don't think you need yet
wideEvent["User"] = new
{
    user.Id,
    user.Subscription,
    user.Trial,
    user.TrialExpiration,
};

wideEvent["Article"] = new
{
    article.Id,
    article.Published, // Captured even though we didn't anticipate the bug
};

// Now you can query: WHERE Article.Published = false GROUP BY User.Trial
// Result: 95% of unpublished articles are from trial users!
```

The wide event pattern lets you answer questions you didn't know to ask. See `context.md` for guidance on which dimensions and cardinalities to include.

### Pitfall 3: Losing Exception Stack Traces

When logging exceptions, the exception object **must** be the first parameter to `LogError` (or `LogWarning`, `LogCritical`). This is what preserves the full stack trace, inner exceptions, and enables exception-specific enrichment by sinks like Serilog, Application Insights, and Seq. Embedding exception details in the wide event dictionary as properties is fine for queryable fields, but it does not replace passing the exception to the logger.

**Anti-pattern -- exception details embedded but not passed to the logger:**

```csharp
catch (Exception ex)
{
    wideEvent["Error"] = new { ex.Message, Type = ex.GetType().Name };
    logger.LogInformation("{@WideEvent}", wideEvent); // Stack trace is LOST
    throw;
}
// The log sink receives a structured event with Error.Message and Error.Type,
// but the actual stack trace, inner exceptions, and exception type hierarchy
// are gone. Debugging in production becomes guesswork.
```

**Correct -- pass the exception as the first parameter:**

```csharp
catch (Exception ex)
{
    wideEvent["Error"] = new { Type = ex.GetType().Name, ex.Message };
    logger.LogError(ex, "Request failed {@WideEvent}", wideEvent);
    throw;
}
// The exception is the first parameter to LogError.
// Serilog, Application Insights, and Seq all receive the full exception:
// - Complete stack trace
// - Inner exceptions (AggregateException, etc.)
// - Exception type hierarchy for filtering
// - Serilog's Exceptions enricher can extract demystified stack traces
//
// The structured Error property on the wide event provides queryable fields.
// The exception parameter provides the full diagnostic payload.
```

See `wide-events.md` for the complete `try/catch/finally` pattern that handles both the wide event emission and exception logging correctly.

### Pitfall 4: Logging Sensitive Data

Log events are stored in log aggregation systems, often with broad read access across engineering teams, and sometimes retained for months or years. Logging sensitive data creates compliance violations (GDPR, CCPA, PCI-DSS, HIPAA) and security risks.

**Anti-pattern -- accidentally logging PII, secrets, and payment data:**

```csharp
// Logging raw email addresses
wideEvent["User"] = new { user.Id, user.Email, user.PhoneNumber };

// Logging authentication tokens
wideEvent["Auth"] = new { Token = context.Request.Headers.Authorization.ToString() };

// Logging connection strings (may contain passwords)
_logger.LogError("Database connection failed: {ConnectionString}", connectionString);

// Logging payment card details
wideEvent["Payment"] = new { card.Number, card.Cvv, card.ExpiryDate };

// Logging full request bodies that may contain passwords
wideEvent["RequestBody"] = await new StreamReader(context.Request.Body).ReadToEndAsync();
```

**Correct -- mask, hash, or omit sensitive fields:**

```csharp
// Hash email for correlation without exposing PII
wideEvent["User"] = new
{
    user.Id,
    EmailHash = Convert.ToHexString(SHA256.HashData(
        Encoding.UTF8.GetBytes(user.Email.ToLowerInvariant())))[..16],
    // First 16 hex chars is enough for correlation, not reversible
};

// Log token metadata, not the token itself
wideEvent["Auth"] = new
{
    Scheme = context.Request.Headers.Authorization.ToString().Split(' ')[0],
    HasToken = !string.IsNullOrEmpty(context.Request.Headers.Authorization),
};

// Log connection target, not the connection string
wideEvent["Database"] = new
{
    Server = new SqlConnectionStringBuilder(connectionString).DataSource,
    Database = new SqlConnectionStringBuilder(connectionString).InitialCatalog,
};

// Log card metadata only
wideEvent["Payment"] = new
{
    CardLast4 = card.Number[^4..],
    CardBrand = card.Brand,
};
```

**What to watch for:**

- **Email addresses, phone numbers, IP addresses**: PII under GDPR/CCPA. Hash or omit.
- **Authorization headers, API keys, JWT tokens**: Secrets. Log presence/scheme only.
- **Connection strings**: May contain passwords. Extract host/database name only.
- **Credit card numbers, CVVs**: PCI-DSS violation. Log last 4 digits and brand at most.
- **Request/response bodies**: May contain passwords, personal data, or health information. Log specific extracted fields, never raw bodies.
- **User-agent strings with device identifiers**: Can be fingerprinting data under privacy regulations.

**Serilog destructuring policies can act as a safety net:**

```csharp
// Mask sensitive properties globally via a destructuring policy
public class SensitiveDataDestructuringPolicy : IDestructuringPolicy
{
    private static readonly HashSet<string> SensitiveProperties = new(StringComparer.OrdinalIgnoreCase)
    {
        "Password", "Secret", "Token", "ApiKey", "ConnectionString",
        "CreditCard", "Cvv", "Ssn", "Authorization",
    };

    public bool TryDestructure(
        object value,
        ILogEventPropertyValueFactory propertyValueFactory,
        [NotNullWhen(true)] out LogEventPropertyValue? result)
    {
        // Implementation: inspect properties and mask sensitive ones
        result = null;
        return false;
    }
}

Log.Logger = new LoggerConfiguration()
    .Destructure.With<SensitiveDataDestructuringPolicy>()
    .CreateLogger();
```

A destructuring policy is a safety net, not a substitute for intentional field selection. Prefer explicit projections (`new { user.Id, user.Subscription }`) over logging full objects and relying on the policy to strip sensitive fields.

### Pitfall 5: Unbounded Object Destructuring

Logging full entity objects -- especially EF Core entities with navigation properties -- is a common source of bloated logs, serialization failures, circular reference exceptions, and accidental sensitive data leakage.

**Anti-pattern -- logging full entities:**

```csharp
// Logging the full EF Core entity
wideEvent["Order"] = order;
// This may pull in:
// - order.Customer (navigation property)
// - order.Customer.Orders (circular reference!)
// - order.LineItems (collection with 500 items)
// - order.LineItems[0].Product.Category.Products... (deep graph)
// - order.Customer.Email, order.Customer.PhoneNumber (PII!)
//
// Results: serialization failure, multi-megabyte log entry,
// or Serilog silently truncating at depth limit

// Logging an entire request object
wideEvent["Request"] = requestDto;
// May contain nested objects, large collections, or sensitive fields
// you didn't realize were there
```

**Correct -- log explicit projections with only the fields you need:**

```csharp
// Project only the fields relevant for debugging and querying
wideEvent["Order"] = new
{
    order.Id,
    order.Status,
    order.TotalCents,
    ItemCount = order.LineItems.Count,
    order.CreatedAt,
};

// For the customer, log only non-sensitive identifiers
wideEvent["Customer"] = new
{
    order.Customer.Id,
    order.Customer.Subscription,
};
```

**Why this matters:**

- **Navigation properties and lazy loading**: EF Core entities may trigger lazy loading when serialized, causing unexpected database queries during log emission. A single `wideEvent["Order"] = order` can execute dozens of SQL queries.
- **Circular references**: `Order.Customer.Orders[0].Customer...` causes `JsonException` or infinite recursion. Serilog's destructuring will throw or produce truncated output.
- **Log storage costs**: A single wide event that accidentally serializes a deep object graph can be megabytes, blowing up log ingestion costs.
- **Sensitive data leakage**: Full entity graphs may include fields you didn't realize were there -- passwords, tokens, PII (see Pitfall 4).

**Configure Serilog destructuring limits as a safety net:**

```csharp
Log.Logger = new LoggerConfiguration()
    .Destructure.ToMaximumDepth(3)                // Limit nesting depth
    .Destructure.ToMaximumStringLength(1024)       // Limit string field length
    .Destructure.ToMaximumCollectionCount(10)       // Limit array/collection length
    .CreateLogger();
```

These limits catch mistakes; they are not a substitute for intentional field selection. Always prefer explicit projections (`new { order.Id, order.Status }`) over passing full objects, even when limits are configured. See `wide-events.md` for more on size and shape controls.

### Pitfall 6: String Interpolation in Log Message Templates

This is a .NET-specific pitfall. Using `$"..."` string interpolation in log calls defeats structured logging entirely. The log provider cannot extract individual field values for querying -- it receives one pre-formatted string instead of a template with named parameters.

**Anti-pattern -- string interpolation:**

```csharp
// String interpolation - Serilog sees ONE big string, not individual fields
_logger.LogInformation($"User {user.Id} placed order {order.Id} for {cart.Total} cents");
// Stored as: "User user_456 placed order ord_123 for 15999 cents"
// Cannot query by UserId, OrderId, or Total separately

// Also wrong - concatenation
_logger.LogInformation("User " + user.Id + " placed order " + order.Id);
```

**Correct -- message templates with named placeholders:**

```csharp
// Message template - Serilog extracts each field as a queryable property
_logger.LogInformation(
    "User {UserId} placed order {OrderId} for {TotalCents} cents",
    user.Id, order.Id, cart.Total);
// Stored with structured fields: UserId=user_456, OrderId=ord_123, TotalCents=15999
// Queryable: WHERE UserId = 'user_456' AND TotalCents > 10000
```

**Why this matters beyond querying:**
- The `[LoggerMessage]` source generator requires message templates (it cannot work with interpolated strings).
- Log providers skip string formatting entirely when the log level is disabled. With interpolation, the string is always allocated, even when the level is filtered out.
- Analyzers like `Serilog.Analyzers` and `Microsoft.Extensions.Logging.Analyzers` can validate message templates at compile time, catching mismatched parameter counts before runtime.

See `structure.md` for more on structured logging with message templates, `BeginScope`, and the `{@}` destructuring operator.

### Pitfall 7: Using Console.WriteLine or Debug.WriteLine

`Console.WriteLine` bypasses the entire logging pipeline -- no structured data, no log levels, no sinks, no filtering, no enrichment. `Debug.WriteLine` is stripped entirely in Release builds by the compiler.

**Anti-pattern:**

```csharp
Console.WriteLine($"Processing order {orderId}");  // No structure, no sink routing
Debug.WriteLine($"Cart total: {total}");            // Gone in Release builds
System.Diagnostics.Trace.WriteLine("Request done"); // Legacy, no structure
```

**Correct:**

```csharp
// All logging through ILogger<T>
_logger.LogInformation("Processing order {OrderId}", orderId);
// Or better yet, add it to the wide event:
wideEvent["Order"] = new { Id = orderId, Total = total };
```

If you encounter `Console.WriteLine` in existing code, replace it with `ILogger<T>` and add the relevant data to the wide event. The only acceptable use of `Console.WriteLine` in a production .NET application is during startup, before the logging pipeline is configured (e.g., printing a fatal configuration error that prevents the host from building).

### Pitfall 8: Missing or Manual Request Correlation

Without correlation, you cannot trace a request's journey across services. In .NET, the framework handles this for you -- but only if you use it correctly.

**Anti-pattern -- no correlation at all:**

```csharp
// Service A logs
logger.LogInformation("Order created, OrderId: {OrderId}", "ord_123");

// Service B logs
logger.LogInformation("Inventory reserved, Items: {Items}", 3);

// No way to connect these two events!
```

**Anti-pattern -- manual header propagation:**

```csharp
// WRONG - manual correlation header propagation is unnecessary
var correlationId = Guid.NewGuid().ToString();
using var client = httpClientFactory.CreateClient();
client.DefaultRequestHeaders.Add("X-Correlation-Id", correlationId);
await client.PostAsJsonAsync("http://inventory/reserve", items);

// Service B reads:
var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault();
```

This manual approach is fragile, easy to forget, and reinvents what the framework already provides.

**Correct -- use `Activity.Current?.TraceId` with automatic propagation:**

```csharp
// Include TraceId in your wide event
wideEvent["TraceId"] = Activity.Current?.TraceId.ToString() ?? context.TraceIdentifier;

// When calling downstream services, just call them normally.
// ASP.NET Core + HttpClient automatically propagate W3C Trace Context
// (traceparent/tracestate headers) across service boundaries.
using var client = httpClientFactory.CreateClient();
await client.PostAsJsonAsync("http://inventory/reserve", items);
// traceparent header is added automatically -- no manual propagation needed
```

**Key points:**

- Use `Activity.Current?.TraceId`, not `Activity.Current?.Id`. The `Id` property includes the span ID and is formatted differently depending on the `ActivityIdFormat`. `TraceId` is always a 32-character hex string that stays the same across the entire distributed trace.
- ASP.NET Core automatically creates an `Activity` for each incoming request and populates it from the incoming `traceparent` header.
- `HttpClient` automatically propagates the `traceparent` header to outgoing requests (via `DiagnosticsHandler`).
- You do not need custom middleware, custom headers, or manual propagation. The framework handles W3C Trace Context end-to-end.
- For business-level correlation (e.g., an `OrderId` that spans multiple requests over days), add a domain-specific correlation property to your wide events. This is a separate concept from trace correlation. See `wide-events.md` for details on business correlation IDs.
