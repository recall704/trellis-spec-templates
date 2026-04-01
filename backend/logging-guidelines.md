# Logging Guidelines

> How logging is done in this project.

---

## Overview

The backend uses **zap** for structured logging with support for both JSON (production) and console (development) output. The logger bridges `log/slog` and the standard `log` package so all logging flows through a single structured pipeline. Logs are designed for machine parsing and human readability depending on environment.

---

## Log Levels

| Level | Use Case |
|-------|----------|
| `Debug` | Detailed diagnostic info (query params, feature flags, internal state) |
| `Info` | Normal operational events (server start, request lifecycle, state transitions) |
| `Warn` | Unexpected but handled situations (retry, fallback, degraded mode) |
| `Error` | Errors that need attention but don't crash the service |
| `Fatal` | Unrecoverable errors (port bind failed, config invalid) |

---

## Logger Access

### Global Logger

```go
// *zap.Logger for performance-critical paths
logger.L().Info("server started", zap.String("addr", addr))

// *zap.SugaredLogger for convenience
logger.S().Infow("request completed", "method", "GET", "path", "/api/v1/entities")

// With pre-attached fields
logger.With(zap.Int64("entity_id", id)).Info("processing")
```

### Context-Based Logger

Use `logger.FromContext` to retrieve a logger tied to the request context:

```go
ctx = logger.IntoContext(ctx, baseLogger.With(zap.String("request_id", reqID)))
l := logger.FromContext(ctx)
l.Info("processing request")
```

This maintains request correlation across handler → service → repository layers.

### Named Logger

Create scoped loggers for specific components:

```go
componentLogger := logger.L().Named("repository.account")
componentLogger.Info("account created", zap.Int64("id", id))
```

---

## Structured Logging

### Basic Usage

Always use structured fields instead of string interpolation:

```go
// Good: structured, queryable
logger.Info("entity created",
    zap.Int64("entity_id", entity.ID),
    zap.String("platform", entity.Platform),
)

// Bad: loses structure
logger.Info(fmt.Sprintf("entity %d created", entity.ID))
```

### Error Logging

Always include the error and relevant context:

```go
// Good: includes context and error
logger.Error("failed to update entity",
    zap.Int64("entity_id", id),
    zap.String("field", "credentials"),
    zap.Error(err),
)

// Bad: generic error
logger.Error("failed", zap.Error(err))
```

### Legacy Printf Compatibility

For gradual migration or third-party code, use `LegacyPrintf`:

```go
logger.LegacyPrintf("component.name", "operation failed: id=%d err=%v", id, err)
```

This routes printf-style calls through the structured logger with a `legacy_printf` tag and infers the log level from message content.

---

## Log Configuration

### InitOptions

```go
type InitOptions struct {
    Level           string          // "debug", "info", "warn", "error"
    Format          string          // "json", "console"
    ServiceName     string          // Service identifier
    Environment     string          // "production", "development", "bootstrap"
    Caller          bool            // Include caller info
    StacktraceLevel string          // "none", "error", "fatal"
    Output          OutputOptions   // stdout and/or file output
    Rotation        RotationOptions // Log file rotation
    Sampling        SamplingOptions // Log sampling for high-volume paths
}
```

### Output Options

Logs can be written to multiple destinations simultaneously:

```yaml
logging:
  level: info
  format: json
  output: stdout        # stdout, stderr, or file path
  stacktrace_level: error
  rotation:
    max_size_mb: 100
    max_backups: 10
    max_age_days: 7
    compress: true
  sampling:
    enabled: true
    initial: 100
    thereafter: 100
```

### Reconfiguration

The logger can be reconfigured at runtime:

```go
logger.Reconfigure(func(opts *logger.InitOptions) error {
    opts.Level = "debug"
    return nil
})

// Or change level directly
logger.SetLevel("debug")
```

---

## Sink Interface

For advanced observability pipelines, implement the `Sink` interface to receive log events:

```go
type Sink interface {
    WriteLogEvent(event *LogEvent)
}

type LogEvent struct {
    Time       time.Time
    Level      string
    Component  string
    Message    string
    LoggerName string
    Fields     map[string]any
}

logger.SetSink(mySink)
```

