# Error Handling

> How errors are handled in this project.

---

## Overview

The backend uses a structured error system based on `ApplicationError` which wraps HTTP status codes with machine-readable error codes and user-facing messages. Errors are designed to be consumed by both the API layer and internal logging/monitoring systems.

---

## Error Types

### ApplicationError

The primary error type in `internal/pkg/errors/`:

```go
type ApplicationError struct {
    Status
    cause error
}

type Status struct {
    Code     int32             `json:"code"`      // HTTP status code
    Reason   string            `json:"reason,omitempty"`  // Machine-readable code
    Message  string            `json:"message"`          // User-facing message
    Metadata map[string]string `json:"metadata,omitempty"` // Extra details
}
```

`ApplicationError` implements `error`, supports `Unwrap()` for Go 1.13 error chains, and `Is()` for matching by code and reason.

### Error Creator Helpers

Convenience functions for common HTTP error codes:

```go
errors.BadRequest("VALIDATION_ERROR", "invalid input")       // 400
errors.Unauthorized("TOKEN_EXPIRED", "token has expired")     // 401
errors.Forbidden("INSUFFICIENT_PERMISSIONS", "access denied") // 403
errors.NotFound("RESOURCE_NOT_FOUND", "resource not found")   // 404
errors.Conflict("DUPLICATE_ENTRY", "resource already exists") // 409
errors.TooManyRequests("RATE_LIMITED", "too many requests")   // 429
errors.InternalServer("INTERNAL_ERROR", "unexpected error")   // 500
errors.ServiceUnavailable("SERVICE_DOWN", "service unavailable") // 503
errors.GatewayTimeout("UPSTREAM_TIMEOUT", "upstream timed out")  // 504
```

### Sentinel Errors

Define sentinel errors in the service package for domain-specific conditions:

```go
var (
    ErrEntityNotFound = errors.NotFound("ENTITY_NOT_FOUND", "entity not found")
    ErrInvalidInput   = errors.BadRequest("INVALID_INPUT", "invalid input")
)
```

### Error Inspection

Helper functions to inspect errors (support wrapped errors):

```go
errors.Code(err)        // returns HTTP status code
errors.Reason(err)      // returns machine-readable reason
errors.Message(err)     // returns user-facing message
errors.FromError(err)   // converts any error to *ApplicationError
errors.ToHTTP(err)      // returns (statusCode, Status body)
```

---

## Error Handling Patterns

### In Services

Services return errors with context using `fmt.Errorf`:

```go
func (s *EntityService) GetByID(ctx context.Context, id int64) (*Entity, error) {
    entity, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get entity: %w", err)  // wrap, don't munge
    }
    return entity, nil
}
```

Use `errors.Is()` to check for sentinel errors:

```go
if errors.Is(err, service.ErrEntityNotFound) {
    return nil, err
}
```

### In Repository

Repository translates persistence errors to service errors:

```go
func (r *entityRepository) GetByID(ctx context.Context, id int64) (*service.Entity, error) {
    m, err := r.client.Entity.Query().Where(dbentity.IDEQ(id)).Only(ctx)
    if err != nil {
        return nil, translatePersistenceError(err, service.ErrEntityNotFound, nil)
    }
    // ...
}
```

The `translatePersistenceError` helper maps database errors to domain errors:

```go
func translatePersistenceError(err error, notFound, conflict *infraerrors.ApplicationError) error {
    if err == nil {
        return nil
    }
    if notFound != nil && (errors.Is(err, sql.ErrNoRows) || dbent.IsNotFound(err)) {
        return notFound.WithCause(err)
    }
    if conflict != nil && isUniqueConstraintViolation(err) {
        return conflict.WithCause(err)
    }
    return err
}
```

### In Handlers

Handlers convert errors to HTTP responses using the response package:

```go
func (h *EntityHandler) Get(c *gin.Context) {
    id, err := parseID(c.Param("id"))
    if err != nil {
        response.ErrorFrom(c, err)
        return
    }

    entity, err := h.service.GetByID(c.Request.Context(), id)
    if err != nil {
        response.ErrorFrom(c, err)
        return
    }

    response.Success(c, entity)
}
```

The `response.ErrorFrom` function:
1. Converts any error to `*ApplicationError` via `errors.ToHTTP`
2. Logs 5xx errors with full details for debugging
3. Writes a standardized JSON error response

