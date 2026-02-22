---
title: Wide Events / Canonical Log Lines
impact: CRITICAL
tags: logging, wide-events, canonical-log-lines, dotnet, csharp
---

## Wide Events / Canonical Log Lines

**Impact: CRITICAL**

Wide events (also called canonical log lines) are the primary pattern for request-level logging. For each request, emit **a context-rich summary event per service** at request completion. Instead of scattering 10-20 log lines throughout your controller or handler, consolidate all relevant context into one structured event emitted at the end of the request.

Wide events are the **preferred default** for request summaries, but they are not the only logging you will ever need. Long-running requests, streaming responses, background jobs with multiple phases, and multi-step workflows may require additional step-level events to provide visibility into progress. Prefer one summary event; add step events only when a single end-of-request event would leave you blind to what happened during the request.

### The Pattern

Build the event throughout the request lifecycle, then emit once at completion in a `finally` block. This ensures the event is always emitted with complete context, even during failures.

**Incorrect -- scattered logs with disconnected context:**

```csharp
app.MapPost("/articles", async (CreateArticleRequest body, HttpContext context, ILogger<Program> logger) =>
{
    logger.LogInformation("Received POST /articles request");

    logger.LogInformation("Request body parsed, Title: {Title}", body.Title);

    var user = await GetUser(context.User);
    logger.LogInformation("User fetched, UserId: {UserId}", user.Id);

    var article = await database.SaveArticle(body, user.Id);
    logger.LogInformation("Article saved, ArticleId: {ArticleId}", article.Id);

    await cache.SetAsync(article.Id, article);
    logger.LogInformation("Cache updated");

    logger.LogInformation("Request completed successfully");
    return Results.Created($"/articles/{article.Id}", article);
});
// 6 disconnected log lines with scattered context
// Cannot query: "show me all article creates by free trial users"
```

**Correct -- single wide event with all context:**

```csharp
app.MapPost("/articles", async (CreateArticleRequest body, HttpContext context, ILogger<Program> logger) =>
{
    var startTime = Stopwatch.GetTimestamp();
    var wideEvent = new Dictionary<string, object?>
    {
        ["Method"] = "POST",
        ["Path"] = "/articles",
        ["Service"] = "articles",
        ["TraceId"] = Activity.Current?.TraceId.ToString() ?? context.TraceIdentifier,
    };

    try
    {
        var user = await GetUser(context.User);
        wideEvent["User"] = new
        {
            user.Id,
            user.Subscription,
            user.Trial,
        };

        var article = await database.SaveArticle(body, user.Id);
        wideEvent["Article"] = new
        {
            article.Id,
            article.Title,
            article.Published,
        };

        await cache.SetAsync(article.Id, article);
        wideEvent["Cache"] = new { Operation = "write", Key = article.Id };

        wideEvent["StatusCode"] = 201;
        wideEvent["Outcome"] = "success";
        return Results.Created($"/articles/{article.Id}", article);
    }
    catch (Exception ex)
    {
        wideEvent["StatusCode"] = 500;
        wideEvent["Outcome"] = "error";
        wideEvent["Error"] = new { Type = ex.GetType().Name, ex.Message };

        // Pass the exception to the logger to preserve the full stack trace.
        // The structured Error property above is for querying; the exception
        // parameter is for the stack trace in the log sink.
        logger.LogError(ex, "Request failed {@WideEvent}", wideEvent);
        throw;
    }
    finally
    {
        wideEvent["DurationMs"] = Stopwatch.GetElapsedTime(startTime).TotalMilliseconds;
        wideEvent["Timestamp"] = DateTimeOffset.UtcNow;

        // Emit the summary event. In the catch block we already logged with
        // LogError (which includes the exception); in finally we emit the
        // wide event at the appropriate level for the outcome.
        var outcome = wideEvent.GetValueOrDefault("Outcome")?.ToString();
        if (outcome != "error")
        {
            logger.LogInformation("{@WideEvent}", wideEvent);
        }
    }
});
// Single event with all context - queryable by any field
```

### Exception Logging

When catching exceptions, **always pass the exception object to the logger**. The structured `Error` property on the wide event provides queryable fields (`Type`, `Message`), but the exception parameter on `LogError` is what preserves the full stack trace. Without the exception parameter, stack traces are lost and debugging becomes impossible.

