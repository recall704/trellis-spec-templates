# Quality Guidelines

> Code quality standards for backend development.

---

## Overview

The project enforces code quality through golangci-lint, comprehensive testing (unit + integration + e2e), and clear architectural boundaries. This document captures the key rules and patterns.

---

## Forbidden Patterns

### 1. Service Cannot Import Repository

The `service` layer must NOT import the `repository` layer. This architectural boundary is enforced by golangci-lint via `depguard`.

```go
// BAD - service importing repository
package service
import "yourmodule/internal/repository"

// GOOD - service only knows interfaces
package service
type EntityRepository interface {
    Create(ctx context.Context, entity *Entity) error
    GetByID(ctx context.Context, id int64) (*Entity, error)
}
```

**Exceptions:** A small set of ops-related service files may import repository for operational concerns. These are explicitly excluded from the lint rule.

### 2. Handler Cannot Import Repository

Handlers should delegate to services, not access data directly. Enforced by golangci-lint.

### 3. Service Cannot Import Infrastructure Directly

The `service` layer must NOT import Redis, GORM, or other infrastructure packages directly. Infrastructure access goes through repository or cache interfaces.

### 4. Use `any` Instead of `interface{}`

Go 1.18+ prefers `any` for empty interface. Enforced by gofmt rewrite rule:

```go
// Bad
func process(data interface{}) {}

// Good
func process(data any) {}
```

### 5. No Raw Error Wrapping Without Context

```go
// Bad - loses what operation failed
return nil, err

// Good - adds context
return nil, fmt.Errorf("create entity: %w", err)
```

---

## Required Patterns

### 1. Interface Definition in Service Layer

Repository interfaces are defined in the service package that uses them:

```go
// internal/service/entity_service.go
type EntityRepository interface {
    Create(ctx context.Context, entity *Entity) error
    GetByID(ctx context.Context, id int64) (*Entity, error)
    // ...
}
```

### 2. Soft Delete Always Applied

Ent automatically filters soft-deleted records via interceptors. For raw SQL:

```sql
-- Always include:
WHERE deleted_at IS NULL
```

### 3. Error Translation at Repository Boundary

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

### 4. Use Initialisms Correctly

Go initialisms should be all uppercase in exported names. Enforced by staticcheck:

```go
// Bad
func GetUrl() string
func SetApiKey(key string)

// Good
func GetURL() string
func SetAPIKey(key string)
```

### 5. Compile-Time Interface Assertions

Verify stub implementations satisfy interfaces at compile time:

```go
var _ service.ConcurrencyCache = StubConcurrencyCache{}
```

### 6. Test Fixtures with Functional Options

Create test data using functional options for flexibility:

```go
func NewTestEntity(opts ...func(*service.Entity)) *service.Entity {
    e := &service.Entity{
        ID:     1,
        Name:   "test-entity",
        Status: service.StatusActive,
    }
    for _, opt := range opts {
        opt(e)
    }
    return e
}

// Usage:
entity := NewTestEntity(func(e *service.Entity) {
    e.Status = service.StatusDisabled
})
```

---

## Testing Requirements

### Test File Naming

| Type | Suffix | Example |
|------|--------|---------|
| Unit tests | `_test.go` | `entity_service_test.go` |
| Integration tests | `_integration_test.go` | `entity_repo_integration_test.go` |
| Benchmark tests | `_benchmark_test.go` | `gateway_service_benchmark_test.go` |
| E2E tests | `e2e_*_test.go` | `e2e_gateway_test.go` |

### Test Build Tags

```go
//go:build unit

package service_test
```

| Tag | Purpose |
|-----|---------|
| `unit` | Unit tests with mocks/stubs |
| `integration` | Integration tests with real database |
| `e2e` | End-to-end tests against running server |

Run commands:

```bash
make test-unit        # go test -tags=unit ./...
make test-integration # go test -tags=integration ./...
make test-e2e-local   # go test -tags=e2e -v -timeout=300s ./internal/integration/...
make test             # go test ./... + golangci-lint
```

### Unit Test Patterns

Use table-driven tests with descriptive subtest names:

```go
func TestEntityService_Delete(t *testing.T) {
    tests := []struct {
        name      string
        exists    bool
        existsErr error
        deleteErr error
        wantErr   bool
    }{
        {
            name:    "not found returns error",
            exists:  false,
            wantErr: true,
        },
        {
            name:      "check error propagates",
            existsErr: errors.New("db down"),
            wantErr:   true,
        },
        {
            name:      "delete error propagates",
            exists:    true,
            deleteErr: errors.New("delete failed"),
            wantErr:   true,
        },
        {
            name:    "success returns nil",
            exists:  true,
            wantErr: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := &repoStub{exists: tt.exists, existsErr: tt.existsErr, deleteErr: tt.deleteErr}
            svc := &EntityService{repo: repo}

            err := svc.Delete(context.Background(), 55)
            if tt.wantErr {
                require.Error(t, err)
            } else {
                require.NoError(t, err)
            }
        })
    }
}
```

### Stub Pattern for Repository Interfaces

Use focused stubs that panic on unexpected calls to catch incorrect usage quickly:

