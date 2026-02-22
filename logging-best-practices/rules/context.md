---
title: Context, Cardinality, and Dimensionality
impact: CRITICAL
tags: logging, context, cardinality, dimensionality, pii, gdpr, sensitive-data, dotnet, csharp
---

## Context, Cardinality, and Dimensionality

**Impact: CRITICAL**

Wide events must be context-rich with high cardinality and high dimensionality. This enables you to answer questions you haven't anticipated yet - the "unknown unknowns" that traditional logging misses. However, rich context comes with obligations: you must protect sensitive data, control costs, and understand the trade-offs of every field you add.

### Sensitive Data: What Must Never Be Logged

Before adding any field to a wide event, ask: "Would I be comfortable if this value appeared in a support engineer's screen, a log aggregator's search index, or a breach disclosure report?"

**Never log these values:**

- Passwords, password hashes, or password reset tokens
- API keys, client secrets, bearer tokens, JWTs, or refresh tokens
- Credit card numbers (full or partial beyond the last four digits)
- Social Security numbers, national ID numbers, or tax identifiers
- Database connection strings (these contain credentials)
- Encryption keys or signing keys

**Treat these as PII requiring explicit justification:**

- Email addresses
- Full names
- Phone numbers
- IP addresses (considered PII under GDPR)
- Physical addresses
- Dates of birth
- Device identifiers that can be linked to a person

Under GDPR, CCPA, and similar regulations, logging PII creates a data processing obligation. Logs become subject to data subject access requests, right-to-deletion requests, and retention policies. The simplest way to stay compliant is to not log PII in the first place.

**Use stable surrogate identifiers instead of raw PII:**

```csharp
// WRONG - logs raw PII
wideEvent["User"] = new
{
    Id = "user_456",
    Email = "alice@example.com",            // PII
    FullName = "Alice Johnson",              // PII
    LifetimeValueCents = 284700L,            // Could be PII in some jurisdictions
};

// CORRECT - uses surrogate IDs and redacted values
wideEvent["User"] = new
{
    Id = "user_456",                         // Opaque surrogate ID
    EmailHash = HashPii(user.Email),         // One-way hash for correlation
    Subscription = "premium",                // Non-PII business attribute
    AccountAgeDays = 847,                    // Non-PII derived value
};
```

**Masking helper for cases where partial values aid debugging:**

```csharp
public static class PiiMasking
{
    /// <summary>
    /// Masks an email to show only the domain.
    /// "alice@example.com" becomes "***@example.com"
    /// </summary>
    public static string MaskEmail(string email)
    {
        var atIndex = email.IndexOf('@');
        return atIndex > 0 ? $"***{email[atIndex..]}" : "***";
    }

    /// <summary>
    /// Returns a stable, one-way hash suitable for log correlation.
    /// Use when you need to group events by the same user/email
    /// without exposing the raw value.
    /// </summary>
    public static string HashForCorrelation(string value)
    {
        var bytes = System.Security.Cryptography.SHA256.HashData(
            System.Text.Encoding.UTF8.GetBytes(value));
        return Convert.ToHexString(bytes)[..16]; // Truncated for readability
    }

    /// <summary>
    /// Shows only the last four digits of a numeric identifier.
    /// "4111111111111111" becomes "************1111"
    /// </summary>
    public static string MaskToLastFour(string value)
    {
        if (value.Length <= 4) return "****";
        return new string('*', value.Length - 4) + value[^4..];
    }
}

// Usage in wide events
wideEvent["User"] = new
{
    Id = user.Id,
    EmailDomain = PiiMasking.MaskEmail(user.Email),
    EmailHash = PiiMasking.HashForCorrelation(user.Email),
};
```

**Audit your existing wide events.** The examples elsewhere in this skill intentionally log `Email` and `LifetimeValueCents` as business context. In your jurisdiction, these may qualify as personal data. Before adopting those examples, decide whether each field is necessary and whether it requires masking, hashing, or removal. Create an explicit allow-list of fields that may appear in logs, and review it periodically.

### High Cardinality

High cardinality means a field can have millions or billions of unique values. User IDs, request IDs, and transaction IDs are high cardinality fields. Your logging must support querying against any specific value of these fields. Without high cardinality support, you cannot debug issues for specific users.

