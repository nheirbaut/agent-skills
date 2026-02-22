---
name: logging-best-practices
description: Logging best practices for .NET applications using structured wide events, ILogger<T>, Serilog, and OpenTelemetry correlation
license: MIT
metadata:
  author: nheirbaut
  version: "2.0.0"
---

# Logging Best Practices Skill (.NET)

Version: 2.0.0

## Purpose
This skill provides guidelines for effective logging in .NET applications. The recommended pattern for request-level logging is **wide events** (also called canonical log lines): emit one context-rich summary event per request per service, rather than scattering many small log calls throughout the handler.

For HTTP workloads, the **recommended implementation** is `UseSerilogRequestLogging()` + `IDiagnosticContext` — it produces a wide event with minimal handler code. If your application uses OpenTelemetry, enriching the active `Activity` span via `Activity.SetTag()` is a modern alternative that feeds both traces and logs. A manual `Dictionary<string, object?>` approach gives full control when needed.

Wide events are the **preferred default**, not a strict rule. Long-running operations, streaming responses, background jobs with multiple phases, and retry loops may legitimately need additional step-level events for visibility into progress. Prefer one summary event; add step events only when a single end-of-request event would leave you blind.
The stack covered by these guidelines: `ILogger<T>` via `Microsoft.Extensions.Logging`, **Serilog** as the logging provider, ASP.NET Core middleware, and **OpenTelemetry / W3C Trace Context** for distributed correlation.

## When to Apply

Apply these guidelines when:
- Writing or reviewing logging code in a .NET project
- Adding `_logger.LogInformation(...)`, `Console.WriteLine(...)`, or similar
- Designing logging strategy for new .NET services
- Setting up Serilog, Application Insights, or other .NET logging infrastructure
- Reviewing correlation, context enrichment, or log level choices

## Core Principles

### 1. Wide Events (CRITICAL)
Emit **one context-rich event per request per service**. Choose the implementation tier that fits your stack:

| Approach | Boilerplate | Provider | Best For |
|----------|-------------|----------|----------|
| `UseSerilogRequestLogging` + `IDiagnosticContext` | ~4 lines/handler | Serilog only | Most HTTP workloads (recommended) |
| `Activity.SetTag()` (OpenTelemetry spans) | ~4 lines/handler | Any OTel-compatible | Apps already using OTel tracing |
| Manual `Dictionary<string, object?>` | ~40 lines/handler | Any `ILogger` | Full control over event shape |

#### Recommended: `UseSerilogRequestLogging` + `IDiagnosticContext`

Serilog's built-in request logging middleware automatically captures timing, status code, method, and path. Inject `IDiagnosticContext` into handlers to attach business context to the same summary event — no try/catch/finally boilerplate needed.

```csharp
// Program.cs — configure once
app.UseSerilogRequestLogging(options =>
{
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestId", httpContext.TraceIdentifier);
    };
});

// Handler — just enrich the existing event
app.MapPost("/checkout", async (HttpContext context, IDiagnosticContext diagnosticContext) =>
{
    var user = await GetUser(context.User);
    diagnosticContext.Set("User", new { user.Id, user.Subscription, user.Trial });

    var cart = await GetCart(user.Id);
    diagnosticContext.Set("Cart", new { TotalCents = cart.Total, ItemCount = cart.Items.Count });

    return Results.Ok(new { Success = true });
});
```

Serilog emits a single wide event at request completion with all enriched properties, timing, status code, method, and path included automatically.

#### Alternative: OpenTelemetry Span Enrichment

If your application uses OpenTelemetry, enrich the active span instead. Tags on the span flow to both traces and structured logs (via the OTel log bridge), giving you wide-event semantics without a Serilog dependency.

```csharp
app.MapPost("/checkout", async (HttpContext context) =>
{
    var user = await GetUser(context.User);
    Activity.Current?.SetTag("user.id", user.Id);
    Activity.Current?.SetTag("user.subscription", user.Subscription);

    var cart = await GetCart(user.Id);
    Activity.Current?.SetTag("cart.total_cents", cart.Total);
    Activity.Current?.SetTag("cart.item_count", cart.Items.Count);

    return Results.Ok(new { Success = true });
});
```

#### Full Control: Manual Dictionary