```csharp
// WRONG - stack trace is lost
catch (Exception ex)
{
    wideEvent["Error"] = new { ex.Message, Type = ex.GetType().Name };
    logger.LogInformation("{@WideEvent}", wideEvent); // No exception parameter!
    throw;
}

// CORRECT - stack trace is preserved
catch (Exception ex)
{
    wideEvent["Error"] = new { Type = ex.GetType().Name, ex.Message };
    logger.LogError(ex, "Request failed {@WideEvent}", wideEvent);
    throw;
}
```

The `finally` block should use `LogInformation` for successful outcomes. Error outcomes are already logged in the `catch` block with `LogError` and the exception object, so the `finally` block can skip re-emitting them (or emit at `LogInformation` level as a summary-only record, depending on your preference). The key rule: exceptions must be logged with `LogError(ex, ...)` -- never swallow the exception parameter.

### Correlation with W3C Trace Context

.NET uses **W3C Trace Context** for distributed tracing out of the box. ASP.NET Core and `HttpClient` automatically propagate `traceparent` and `tracestate` headers across service boundaries. You do not need to manually propagate a correlation header.

Use `Activity.Current?.TraceId` as the primary correlation key in your wide events. `TraceId` is the same across all services involved in a single distributed trace, making it the right key for cross-service correlation.

```csharp
// Every wide event includes the trace ID for cross-service correlation.
// ASP.NET Core populates Activity.Current automatically from incoming
// traceparent headers. HttpClient propagates it to downstream calls.
wideEvent["TraceId"] = Activity.Current?.TraceId.ToString() ?? context.TraceIdentifier;
```

**What you do NOT need to do:**

```csharp
// WRONG - manual header propagation is unnecessary with W3C Trace Context.
// ASP.NET Core + HttpClient handle traceparent/tracestate automatically.
using var client = httpClientFactory.CreateClient();
client.DefaultRequestHeaders.Add("X-Correlation-Id", correlationId);
await client.PostAsJsonAsync("http://downstream/endpoint", data);

// CORRECT - just call the downstream service. The framework handles propagation.
using var client = httpClientFactory.CreateClient();
await client.PostAsJsonAsync("http://downstream/endpoint", data);
// traceparent header is added automatically by HttpClient
```

**`TraceId` vs `Activity.Current?.Id`:** Use `Activity.Current?.TraceId` (not `Activity.Current?.Id`). The `Id` property includes the span ID and is formatted differently depending on the `ActivityIdFormat`. `TraceId` is always a 32-character hex string that stays the same across the entire distributed trace.

### Business Correlation IDs

Sometimes you need a **business correlation ID** in addition to the trace ID. For example, an `OrderId` that ties together multiple requests over time (order created, payment processed, shipment dispatched). This is a separate concept from trace correlation -- a single business entity may span many independent traces.

```csharp
// Trace correlation (automatic, per-request, cross-service)
wideEvent["TraceId"] = Activity.Current?.TraceId.ToString();

// Business correlation (domain-specific, cross-request, long-lived)
wideEvent["OrderId"] = order.Id;

// These serve different purposes:
// - TraceId: "show me everything that happened during THIS request across all services"
// - OrderId: "show me everything that happened to THIS order across all requests"
```

### Emit in Finally Block

Always emit wide events in a `finally` block. This guarantees the event is emitted with complete context regardless of whether the request succeeded, failed with a handled error, or threw an unhandled exception. Without the `finally` block, a thrown exception skips the emission entirely, creating a blind spot for exactly the requests you most need visibility into.

### When Additional Step-Level Events Are Appropriate

Wide events are the **default and preferred** pattern for request summaries. However, there are legitimate cases where a single end-of-request event is insufficient:

- **Long-running requests** (minutes or hours): Emit progress events so operators know the request is still alive and can see where time is being spent.
- **Streaming responses**: The "end of request" may be far in the future; log when the stream opens and when it closes.
- **Background jobs and queue consumers**: A single job may process many items. Log a summary per batch or per item in addition to the overall job summary.
- **Multi-phase workflows**: A saga or orchestration that calls multiple downstream services in sequence may benefit from a per-phase event, especially when individual phases can fail independently.
- **Retry loops**: Log each retry attempt so you can see how many retries occurred and what changed between them.

In all these cases, the wide event remains the **primary** summary. Step-level events are supplementary. Tag step events with the same `TraceId` so they can be correlated with the summary.

### Use a Typed Wide Event Class (Optional)