**Why high cardinality matters:** When a customer reports "my checkout failed," you need to filter by their exact user ID across millions of events. Low-cardinality fields like `StatusCode` (a few hundred values) or `Region` (a handful) help with aggregation but cannot isolate a single user's experience.

**High-cardinality guardrails:**

High-cardinality fields are essential for debugging but have real costs:

- **Index size**: Every unique value in a high-cardinality field consumes index space. A field with 100 million unique values creates a large index entry in backends like Elasticsearch, Loki, or Application Insights.
- **Metrics backends**: If you export log fields to metrics (e.g., Prometheus, Azure Monitor Metrics), high-cardinality labels will explode metric series counts. Most metrics backends have hard limits (e.g., Prometheus recommends fewer than 10 label values per metric; Azure Monitor Metrics has a 50,000 time series limit per metric). Never use user IDs, request IDs, or other unbounded fields as metric labels.
- **Query performance**: Aggregating over a field with billions of unique values is slower than aggregating over a field with a few thousand. Design your queries accordingly.
- **Regulated data**: Raw user IDs may be considered personal data under GDPR if they can be linked back to a natural person. Use opaque surrogate identifiers (e.g., `user_456` rather than an email or SSN) and ensure your identity mapping is access-controlled.

```csharp
// High cardinality fields - essential for debugging, use surrogate IDs
wideEvent["UserId"] = user.Id;           // Opaque surrogate, not email or SSN
wideEvent["RequestId"] = context.TraceIdentifier;
wideEvent["OrderId"] = order.Id;

// Low cardinality fields - good for aggregation and dashboards
wideEvent["Region"] = "westeurope";
wideEvent["Subscription"] = "premium";   // A few tiers, not per-user
wideEvent["StatusCode"] = 200;
wideEvent["Outcome"] = "success";
```

**Rule of thumb:** Log high-cardinality fields in your log/trace backend (Seq, Application Insights, Elasticsearch). Never promote them to metric labels unless you are certain the cardinality is bounded.

### High Dimensionality

High dimensionality means your events have many fields (20-100+). More dimensions mean more questions you can answer without redeploying code.

```csharp
var wideEvent = new Dictionary<string, object?>
{
    // Timing
    ["Timestamp"] = DateTimeOffset.UtcNow,
    ["DurationMs"] = 268.0,

    // Request context
    ["Method"] = "POST",
    ["RouteTemplate"] = "/checkout",
    ["Path"] = "/checkout",
    ["RequestId"] = "req_abc123",

    // Infrastructure
    ["Service"] = "checkout-service",
    ["Version"] = "2.4.1",
    ["Region"] = "westeurope",
    ["CommitHash"] = "690de31f",

    // User context (HIGH CARDINALITY - millions of unique values)
    ["User"] = new
    {
        Id = "user_456",                    // Opaque surrogate ID
        Subscription = "premium",
        AccountAgeDays = 847,
    },

    // Business context
    ["Cart"] = new
    {
        Id = "cart_xyz",
        ItemCount = 3,
        TotalCents = 15999,
        CouponApplied = "SAVE20",
    },

    // Payment details
    ["Payment"] = new
    {
        Method = "card",
        Provider = "stripe",
        LatencyMs = 189.0,
    },

    // Feature flags - crucial for debugging rollouts
    ["FeatureFlags"] = new
    {
        NewCheckoutFlow = true,
    },

    // Outcome
    ["StatusCode"] = 200,
    ["Outcome"] = "success",
};

logger.LogInformation("{@WideEvent}", wideEvent);
```

**Dimensionality guardrails:**

High dimensionality is valuable but has costs that grow with each field you add:

- **Storage**: Every additional property is stored in every event. At 10,000 requests/second, adding a 100-byte property adds ~86 GB/day of raw log data.
- **Indexing**: Many backends index every property by default. More properties means more index entries, more memory, and slower ingestion.
- **Property limits**: Some sinks have hard limits. Application Insights limits custom properties to 200 per event, with 8,192 characters max per property value. Elasticsearch has a default mapping limit of 1,000 fields per index.
- **Query performance**: More indexed fields means more memory pressure on your log backend. Fields that are never queried are pure cost.

**Practical guidelines:**