When you need full control over the event shape, work with any `ILogger` provider, or need custom emission logic (e.g., different log levels per outcome), build the event manually. Consider using a typed `RequestWideEvent` class (see `rules/wide-events.md`) instead of a raw dictionary for compile-time safety.

```csharp
app.MapPost("/checkout", async (HttpContext context, ILogger<Program> logger) =>
{
    var startTime = Stopwatch.GetTimestamp();
    var wideEvent = new Dictionary<string, object?>
    {
        ["Method"] = "POST",
        ["Path"] = "/checkout",
        ["Service"] = "checkout",
        // W3C Trace Context — populated automatically by ASP.NET Core
        ["TraceId"] = Activity.Current?.TraceId.ToString() ?? context.TraceIdentifier,
    };

    try
    {
        var user = await GetUser(context.User);
        wideEvent["User"] = new { user.Id, user.Subscription, user.Trial };
        var cart = await GetCart(user.Id);
        wideEvent["Cart"] = new { TotalCents = cart.Total, ItemCount = cart.Items.Count };
        wideEvent["Outcome"] = "success";
        return Results.Ok(new { Success = true });
    }
    catch (Exception ex)
    {
        wideEvent["StatusCode"] = 500;
        wideEvent["Outcome"] = "error";
        wideEvent["Error"] = new { Type = ex.GetType().Name, ex.Message };
        // The structured Error property is for querying; the exception parameter
        // is what carries the stack trace to the sink.
        logger.LogError(ex, "Request failed {@WideEvent}", wideEvent);
        throw;
    }
    finally
    {
        wideEvent["DurationMs"] = Stopwatch.GetElapsedTime(startTime).TotalMilliseconds;
        wideEvent["Timestamp"] = DateTimeOffset.UtcNow;
        // Error cases are already logged with LogError (and the exception) in catch.
        var outcome = wideEvent.GetValueOrDefault("Outcome")?.ToString();
        if (outcome != "error")
        {
            logger.LogInformation("{@WideEvent}", wideEvent);
        }
    }
});
```
Note: `{@WideEvent}` uses Serilog's destructuring operator. With other providers, use `BeginScope` or `LogContext.PushProperty` instead. See `rules/wide-events.md` and `rules/structure.md` for details.

### 2. High Cardinality & Dimensionality (CRITICAL)

Include fields with **high cardinality** (user IDs, request IDs — millions of unique values) for per-user debugging, and **high dimensionality** (20–100+ fields per event) to answer questions you haven't anticipated yet.

Guardrails to keep in mind:
- Never use high-cardinality fields (user IDs, request IDs) as metric labels — it explodes metric series counts.
- Use the route template (`/users/{id}`) as your grouping dimension, not the raw path (`/users/12345`).
- Cap property count around 30–60 per event. Configure Serilog's `ToMaximumDepth`, `ToMaximumStringLength`, and `ToMaximumCollectionCount` as safety nets.
- User IDs may be PII under GDPR — use opaque surrogate identifiers, not emails or national IDs.

### 3. Business Context (CRITICAL)

Always include business-specific context: user subscription tier, cart value, feature flags, account age. The goal is to know "an enterprise customer couldn't complete a $2,499 purchase with the new payment flow enabled" — not just "checkout failed."

Note: fields like `LifetimeValueCents` can qualify as personal data in some jurisdictions. Aggregate into tiers (`"LtvTier": "high"`) if there is any GDPR/CCPA ambiguity.

### 4. Environment Characteristics (CRITICAL)

Include deployment and runtime info in every event: commit hash, service version, region, instance ID, environment name. Capture it once at startup and attach via middleware or Serilog enrichers.

Use `Environment.GetEnvironmentVariable(...)` for container-native deployments, or `IConfiguration` + `IWebHostEnvironment` for the idiomatic .NET approach with `appsettings.json` overrides. Both are valid; choose based on your deployment model.

### 5. Proper Log Levels (HIGH)

Use all six .NET log levels to reflect actual operational significance:

| Level | When to Use |
|-------|-------------|
| **Trace** | Extremely detailed diagnostics; opt-in only, disabled in production |
| **Debug** | Development diagnostics; disabled by default in production |
| **Information** | Normal operation — request summaries, successful events, startup/shutdown |
| **Warning** | Degraded but functional — retry succeeded, fallback used, rate limit approaching |
| **Error** | A specific operation failed — request not fulfilled, dependency timed out |
| **Critical** | System integrity threatened — connection pool exhausted, unrecoverable state |

