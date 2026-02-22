---
title: Structure and Format
impact: HIGH
tags: logging, structured-logging, serilog, ilogger, middleware, dotnet, csharp
---

## Structure and Format

**Impact: HIGH**

Structured logging with consistent formats enables efficient querying and analysis. The right structure transforms logs from text files into queryable data.

### Use ILogger<T> via Dependency Injection

Use .NET's built-in `ILogger<T>` resolved through dependency injection. Configure your logging provider (Serilog recommended) once at startup in `Program.cs`. Never instantiate loggers manually, and never use `Console.WriteLine` for operational logging.

```csharp
// Program.cs - Configure Serilog once at startup
using Serilog;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithProperty("Service", Environment.GetEnvironmentVariable("SERVICE_NAME"))
    .Enrich.WithProperty("Version", Environment.GetEnvironmentVariable("SERVICE_VERSION"))
    .Enrich.WithProperty("CommitHash", Environment.GetEnvironmentVariable("COMMIT_SHA"))
    .Enrich.WithProperty("Region", Environment.GetEnvironmentVariable("AZURE_REGION"))
    .Enrich.WithProperty("Environment", Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT"))
    .WriteTo.Console(new RenderedCompactJsonFormatter())
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog();

// Usage everywhere else - just inject ILogger<T>
public class CheckoutService
{
    private readonly ILogger<CheckoutService> _logger;

    public CheckoutService(ILogger<CheckoutService> logger)
    {
        _logger = logger;
    }

    public async Task<Order> ProcessCheckout(Cart cart)
    {
        // Use structured logging with message templates
        _logger.LogInformation("Checkout completed for cart {CartId} with total {CartTotal}",
            cart.Id, cart.Total);
        return order;
    }
}
```

**Benefits:**
- Consistent log format across all classes
- Environment context automatically included via Serilog enrichers
- Single place to change log level or destination
- Testable (inject `NullLogger<T>` or `FakeLogger<T>` in tests)

**Avoid:**
```csharp
// DON'T create loggers manually
var logger = new LoggerConfiguration().CreateLogger(); // Each class creates its own
Console.WriteLine("some event");                       // Bypasses the logger entirely
Debug.WriteLine("some event");                         // Not available in Release builds
```

### Use Proper Log Levels

Use log levels that reflect the operational significance of each event. The correct levels for production .NET applications:

| Level | When to Use | Example |
|-------|-------------|---------|
| **Information** | Normal operational flow. Request summaries, successful operations, startup/shutdown. | `"Request completed {Method} {Path} in {DurationMs}ms"` |
| **Warning** | Degraded but functional. The operation succeeded via a fallback, retry, or with reduced quality. | `"Payment retry succeeded on attempt {Attempt}"`, `"Rate limit approaching for tenant {TenantId}"`, `"Fallback cache used, primary cache unavailable"` |
| **Error** | A specific operation failed. The request could not be fulfilled, a dependency call failed, or data was invalid. | `"Payment processing failed for order {OrderId}"`, `"Database query timed out after {TimeoutMs}ms"` |
| **Critical** | System integrity is threatened. The application may need to shut down, data corruption is possible, or a critical dependency is completely unavailable. | `"Database connection pool exhausted"`, `"Certificate expiry imminent"`, `"Unrecoverable state in {Component}"` |

**Debug and Trace** are opt-in diagnostics. They are disabled by default in production via `MinimumLevel` configuration. Use them during development for detailed internal state, but do not rely on them for operational visibility. If you find yourself wanting `Debug` logs, consider adding that context to your wide event instead.

```csharp
// Correct level usage
_logger.LogInformation("Order {OrderId} created for user {UserId}", order.Id, user.Id);
_logger.LogWarning("Retry succeeded on attempt {Attempt} for {OrderId}", attempt, order.Id);
_logger.LogError(ex, "Payment failed for order {OrderId}", order.Id);
_logger.LogCritical("Connection pool exhausted, service degraded");
```

**Avoid the "only Information and Error" trap.** Warning exists for a reason: a retry that succeeded is not an error, but it is a signal of degradation you want to alert on. Critical exists because not all errors threaten the system - but some do, and those need separate alerting thresholds.

### Use Serilog Request Logging for HTTP Workloads

For ASP.NET Core HTTP applications, prefer Serilog's built-in `UseSerilogRequestLogging()` over custom middleware. It produces a single summary event per request with timing, status code, and path - exactly the "one event per request" pattern.