- **Cap property count**: Aim for 30-60 properties per wide event. If you consistently exceed this, consider whether all properties are genuinely useful for debugging or aggregation.
- **Set max string lengths**: Long strings (stack traces, request bodies, serialized objects) should be truncated. Configure this at the sink level as a safety net.
- **Avoid logging full request/response bodies**: These are large, often contain sensitive data, and are rarely needed. Log specific fields you actually query on.
- **Review field usage**: Periodically check which fields are actually used in queries and dashboards. Remove fields that nobody queries.

**Configure Serilog destructuring policies as safety nets:**

```csharp
Log.Logger = new LoggerConfiguration()
    .Destructure.ToMaximumDepth(3)                // Prevent deep object graph serialization
    .Destructure.ToMaximumStringLength(1024)       // Truncate long strings
    .Destructure.ToMaximumCollectionCount(10)      // Limit array/collection serialization
    .CreateLogger();
```

These limits catch mistakes (e.g., accidentally logging an EF Core entity with navigation properties) but are not a substitute for intentional field selection. Always log explicit projections (`new { order.Id, order.Status }`) rather than passing full objects.

**Per-type destructuring policies for domain objects:**

When you have domain objects that should always be logged in a specific way, use a custom destructuring policy to enforce it:

```csharp
using Serilog.Core;
using Serilog.Events;

public class UserDestructuringPolicy : IDestructuringPolicy
{
    public bool TryDestructure(
        object value,
        ILogEventPropertyValueFactory propertyValueFactory,
        out LogEventPropertyValue? result)
    {
        if (value is User user)
        {
            result = new StructureValue(new[]
            {
                new LogEventProperty("Id", new ScalarValue(user.Id)),
                new LogEventProperty("Subscription", new ScalarValue(user.Subscription)),
                new LogEventProperty("AccountAgeDays", new ScalarValue(user.AccountAgeDays)),
                // Email deliberately excluded - PII
            });
            return true;
        }

        result = null;
        return false;
    }
}

// Register in Serilog configuration
Log.Logger = new LoggerConfiguration()
    .Destructure.With<UserDestructuringPolicy>()
    .Destructure.ToMaximumDepth(3)
    .Destructure.ToMaximumStringLength(1024)
    .Destructure.ToMaximumCollectionCount(10)
    .CreateLogger();
```

This ensures that even if someone logs a full `User` object, only safe, pre-approved fields reach the log sink.

### Always Include Business Context

Include business-specific context, not just technical details. User subscription tier, cart value, feature flags, account age - this context helps prioritise issues and understand business impact.

```csharp
var wideEvent = new Dictionary<string, object?>
{
    ["RequestId"] = "req_123",
    ["Method"] = "POST",
    ["Path"] = "/checkout",
    ["StatusCode"] = 500,

    // Business context that changes response priority
    ["User"] = new
    {
        Id = "user_456",                        // Opaque surrogate ID
        Subscription = "enterprise",            // High-value customer
        AccountAgeDays = 1247,                  // Long-term customer
    },

    ["Cart"] = new
    {
        TotalCents = 249900,                    // $2,499 order
        ContainsAnnualPlan = true,              // Recurring revenue at stake
    },

    ["FeatureFlags"] = new
    {
        NewPaymentFlow = true,                  // Was new code involved?
    },

    ["Error"] = new
    {
        Type = "PaymentException",
        Code = "card_declined",
    },
};
// Now you KNOW this is critical: Enterprise customer, long-term account,
// trying to make a $2,499 purchase, and NewPaymentFlow is enabled
```

Business context transforms debugging from "something broke" to "this enterprise customer can't complete a $2,499 order, and the new payment flow feature flag is enabled." That second sentence changes the priority, the response team, and the resolution timeline.

Note: The original example included `LifetimeValueCents` as business context. This is a powerful signal for prioritisation, but it may qualify as personal data in some jurisdictions because it can be linked to an identifiable individual. If you log monetary values tied to a user, ensure your data processing agreements cover it, or aggregate it into a tier (e.g., `"LtvTier": "high"`) instead.

### Route Template vs. Raw Path

Log the route template as the primary grouping dimension, not the raw path. The raw path (`/users/12345`) is high-cardinality and creates millions of unique values that are useless for aggregation. The route template (`/users/{id}`) groups all requests to the same endpoint.