Do not simplify to "only Information and Error." Warning is the right level for a retry that succeeded. Critical is the right level for events that warrant separate alerting thresholds.

### 6. Single Logger via DI (HIGH)

Use `ILogger<T>` resolved through dependency injection everywhere. Configure Serilog once in `Program.cs` with enrichers for environment context. Never instantiate loggers manually and never use `Console.WriteLine` for operational logging (the only acceptable exception is startup before the logging pipeline is ready).

### 7. Request Logging (HIGH)
Choose the approach that matches your stack and control requirements:

| Approach | When to Use |
|----------|-------------|
| `UseSerilogRequestLogging` + `IDiagnosticContext` | Default for HTTP workloads with Serilog. Automatic timing, status, method, path. Minimal handler code. |
| `Activity.SetTag()` (OpenTelemetry) | Application already instrumented with OTel. Tags flow to traces and logs. |
| Custom `WideEventMiddleware` with dictionary | Need full control over event shape, custom log levels per status code, or provider independence. |
| Custom `WideEventMiddleware` with typed class | Same as above, plus compile-time schema safety for well-known event shapes. |

### 8. High-Performance Logging (HIGH)

For hot paths (request middleware, tight loops, high-frequency background work), use `[LoggerMessage]` source generators. They eliminate boxing of value types, parse message templates at compile time rather than per-call, and guard on `IsEnabled` before doing any work.

```csharp
public partial class CheckoutService
{
    [LoggerMessage(EventId = 1000, Level = LogLevel.Information,
        Message = "Order {OrderId} created for user {UserId}")]
    private partial void LogOrderCreated(string orderId, string userId);

    [LoggerMessage(EventId = 1002, Level = LogLevel.Error,
        Message = "Payment failed for order {OrderId}")]
    private partial void LogPaymentFailed(string orderId);
}
```

Define stable `EventId` values per service or feature area so dashboards and alerts survive message template changes.

### 9. W3C Trace Context Correlation (HIGH)

Use `Activity.Current?.TraceId.ToString()` as the correlation key in wide events. ASP.NET Core automatically creates an `Activity` per request from incoming `traceparent` headers. `HttpClient` propagates `traceparent` to downstream calls automatically. You do not need custom `X-Correlation-Id` headers or manual propagation.

Use `TraceId` (not `Activity.Current?.Id`) — `TraceId` is always a 32-character hex string that stays the same across the full distributed trace.

Business correlation IDs (e.g., `OrderId` that spans multiple traces over days) are a separate concept — add them as explicit wide event fields alongside `TraceId`.

### 10. Structure & Consistency (HIGH)

- Use structured message templates with named placeholders — never string interpolation (`$"..."`)
- `{@Property}` (destructuring) is **Serilog-specific**; use `BeginScope` or `LogContext.PushProperty` for provider-agnostic scoped properties
- PascalCase property names per .NET convention, consistent across all services
- Log explicit projections (`new { order.Id, order.Status }`), not full entity objects
- Define schema once per organisation; share via a shared package or documented standard

## Anti-Patterns to Avoid

1. **Scattered logs**: Multiple disconnected `_logger.Log*()` calls per request where a single wide event would carry all the context
2. **Manual logger instantiation**: Creating `new LoggerConfiguration()` in each class instead of injecting `ILogger<T>`
3. **Missing environment context**: No commit hash, version, or region in events
4. **Missing business context**: Technical-only events with no user or business data
5. **Unstructured strings**: `Console.WriteLine("something happened")` instead of structured events
6. **Inconsistent schemas**: `UserId` in one service, `user_id` in another
7. **String interpolation in log calls**: `$"Order {id} created"` instead of `"Order {OrderId} created", id`
8. **Swallowing exception stack traces**: Capturing only `ex.Message` without passing `ex` to `LogError`
9. **Logging PII without masking**: Raw email addresses, full names, IP addresses, payment data
10. **Unbounded object destructuring**: Logging full EF Core entities with navigation properties (circular refs, lazy loading, sensitive data leakage)
11. **Manual correlation header propagation**: Custom `X-Correlation-Id` middleware when W3C Trace Context handles it automatically

## Guidelines

### Wide Events (`rules/wide-events.md`)