```go
type repoStub struct {
    exists     bool
    existsErr  error
    deleteErr  error
    deletedIDs []int64
}

func (s *repoStub) ExistsByID(ctx context.Context, id int64) (bool, error) {
    return s.exists, s.existsErr
}

func (s *repoStub) Delete(ctx context.Context, id int64) error {
    s.deletedIDs = append(s.deletedIDs, id)
    return s.deleteErr
}

// Methods not used in this test panic to catch unexpected calls
func (s *repoStub) Create(ctx context.Context, entity *service.Entity) error {
    panic("unexpected Create call")
}
```

### Shared Test Utilities

Common stubs and fixtures live in `internal/testutil/`:

- **`stubs.go`** — shared stub implementations for common interfaces (cache, gateway, etc.)
- **`fixtures.go`** — test data factories with functional options
- **`httptest.go`** — Gin test context helpers

All test utility files use `//go:build unit` to exclude from production builds.

### Integration Test Patterns

Integration tests use a real database:

```go
//go:build integration

func TestEntityRepository_Create(t *testing.T) {
    client := enttest.NewClient(t)
    defer client.Close()

    repo := repository.NewEntityRepository(client, db)
    // test actual database operations
}
```

### E2E Test Patterns

E2E tests run against a live server via HTTP:

```go
//go:build e2e

func TestGateway_ChatCompletion(t *testing.T) {
    baseURL := getEnv("BASE_URL", "http://localhost:8080")
    apiKey := os.Getenv("TEST_API_KEY")
    require.NotEmpty(t, apiKey, "TEST_API_KEY must be set")

    resp, err := http.Post(baseURL+"/v1/chat/completions", "application/json", body)
    require.NoError(t, err)
    require.Equal(t, http.StatusOK, resp.StatusCode)
}
```

Use environment variables for secrets — never hardcode credentials.

---

## Linting Configuration

### Enabled Linters

| Linter | Purpose |
|--------|---------|
| `depguard` | Enforce architectural boundaries (service↛repository, handler↛repository) |
| `errcheck` | Ensure errors are checked |
| `gosec` | Security vulnerability detection |
| `govet` | Standard Go correctness checks |
| `ineffassign` | Detect ineffective assignments |
| `staticcheck` | Comprehensive static analysis |
| `unused` | Detect unused code |

### Key staticcheck Settings

- **Initialisms**: `ACL`, `API`, `ASCII`, `CPU`, `CSS`, `DNS`, `EOF`, `GUID`, `HTML`, `HTTP`, `HTTPS`, `ID`, `IP`, `JSON`, `QPS`, `RAM`, `RPC`, `SLA`, `SMTP`, `SQL`, `SSH`, `TCP`, `TLS`, `TTL`, `UDP`, `UI`, `GID`, `UID`, `UUID`, `URI`, `URL`, `UTF8`, `VM`, `XML`, `XMPP`, `XSRF`, `XSS`, `SIP`, `RTP`, `AMQP`, `DB`, `TS`
- **Checks**: All enabled except `ST1000` (package comment), `ST1003` (identifier naming), `ST1020-22` (exported comment format)

### Formatter Settings

- `gofmt` with rewrite rules:
  - `interface{}` → `any`
  - `a[b:len(a)]` → `a[b:]`

---

## Code Review Checklist

### Architecture
- [ ] `service` layer does not import `repository` (except approved exceptions)
- [ ] `handler` layer does not import `repository`
- [ ] `service` layer does not import infrastructure packages directly
- [ ] Repository interfaces defined in service package

### Error Handling
- [ ] Errors wrapped with context using `%w` at service boundary
- [ ] Persistence errors translated to domain errors
- [ ] No generic `errors.New()` without meaningful message
- [ ] No `panic` in production code paths (except test stubs)

### Database
- [ ] Soft delete filters applied in raw SQL
- [ ] Transactions used for multi-table operations
- [ ] N+1 queries avoided with edge preloading or batch loading

### Logging
- [ ] Sensitive data redacted before logging
- [ ] Structured logging used (not string concatenation)
- [ ] Errors not double-logged (let caller decide)

### Testing
- [ ] Unit tests for service layer logic
- [ ] Integration tests for repository layer
- [ ] Build tags correct (`unit` vs `integration` vs `e2e`)
- [ ] Table-driven tests for multiple scenarios
- [ ] Stubs panic on unexpected calls to catch incorrect usage

### Code Style
- [ ] `any` used instead of `interface{}`
- [ ] Initialisms in correct casing (URL, API, ID, JSON, etc.)
- [ ] `gofmt` applied
- [ ] golangci-lint passes
- [ ] No unused imports or variables

---

## Linting Commands

```bash
# Run all linters
golangci-lint run ./...

# Run unit tests
go test -tags=unit ./...

# Run integration tests
go test -tags=integration ./...

# Run E2E tests locally
go test -tags=e2e -v -timeout=300s ./internal/integration/...

# Run tests + linting
make test
```

---

## Common Anti-Patterns

1. **God objects**: Large service/repository structs that do too much — split by concern
2. **Premature optimization**: Complex batch operations when simple loops work
3. **Deep nesting**: Too many levels of if/else or error wrapping — use early returns
4. **Magic numbers**: Unnamed constants instead of named constants in `domain/`
5. **Shotgun surgery**: One change requires updating many files of the same type
6. **Leaky abstractions**: Service layer knowing about database/ORM details
7. **Silent failures**: Ignoring errors with `_` without justification
8. **Test coupling**: Tests that depend on execution order or shared mutable state