```csharp
// WRONG - raw path creates unbounded cardinality for grouping
// Every distinct user ID becomes a separate group
wideEvent["Path"] = "/users/12345";
// Querying: GROUP BY Path gives millions of unique entries

// CORRECT - route template for grouping, raw path for specific debugging
wideEvent["RouteTemplate"] = "/users/{id}";     // Bounded cardinality, good for GROUP BY
wideEvent["Path"] = "/users/12345";              // High cardinality, good for exact lookup
```

**Getting the route template in ASP.NET Core:**

```csharp
// In middleware or a filter, extract the route template from the endpoint metadata
public static string? GetRouteTemplate(HttpContext context)
{
    var endpoint = context.GetEndpoint();

    // Minimal APIs and controllers expose route pattern via endpoint metadata
    var routePattern = (endpoint as RouteEndpoint)?.RoutePattern.RawText;
    if (routePattern is not null)
        return routePattern;

    // Fallback: check for the RouteData
    var routeData = context.GetRouteData();
    if (routeData.Values.Count > 0)
    {
        var controller = routeData.Values["controller"];
        var action = routeData.Values["action"];
        if (controller is not null && action is not null)
            return $"{controller}/{action}";
    }

    return context.Request.Path.Value;
}
```

**Using it in middleware:**

```csharp
// In your wide event middleware (after the endpoint has been resolved)
public async Task InvokeAsync(HttpContext context)
{
    // ... timing setup ...

    try
    {
        await _next(context);
    }
    finally
    {
        var wideEvent = context.GetWideEvent();
        wideEvent["RouteTemplate"] = GetRouteTemplate(context);
        wideEvent["Path"] = context.Request.Path.Value;
        // RouteTemplate is only available after endpoint routing has run,
        // so this must be set in the finally block or after _next(context).
    }
}
```

**Important:** The route template is only resolved after endpoint routing runs (`UseRouting()`). If your wide event middleware runs before routing, the endpoint will be null. Place your middleware after `UseRouting()` or set the `RouteTemplate` field in the `finally` block after `_next(context)` returns.

**When using Serilog's `UseSerilogRequestLogging`:**

Serilog already logs `RequestPath` (the raw path). To add the route template, enrich the diagnostic context:

```csharp
app.UseSerilogRequestLogging(options =>
{
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        var endpoint = httpContext.GetEndpoint();
        var routeTemplate = (endpoint as RouteEndpoint)?.RoutePattern.RawText;
        if (routeTemplate is not null)
        {
            diagnosticContext.Set("RouteTemplate", routeTemplate);
        }
    };
});
```

### Always Include Environment Characteristics

Include environment and deployment information in every wide event. This context is essential for correlating issues with deployments, identifying region-specific problems, and understanding the runtime environment.

**Environment fields to include:**

```csharp
// Capture once at startup and reuse via DI or a static class
public static class EnvironmentContext
{
    public static readonly object Info = new
    {
        // Deployment info
        CommitHash = Environment.GetEnvironmentVariable("COMMIT_SHA")
            ?? Environment.GetEnvironmentVariable("GIT_COMMIT"),
        Version = Environment.GetEnvironmentVariable("SERVICE_VERSION")
            ?? typeof(Program).Assembly.GetName().Version?.ToString(),
        DeploymentId = Environment.GetEnvironmentVariable("DEPLOYMENT_ID"),
        DeployTime = Environment.GetEnvironmentVariable("DEPLOY_TIMESTAMP"),

        // Infrastructure
        Service = Environment.GetEnvironmentVariable("SERVICE_NAME"),
        Region = Environment.GetEnvironmentVariable("AZURE_REGION")
            ?? Environment.GetEnvironmentVariable("AWS_REGION"),
        AvailabilityZone = Environment.GetEnvironmentVariable("AVAILABILITY_ZONE"),
        InstanceId = Environment.GetEnvironmentVariable("WEBSITE_INSTANCE_ID")
            ?? Environment.MachineName,

        // Runtime
        DotNetVersion = Environment.Version.ToString(),
        Runtime = System.Runtime.InteropServices.RuntimeInformation.FrameworkDescription,
        ProcessId = Environment.ProcessId,

        // Environment type
        EnvironmentName = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")
            ?? Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT"),
    };
}

// Usage in wide event
wideEvent["Env"] = EnvironmentContext.Info;
```