One summary event per service hop. **Recommended approach**: `UseSerilogRequestLogging()` + `IDiagnosticContext` for HTTP workloads — automatic timing, status, method, path with minimal handler code. **OpenTelemetry alternative**: enrich the active `Activity` span via `SetTag()` when OTel is already in use. **Full control**: manual dictionary or typed `RequestWideEvent` class with try/catch/finally for custom emission logic. Exception handling: always pass the exception object as the first argument to `LogError` to preserve the stack trace — the structured `Error` property on the event is for querying, not a substitute. Correlation via `Activity.Current?.TraceId` (W3C Trace Context, automatic) — not manual header propagation. Business correlation IDs (e.g., `OrderId`) are a separate concept that spans multiple traces. Step-level events are legitimate for long-running operations, streaming, and retry loops — they supplement, not replace, the summary. Configure Serilog destructuring depth/length/collection limits as safety nets.

### Context (`rules/context.md`)

**PII/GDPR**: never log passwords, tokens, full card numbers, connection strings, or encryption keys. Treat email addresses, full names, phone numbers, IP addresses, and dates of birth as PII requiring explicit justification. Use opaque surrogate IDs; hash PII with SHA-256 for correlation; use `PiiMasking` helpers for partial masking. High cardinality with guardrails: index cost, metrics backend explosion, query performance, and regulated data concerns. High dimensionality with guardrails: storage cost, indexing limits (Application Insights caps at 200 properties, Elasticsearch at 1,000 fields), and query performance. Log route template (`/users/{id}`) for grouping, raw path for exact lookup — route template is only available after endpoint routing runs. Environment context via `Environment.GetEnvironmentVariable` (container-native) or `IConfiguration` + `IWebHostEnvironment` (idiomatic .NET). Per-type Serilog destructuring policies enforce safe field selection for domain objects.

### Structure (`rules/structure.md`)

`ILogger<T>` via DI everywhere; configure Serilog once in `Program.cs`. Use all six log levels with operational precision — `Warning` for degraded-but-functional, `Error` for failed operations, `Critical` for system-integrity threats. Prefer `UseSerilogRequestLogging()` + `IDiagnosticContext` for HTTP workloads; use custom `WideEventMiddleware` when you need full control or provider independence (with correct log levels per status code). `{@}` destructuring is Serilog-specific; `BeginScope` and `LogContext.PushProperty` are provider-agnostic alternatives for scoped properties. `[LoggerMessage]` source generators for hot paths: zero boxing, compile-time template parsing, automatic level-check guard. Define stable `EventId` values per feature area. Non-HTTP workloads (Worker Services, message handlers, background jobs): use `Activity` for correlation, `BeginScope` for scope instead of `HttpContext.Items`, and emit a summary event per unit of work. Consistent PascalCase schema; never log unstructured strings.

### Common Pitfalls (`rules/pitfalls.md`)

Unnecessary scattered logs (with nuance: step logs are legitimate for long-running, streaming, retry, and multi-phase workflows). Designing only for known unknowns — wide events with rich context let you investigate bugs you didn't anticipate. Exception stack trace loss: always pass `ex` to `LogError`, not just `ex.Message`. Sensitive data logging (PII, secrets, payment data, raw request bodies) — mask, hash, or omit. Unbounded destructuring of EF Core entities: circular refs, lazy loading side effects, sensitive data leakage, log storage explosion. String interpolation in log calls destroys structured fields. `Console.WriteLine`/`Debug.WriteLine` bypass the entire logging pipeline. Manual `X-Correlation-Id` propagation is unnecessary with W3C Trace Context; use `Activity.Current?.TraceId`.

## References

- [Logging Sucks](https://loggingsucks.com)
- [Observability Wide Events 101](https://boristane.com/blog/observability-wide-events-101/)
- [Stripe - Canonical Log Lines](https://stripe.com/blog/canonical-log-lines)
- [Serilog Best Practices](https://benfoster.io/blog/serilog-best-practices/)
- [ASP.NET Core Logging](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/logging/)
- [High-Performance Logging in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/high-performance-logging)
- [Distributed Tracing in .NET](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/distributed-tracing)
- [OpenTelemetry for .NET](https://opentelemetry.io/docs/languages/dotnet/)

*Adapted from [boristane/agent-skills](https://github.com/boristane/agent-skills) (MIT). Original concepts by Boris Tane.*