```csharp
// Program.cs
var app = builder.Build();

app.UseSerilogRequestLogging(options =>
{
    // Attach additional properties to the request completion event
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers.UserAgent.ToString());
        diagnosticContext.Set("ClientIp", httpContext.Connection.RemoteIpAddress?.ToString());
    };

    // Customise the message template if needed
    options.MessageTemplate =
        "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";

    // Control the log level based on status code or exception
    options.GetLevel = (httpContext, elapsed, ex) =>
    {
        if (ex is not null || httpContext.Response.StatusCode >= 500)
            return Serilog.Events.LogEventLevel.Error;
        if (httpContext.Response.StatusCode >= 400)
            return Serilog.Events.LogEventLevel.Warning;
        return Serilog.Events.LogEventLevel.Information;
    };
});
```

**Enrich from handlers using `IDiagnosticContext`:**

Inject `IDiagnosticContext` into your endpoint handlers or controllers to attach business context to the same request summary event that `UseSerilogRequestLogging` emits.

```csharp
app.MapPost("/checkout", async (
    HttpContext context,
    IDiagnosticContext diagnosticContext,
    CheckoutService checkoutService) =>
{
    var user = context.Features.Get<UserFeature>()!.User;
    diagnosticContext.Set("UserId", user.Id);
    diagnosticContext.Set("Subscription", user.Subscription);

    var cart = await checkoutService.GetCart(user.Id);
    diagnosticContext.Set("CartId", cart.Id);
    diagnosticContext.Set("CartTotal", cart.Total);

    var order = await checkoutService.CreateOrder(cart);
    diagnosticContext.Set("OrderId", order.Id);

    return Results.Created($"/orders/{order.Id}", order);
});
// Serilog emits ONE event at request completion with all enriched properties
```

**Alternative: Custom middleware for full control**

If you need the dictionary-based wide event pattern (e.g., deeply nested properties, non-Serilog providers, or full control over the event shape), use custom middleware. Note the correct log level for error cases:

```csharp
public sealed class WideEventMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<WideEventMiddleware> _logger;

    public WideEventMiddleware(RequestDelegate next, ILogger<WideEventMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var startTime = Stopwatch.GetTimestamp();
        var wideEvent = new Dictionary<string, object?>
        {
            ["RequestId"] = context.TraceIdentifier,
            ["Timestamp"] = DateTimeOffset.UtcNow,
            ["Method"] = context.Request.Method,
            ["Path"] = context.Request.Path.Value,
            ["UserAgent"] = context.Request.Headers.UserAgent.ToString(),
        };

        context.Items["WideEvent"] = wideEvent;
        Exception? caughtException = null;

        try
        {
            await _next(context);
            wideEvent["StatusCode"] = context.Response.StatusCode;
            wideEvent["Outcome"] = context.Response.StatusCode < 400 ? "success" : "error";
        }
        catch (Exception ex)
        {
            caughtException = ex;
            wideEvent["StatusCode"] = 500;
            wideEvent["Outcome"] = "error";
            wideEvent["Error"] = new
            {
                Type = ex.GetType().Name,
                ex.Message,
            };
            throw;
        }
        finally
        {
            wideEvent["DurationMs"] = Stopwatch.GetElapsedTime(startTime).TotalMilliseconds;

            if (caughtException is not null)
            {
                _logger.LogError(caughtException,
                    "Request failed {Method} {Path}", 
                    context.Request.Method, context.Request.Path);
            }
            else if (context.Response.StatusCode >= 500)
            {
                _logger.LogError("Request completed with server error {@WideEvent}", wideEvent);
            }
            else if (context.Response.StatusCode >= 400)
            {
                _logger.LogWarning("Request completed with client error {@WideEvent}", wideEvent);
            }
            else
            {
                _logger.LogInformation("Request completed {@WideEvent}", wideEvent);
            }
        }
    }
}
```

### Use an Extension Method for Ergonomic Access

When using the custom middleware approach, provide an extension method so handlers can access the wide event cleanly:

```csharp
public static class HttpContextExtensions
{
    private const string WideEventKey = "WideEvent";

    public static Dictionary<string, object?> GetWideEvent(this HttpContext context)
        => (Dictionary<string, object?>)context.Items[WideEventKey]!;
}

// Usage
var wideEvent = context.GetWideEvent();
wideEvent["User"] = new { user.Id, user.Subscription };
```

### Use Structured Logging with Message Templates

Use structured logging with message templates. Never use string interpolation (`$"..."`) in log message templates - it defeats structured logging by turning queryable fields into a single opaque string.

```csharp
// CORRECT - structured, queryable
_logger.LogInformation("Order {OrderId} created for {UserId}", order.Id, user.Id);

// WRONG - string interpolation destroys structure
_logger.LogInformation($"Order {order.Id} created for {user.Id}");
// The provider cannot extract OrderId/UserId as separate queryable fields
```

**The `{@}` destructuring operator is Serilog-specific.** The `@` prefix tells Serilog to serialize the object as a structured object rather than calling `.ToString()`. The standard `Microsoft.Extensions.Logging` providers do not recognise `@` and will fall back to `.ToString()`.