The `Environment.GetEnvironmentVariable` approach above works and is common in containerised deployments where configuration is injected via environment variables. However, .NET has a richer configuration system that supports `appsettings.json`, environment-specific overrides (`appsettings.Production.json`), user secrets, Azure Key Vault, and hot-reload.

**Alternative: Using IConfiguration (idiomatic .NET):**

```csharp
// appsettings.json
// {
//   "ServiceInfo": {
//     "Name": "checkout-service",
//     "Version": "2.4.1"
//   }
// }
//
// These values can be overridden per environment via appsettings.Production.json,
// environment variables (ServiceInfo__Name), or any other configuration provider.

public sealed class EnvironmentContextProvider
{
    public object Info { get; }

    public EnvironmentContextProvider(IConfiguration configuration, IWebHostEnvironment env)
    {
        Info = new
        {
            // From configuration (supports appsettings.json, env vars, Key Vault, etc.)
            Service = configuration["ServiceInfo:Name"],
            Version = configuration["ServiceInfo:Version"],
            CommitHash = configuration["ServiceInfo:CommitHash"],
            DeploymentId = configuration["ServiceInfo:DeploymentId"],

            // From IWebHostEnvironment (built-in)
            EnvironmentName = env.EnvironmentName,

            // From the runtime (always available)
            Region = configuration["Azure:Region"]
                ?? Environment.GetEnvironmentVariable("AWS_REGION"),
            InstanceId = Environment.GetEnvironmentVariable("WEBSITE_INSTANCE_ID")
                ?? Environment.MachineName,
            DotNetVersion = Environment.Version.ToString(),
            Runtime = System.Runtime.InteropServices.RuntimeInformation.FrameworkDescription,
            ProcessId = Environment.ProcessId,
        };
    }
}

// Register as a singleton in DI
builder.Services.AddSingleton<EnvironmentContextProvider>();

// Usage in middleware or handlers
public class WideEventMiddleware
{
    private readonly EnvironmentContextProvider _envContext;

    public WideEventMiddleware(RequestDelegate next, EnvironmentContextProvider envContext)
    {
        _envContext = envContext;
        // ...
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var wideEvent = context.GetWideEvent();
        wideEvent["Env"] = _envContext.Info;
        // ...
    }
}
```

The `IConfiguration` approach is preferred when your service already uses `appsettings.json` for other configuration, because it keeps all configuration in one system with consistent override semantics. The raw `Environment.GetEnvironmentVariable` approach is simpler for container-native services that receive all configuration via environment variables. Both are valid; choose based on your deployment model.

**Why environment context matters:**

- **CommitHash**: Instantly identify which code version caused an issue
- **DeploymentId**: Correlate errors with specific deployments
- **Region/AvailabilityZone**: Identify region-specific failures
- **InstanceId**: Debug issues affecting specific instances
- **Version**: Track issues across service versions
- **EnvironmentName**: Distinguish production from staging issues

This environment context should be captured once at service startup and automatically included in every wide event via middleware or Serilog enrichment.

### Summary: Context Field Checklist

When designing your wide event schema, use this checklist:

| Category | Example Fields | Cardinality | Notes |
|----------|---------------|-------------|-------|
| **Request** | Method, RouteTemplate, StatusCode | Low | Good for aggregation and dashboards |
| **Request** | Path, RequestId, TraceId | High | Good for exact lookup, not for metric labels |
| **User** | UserId (surrogate) | High | Use opaque IDs, not PII |
| **User** | Subscription, AccountAgeDays | Low | Safe business context |
| **Business** | CartTotal, ItemCount, CouponApplied | Low-Medium | Drives prioritisation |
| **Feature Flags** | Flag names and values | Low | Essential for rollout debugging |
| **Environment** | Service, Version, CommitHash, Region | Low | Captured once at startup |
| **Timing** | DurationMs, Timestamp | Continuous | Always include |
| **Error** | Error.Type, Error.Code | Low-Medium | For error categorisation |
| **Sensitive** | Email, FullName, IP, CreditCard | High | Mask, hash, or omit. See PII section |