For larger services, a strongly-typed class provides compile-time safety and makes the event schema explicit. If you use a typed class, give its nested properties real types -- avoid `object?` which loses type safety and defeats the purpose.

```csharp
public sealed class RequestWideEvent
{
    public required string Method { get; init; }
    public required string Path { get; init; }
    public required string TraceId { get; init; }
    public DateTimeOffset Timestamp { get; set; } = DateTimeOffset.UtcNow;
    public int StatusCode { get; set; }
    public string Outcome { get; set; } = "unknown";
    public double DurationMs { get; set; }

    public UserContext? User { get; set; }
    public ErrorContext? Error { get; set; }

    public sealed class UserContext
    {
        public required string Id { get; init; }
        public string? Subscription { get; init; }
        public bool? Trial { get; init; }
    }

    public sealed class ErrorContext
    {
        public required string Type { get; init; }
        public required string Message { get; init; }
    }
}
```

The dictionary-based approach (`Dictionary<string, object?>`) is also valid and is more flexible for ad-hoc enrichment. Choose based on your team's preference: typed classes for well-known schemas, dictionaries for maximum flexibility. Both are fine.

### Size and Shape Controls

Wide events should log **selected fields** from domain objects, not entire entity graphs. Uncontrolled destructuring is a common source of bloated logs, leaked sensitive data, and serialization failures.

**Log projections, not entities:**

```csharp
// WRONG - logs the entire entity with all navigation properties
wideEvent["Order"] = order; // May include Customer, LineItems, each with their own relations...

// CORRECT - log only the fields you need for querying and debugging
wideEvent["Order"] = new
{
    order.Id,
    order.Status,
    order.TotalCents,
    ItemCount = order.LineItems.Count,
};
```

**Risks of logging full objects:**

- **Navigation properties and lazy loading**: EF Core entities may have navigation properties (`order.Customer.Orders.First().Customer...`) that trigger lazy loading or produce circular references, causing serialization failures or massive log entries.
- **Circular references**: Serilog's destructuring will throw or produce truncated output when it encounters circular object graphs.
- **Sensitive data leakage**: Full entity graphs may include passwords, tokens, PII, or payment details that should never appear in logs.
- **Log storage costs**: A single wide event that accidentally serializes a deep object graph can be megabytes, blowing up log storage and ingestion costs.

**Set destructuring depth limits in Serilog:**

```csharp
Log.Logger = new LoggerConfiguration()
    .Destructure.ToMaximumDepth(3)                // Limit nesting depth
    .Destructure.ToMaximumStringLength(1024)       // Limit string field length
    .Destructure.ToMaximumCollectionCount(10)       // Limit array/collection length
    .CreateLogger();
```

These limits act as safety nets. Even with limits configured, prefer explicit projections (`new { order.Id, order.Status }`) over passing full objects. The limits catch mistakes; they are not a substitute for intentional field selection.

### Serilog's Built-In Request Logging

If you use Serilog, consider `app.UseSerilogRequestLogging()` as an alternative or complement to a custom wide event middleware. It automatically captures timing, HTTP status, request method, and path for every request, and emits a single summary event.

```csharp
// Program.cs
app.UseSerilogRequestLogging(options =>
{
    // Enrich the automatic request log event with additional properties
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("UserId", httpContext.User.FindFirst("sub")?.Value);
        diagnosticContext.Set("RequestId", httpContext.TraceIdentifier);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers.UserAgent.ToString());
    };
});
```

You can also enrich the event from within your handlers using `IDiagnosticContext`:

```csharp
app.MapPost("/checkout", async (HttpContext context, IDiagnosticContext diagnosticContext) =>
{
    var user = await GetUser(context.User);
    diagnosticContext.Set("User", new { user.Id, user.Subscription });

    var cart = await GetCart(user.Id);
    diagnosticContext.Set("Cart", new { cart.Id, ItemCount = cart.Items.Count, cart.TotalCents });

    // Serilog emits the request completion event with all enriched properties
    return Results.Ok();
});
```

**Trade-offs:** `UseSerilogRequestLogging` is simpler to set up and covers the common case well. A custom middleware gives you full control over the event shape, lets you use a typed class or dictionary, and works with any logging provider (not just Serilog). The custom middleware pattern shown elsewhere in this skill is the more general approach; `UseSerilogRequestLogging` is a convenient shortcut when Serilog is your provider.

Reference: [Stripe Blog - Canonical Log Lines](https://stripe.com/blog/canonical-log-lines)