```csharp
// Serilog-specific: @ means "destructure this object"
_logger.LogInformation("Checkout completed {@Order}", new { cart.Id, cart.Total });

// Provider-agnostic alternative: use BeginScope for structured properties
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["CartId"] = cart.Id,
    ["CartTotal"] = cart.Total,
}))
{
    _logger.LogInformation("Checkout completed");
}
```

If your project might switch logging providers, prefer `BeginScope` with explicit properties over `{@}` destructuring. If you are committed to Serilog, `{@}` is concise and idiomatic.

### Use BeginScope for Request-Scoped Properties

`ILogger.BeginScope()` is the provider-agnostic way to attach properties that appear on every log entry within that scope. Use it for correlation IDs, user context, or any property that should appear on all log entries during a unit of work.

```csharp
// Provider-agnostic: works with any ILogger provider
public async Task<IActionResult> ProcessOrder(OrderRequest request)
{
    using (_logger.BeginScope(new Dictionary<string, object>
    {
        ["OrderId"] = request.OrderId,
        ["UserId"] = request.UserId,
        ["CorrelationId"] = Activity.Current?.Id ?? HttpContext.TraceIdentifier,
    }))
    {
        _logger.LogInformation("Starting order processing");
        // All log entries within this block include OrderId, UserId, CorrelationId

        await ValidateInventory(request);
        await ProcessPayment(request);
        await ConfirmOrder(request);

        _logger.LogInformation("Order processing completed");
        return Ok();
    }
}
```

**Serilog-specific alternative: `LogContext.PushProperty()`**

If you are using Serilog and have `.Enrich.FromLogContext()` in your configuration, you can use `LogContext.PushProperty()` for the same effect with a more concise API:

```csharp
using Serilog.Context;

// Serilog-specific: requires Enrich.FromLogContext() in configuration
using (LogContext.PushProperty("OrderId", request.OrderId))
using (LogContext.PushProperty("UserId", request.UserId))
{
    _logger.LogInformation("Starting order processing");
    // OrderId and UserId appear on every log entry within this block
}
```

Both approaches achieve the same result. Use `BeginScope` for portability, `LogContext.PushProperty` if you are committed to Serilog.

### Use LoggerMessage Source Generators for Hot Paths

For high-throughput logging (request middleware, tight loops, high-frequency background work), use the `[LoggerMessage]` attribute to generate high-performance logging methods at compile time. This eliminates boxing of value types, avoids parsing the message template on every call, and skips string formatting entirely when the log level is disabled.

```csharp
public partial class CheckoutService
{
    private readonly ILogger<CheckoutService> _logger;

    public CheckoutService(ILogger<CheckoutService> logger)
    {
        _logger = logger;
    }

    // Source generator creates the implementation at compile time
    [LoggerMessage(
        EventId = 1000,
        Level = LogLevel.Information,
        Message = "Order {OrderId} created for user {UserId} with total {TotalCents}")]
    private partial void LogOrderCreated(string orderId, string userId, long totalCents);

    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Warning,
        Message = "Payment retry succeeded on attempt {Attempt} for order {OrderId}")]
    private partial void LogPaymentRetry(int attempt, string orderId);

    [LoggerMessage(
        EventId = 1002,
        Level = LogLevel.Error,
        Message = "Payment failed for order {OrderId}")]
    private partial void LogPaymentFailed(string orderId);

    public async Task<Order> ProcessCheckout(Cart cart, User user)
    {
        var order = await CreateOrder(cart, user);
        LogOrderCreated(order.Id, user.Id, cart.TotalCents);
        return order;
    }
}
```

**Why use source generators:**
- **Zero boxing**: Value types (`int`, `long`, `double`) are not boxed into `object[]`
- **Compile-time template parsing**: The message template is parsed once at compile time, not on every log call
- **Level-check guard**: The generated method checks `IsEnabled(LogLevel)` before doing any work
- **Consistent EventIds**: Forces you to assign stable event IDs (see EventId guidance below)

Use source generators for code that runs on every request or in tight loops. For infrequent business-logic logging, the standard `_logger.LogInformation(...)` extension methods are fine.

### Assign Stable EventIds for Key Events

Assign meaningful, stable `EventId` values (or event names) to key operational events. This makes it possible to build dashboards, alerts, and queries that survive message template changes.