---

## API Error Responses

All API errors follow a consistent JSON envelope:

```json
{
  "code": 404,
  "reason": "ENTITY_NOT_FOUND",
  "message": "entity not found",
  "metadata": {
    "field": "id",
    "value": "12345"
  },
  "data": null
}
```

### Standard HTTP Error Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| 400 | Bad Request | Invalid input, validation failure |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource, constraint violation |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Error | Unexpected server errors |
| 503 | Service Unavailable | Downstream service unavailable |
| 504 | Gateway Timeout | Upstream request timed out |

### Error Reason Codes

Reason codes follow the `SCOPED_DESCRIPTOR` convention:

| Reason | HTTP Code | Meaning |
|--------|-----------|---------|
| `ENTITY_NOT_FOUND` | 404 | Resource does not exist |
| `INVALID_INPUT` | 400 | Required field is missing or invalid |
| `UNAUTHORIZED` | 401 | Authentication required |
| `TOKEN_EXPIRED` | 401 | Token has expired |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `DUPLICATE_ENTRY` | 409 | Resource already exists |
| `RATE_LIMITED` | 429 | Rate limit exceeded |

---

## Error Propagation Best Practices

### DO

```go
// Wrap errors with context at service boundary
return nil, fmt.Errorf("create entity: %w", err)

// Use sentinel errors for known conditions
if errors.Is(err, service.ErrEntityNotFound) {
    return nil, err
}

// Translate persistence errors at repository boundary
return nil, translatePersistenceError(err, ErrNotFound, ErrDuplicate)

// Attach cause to preserve the original error chain
return ErrNotFound.WithCause(err)

// Add metadata for debugging
return ErrInvalidInput.WithMetadata(map[string]string{
    "field": "email",
    "value": providedEmail,
})
```

### DON'T

```go
// Don't lose error context
return nil, err  // loses what failed

// Don't create generic errors
return nil, errors.New("something went wrong")

// Don't log and return
logger.Error("failed", err)
return nil, err  // double-logging

// Don't expose internal details to clients
return nil, fmt.Errorf("database error: %v", err)  // leaks DB internals
```

---

## Panic Recovery

The recovery middleware converts panics into standardized JSON error responses:

```go
// From internal/server/middleware/recovery.go
func Recovery() gin.HandlerFunc {
    return gin.CustomRecoveryWithWriter(gin.DefaultErrorWriter, func(c *gin.Context, recovered any) {
        // Captures stack trace, request context
        // Returns 500 with panic details in metadata
    })
}
```

Broken pipe errors (`broken pipe`, `connection reset by peer`) are handled silently without writing a response.

---

## Common Mistakes

### 1. Not Translating ORM/Database Errors

Database errors should be translated to domain errors at the repository boundary:

```go
// Bad
m, err := client.Entity.Query().Only(ctx)
if err != nil {
    return nil, err  // exposes database internals
}

// Good
m, err := client.Entity.Query().Only(ctx)
if err != nil {
    return nil, translatePersistenceError(err, ErrEntityNotFound, nil)
}
```

### 2. Panic on Nil Pointer

Always check repository returns before dereferencing:

```go
// Bad
entity, _ := s.repo.GetByID(ctx, id)
entity.Name = "new"  // panic if entity is nil

// Good
entity, err := s.repo.GetByID(ctx, id)
if err != nil {
    return nil, err
}
```

### 3. Logging Errors Already Returned

Let the caller log errors at the appropriate level:

```go
// Bad - double logging
logger.Error("get entity failed", err)
return nil, err

// Good - let caller decide
return nil, err
```

### 4. Not Using Error Chains

Always use `%w` for wrapping to preserve error chains:

```go
// Bad - loses cause
return nil, fmt.Errorf("failed to process: %v", err)

// Good - preserves chain for errors.Is/errors.As
return nil, fmt.Errorf("failed to process: %w", err)
```

---

## Response Envelope

### Success Response

```json
{
  "code": 0,
  "message": "success",
  "data": { ... }
}
```

### Paginated Response

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [ ... ],
    "total": 100,
    "page": 1,
    "page_size": 20,
    "pages": 5
  }
}
```

### Async Accepted Response (HTTP 202)

```json
{
  "code": 0,
  "message": "accepted",
  "data": { ... }
}
```
