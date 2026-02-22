# Logging Best Practices (.NET)

An OpenCode skill for structured logging in .NET applications.

## What it covers

- **Wide events** (canonical log lines) — one context-rich summary event per request per service
- **`ILogger<T>`** with Serilog as the logging provider
- **OpenTelemetry / W3C Trace Context** for distributed correlation via `Activity`
- **`[LoggerMessage]` source generators** for high-performance, zero-allocation logging
- **`UseSerilogRequestLogging()`** and `IDiagnosticContext` for HTTP workloads
- **PII/GDPR-aware** context enrichment
- **Non-HTTP workloads** — Worker Services, message handlers, background jobs

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Skill definition and summary |
| `metadata.json` | Skill metadata |
| `rules/structure.md` | Logger setup, Serilog configuration, log levels, `LoggerMessage` generators, `EventId` conventions |
| `rules/wide-events.md` | Wide event pattern, middleware implementations, request/response logging |
| `rules/context.md` | Correlation via `Activity`, scoped properties, `BeginScope`, `LogContext`, PII handling |
| `rules/pitfalls.md` | Common mistakes — stack trace loss, string interpolation, unbounded destructuring, `Console.WriteLine` |

## Attribution

Adapted from [boristane/agent-skills](https://github.com/boristane/agent-skills) (MIT). Original concepts by Boris Tane.