```csharp
// Define event IDs in a central location per service or feature area
public static class LogEvents
{
    // Request lifecycle
    public static readonly EventId RequestCompleted = new(1000, "RequestCompleted");
    public static readonly EventId RequestFailed = new(1001, "RequestFailed");

    // Authentication
    public static readonly EventId AuthFailure = new(2000, "AuthFailure");
    public static readonly EventId TokenExpired = new(2001, "TokenExpired");

    // Dependencies
    public static readonly EventId DependencyCallFailed = new(3000, "DependencyCallFailed");
    public static readonly EventId DependencyCallSlow = new(3001, "DependencyCallSlow");
    public static readonly EventId CircuitBreakerTripped = new(3002, "CircuitBreakerTripped");

    // Business operations
    public static readonly EventId OrderCreated = new(4000, "OrderCreated");
    public static readonly EventId PaymentProcessed = new(4001, "PaymentProcessed");
    public static readonly EventId PaymentFailed = new(4002, "PaymentFailed");
}

// Usage with standard ILogger
_logger.LogInformation(LogEvents.RequestCompleted,
    "Request completed {Method} {Path} in {DurationMs}ms",
    method, path, durationMs);

_logger.LogError(LogEvents.DependencyCallFailed, ex,
    "Call to {DependencyName} failed after {DurationMs}ms",
    dependencyName, durationMs);
```

**Benefits of stable EventIds:**
- Alerts and dashboards reference EventId `3000` instead of a fragile message substring
- Log queries like `WHERE EventId = 2000` survive template rewording
- When using `[LoggerMessage]` source generators, EventId is a required parameter, reinforcing this practice

### Non-HTTP Workloads: Worker Services, Message Handlers, Background Jobs

Not all .NET applications are HTTP APIs. For Worker Services (`IHostedService`), message consumers, and background jobs, the same logging principles apply, but the mechanics differ.

**Use `Activity` for correlation instead of `HttpContext`:**

```csharp
public class OrderMessageHandler
{
    private readonly ILogger<OrderMessageHandler> _logger;

    public OrderMessageHandler(ILogger<OrderMessageHandler> logger)
    {
        _logger = logger;
    }

    public async Task HandleAsync(OrderMessage message, CancellationToken ct)
    {
        // Start an Activity for distributed tracing correlation
        using var activity = new ActivitySource("MyService").StartActivity("ProcessOrder");
        activity?.SetTag("order.id", message.OrderId);

        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderId"] = message.OrderId,
            ["CorrelationId"] = message.CorrelationId,
            ["MessageType"] = nameof(OrderMessage),
        }))
        {
            var startTime = Stopwatch.GetTimestamp();

            try
            {
                await ProcessOrder(message, ct);

                _logger.LogInformation("Order message processed successfully");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Order message processing failed");
                throw;
            }
            finally
            {
                var durationMs = Stopwatch.GetElapsedTime(startTime).TotalMilliseconds;
                _logger.LogInformation("Order processing duration {DurationMs}ms", durationMs);
            }
        }
    }
}
```

**Use scope-based context instead of `HttpContext.Items`:**

In HTTP middleware you can store the wide event in `HttpContext.Items`. In non-HTTP workloads, use `ILogger.BeginScope()` or `LogContext.PushProperty()` to attach contextual properties. Pass a context object explicitly if multiple methods need to enrich it.

**Emit a job summary event:**

For background jobs and message handlers, emit a summary event at the end of each unit of work, just as you would for an HTTP request:

```csharp
public class DailyReportJob : BackgroundService
{
    private readonly ILogger<DailyReportJob> _logger;

    public DailyReportJob(ILogger<DailyReportJob> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var activity = new ActivitySource("MyService").StartActivity("DailyReport");
            var startTime = Stopwatch.GetTimestamp();
            var recordsProcessed = 0;
            var errors = 0;

            try
            {
                recordsProcessed = await GenerateReport(stoppingToken);
            }
            catch (Exception ex)
            {
                errors++;
                _logger.LogError(ex, "Daily report generation failed");
            }
            finally
            {
                _logger.LogInformation(
                    "Daily report job completed. Records: {RecordsProcessed}, Errors: {Errors}, Duration: {DurationMs}ms",
                    recordsProcessed, errors, Stopwatch.GetElapsedTime(startTime).TotalMilliseconds);
            }

            await Task.Delay(TimeSpan.FromHours(24), stoppingToken);
        }
    }
}
```

### Maintain Consistent Schema

Use consistent property names across all services. Follow .NET PascalCase conventions. If one service uses `UserId` and another uses `user_id`, querying becomes painful.

```csharp
// All services use the same schema with PascalCase
var wideEvent = new
{
    RequestId = "req_abc",
    User = new { Id = "user_123" },
    DurationMs = 268.0,
    StatusCode = 200,
};
```

Define your schema once and share it across services via a shared NuGet package or documented standard.

### Never Log Unstructured Strings

Every log must be structured with queryable fields. `Console.WriteLine("User logged in")` is useless for debugging at scale.

```csharp
// Add the data to your wide event or use structured message templates
wideEvent["Order"] = new { Id = orderId, Status = "created" };
wideEvent["Payment"] = new { Error = new { ex.Message } };
// Now it's queryable: WHERE Order.Status = 'created'
```

If you're tempted to write `Console.WriteLine("something happened")`, ask: "What fields would make this queryable?" Then add those fields to your wide event instead.