The sink receives events regardless of log level, enabling scenarios like indexing all ops logs while only emitting `info`+ to stdout.

---

## Standard Library Bridging

The logger automatically bridges:

- **`log/slog`** — all `slog` calls route through zap via a custom handler
- **`log` (standard library)** — `log.Print`, `log.Printf` etc. are captured and routed through zap with level inference from message content

This means third-party libraries using standard logging are automatically captured in the structured pipeline.

---

## What to Log

### Request Lifecycle

Log at the start and end of significant operations:

```go
logger.Info("creating entity",
    zap.String("name", req.Name),
    zap.String("platform", req.Platform),
)
logger.Info("entity created",
    zap.Int64("entity_id", entity.ID),
)
```

### State Transitions

Log important state changes:

```go
logger.Info("entity rate limited",
    zap.Int64("entity_id", id),
    zap.Time("reset_at", resetAt),
)
```

### Performance Metrics

Log timing and throughput data:

```go
logger.Info("operation completed",
    zap.String("operation", "batch_update"),
    zap.Int("count", len(items)),
    zap.Duration("duration", elapsed),
)
```

---

## What NOT to Log

### Sensitive Data

Never log:
- API keys, tokens, passwords
- Personal identifiable information (PII)
- Request/response bodies containing credentials
- Authorization headers

```go
// Bad: logs sensitive credential
logger.Info("credentials", zap.Any("creds", credentials))

// Good: log presence without content
logger.Info("credentials updated", zap.Int64("entity_id", id))
```

### Large Data

Avoid logging:
- Full request/response bodies
- Large JSON payloads
- Stack traces in non-error paths

### Redaction

Use a redaction utility for text that may contain sensitive data:

```go
logger.Error("request failed",
    zap.String("body_preview", logredact.RedactText(body)),
)
```

---

## Log Format

### Production (JSON)

```json
{
  "level": "INFO",
  "time": "2026-03-29T10:15:30.000Z",
  "caller": "handler/entity_handler.go:45",
  "msg": "entity created",
  "entity_id": 123,
  "platform": "example",
  "service": "myservice",
  "env": "production"
}
```

### Development (Console)

```
2026-03-29 10:15:30 INF handler/entity_handler.go:45 > entity created entity_id=123 platform=example
```

---

## Testing with Logs

Use zap's observer for capturing and asserting on log output:

```go
// Capture logs in tests
observedLogs := zapcore.NewCore(
    zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig()),
    zapcore.NewMapObjectEncoder(),
    zap.DebugLevel,
)
observed := zaptest.NewObserver(observedLogs)
testLogger := zap.New(observed)

// Assert on log entries
entries := observed.All()
require.Len(t, entries, 1)
require.Equal(t, "entity created", entries[0].Message)
```

Or use `zap.NewExample()` for simple cases:

```go
logger, buf := zap.NewDevelopment()
logger.Info("test message", zap.String("key", "value"))
// buf contains the formatted output
```

---

## Common Mistakes

### 1. Not Including Error

```go
// Bad
logger.Info("failed to process")

// Good
logger.Info("failed to process", zap.Error(err))
```

### 2. Using String Interpolation

```go
// Bad - loses structure
logger.Info("entity " + entityID + " not found")

// Good - maintains structure
logger.Info("entity not found", zap.Int64("entity_id", entityID))
```

### 3. Logging Without Appropriate Level

```go
// Bad - error should be Error level
logger.Info("connection failed", zap.Error(err))

// Good
logger.Error("connection failed", zap.Error(err))
```

### 4. Not Using Logger from Context

```go
// Bad - loses request correlation
globalLogger.Info("request received")

// Good - maintains trace context
l := logger.FromContext(ctx)
l.Info("request received")
```

### 5. Double-Logging Errors

```go
// Bad - log in repository AND handler
logger.Error("db query failed", zap.Error(err))
return nil, err  // handler also logs

// Good - let the handler decide logging level
return nil, err
```

### 6. Logging Sensitive Data

```go
// Bad - exposes credentials
logger.Info("auth request", zap.Any("headers", r.Header))

// Good - log only safe metadata
logger.Info("auth request",
    zap.String("method", r.Method),
    zap.String("path", r.URL.Path),
)
```
