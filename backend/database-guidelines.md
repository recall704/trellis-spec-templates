# Database Guidelines

> Database patterns and conventions for this project.

---

## Overview

The project uses **PostgreSQL** as the primary database with **Ent ORM** for type-safe queries. SQL migrations are the authoritative source of truth for schema — Ent schema is used for code generation, not for applying migrations.

---

## ORM and Query Patterns

### Ent ORM Usage

Ent provides type-safe database access with code generation.

```go
// Generated types from ent/schema
import dbentity "yourmodule/ent/entity"

// Querying with type-safe predicates
records, err := client.Entity.Query().
    Where(dbentity.StatusEQ("active")).
    Where(dbentity.PlatformEQ("example")).
    Order(ent.Asc(dbentity.FieldPriority)).
    All(ctx)
```

### When to Use Raw SQL

Use raw SQL for:
- Complex batch updates with CASE expressions
- JSONB manipulation (`jsonb_set`, `||` merge, `-` key removal)
- Window functions and aggregation
- Performance-critical queries that Ent cannot optimize
- Partial index queries that Ent predicates cannot express

Example — JSONB merge for atomic field updates:

```go
result, err := client.ExecContext(ctx,
    "UPDATE entities SET extra = COALESCE(extra, '{}'::jsonb) || $1::jsonb, updated_at = NOW() WHERE id = $2 AND deleted_at IS NULL",
    string(payload), id,
)
```

Example — JSONB partial key removal:

```go
result, err := client.ExecContext(ctx,
    "UPDATE entities SET extra = COALESCE(extra, '{}'::jsonb) - 'obsolete_key', updated_at = NOW() WHERE id = $1 AND deleted_at IS NULL",
    id,
)
```

Example — JSON path queries with Ent's `sqljson` helpers:

```go
import "entgo.io/ent/dialect/sql/sqljson"

m, err := client.Entity.Query().
    Where(func(s *entsql.Selector) {
        s.Where(sqljson.ValueEQ(entity.FieldExtra, "value", sqljson.Path("nested_key")))
    }).
    Only(ctx)
```

### Query Patterns

#### Soft Delete

All queries should filter out soft-deleted records. Ent mixins handle this automatically via interceptors.

```go
// SoftDeleteMixin automatically adds `deleted_at IS NULL` to all queries
records, err := client.Entity.Query().All(ctx) // automatically filters deleted
```

To query or delete including soft-deleted records, use the skip mechanism:

```go
ctx = mixins.SkipSoftDelete(ctx)
records, err := client.Entity.Query().All(ctx) // includes soft-deleted
```

#### Pagination

Use offset/limit pagination:

```go
records, err := q.
    Offset(params.Offset()).
    Limit(params.Limit()).
    All(ctx)
```

#### N+1 Prevention

Preload edges when fetching related data:

```go
// WithRelated() preloads the edge to avoid N+1
records, err := client.Entity.
    Query().
    Where(dbentity.IDIn(ids...)).
    WithRelated().
    All(ctx)

// Access preloaded data without additional queries
for _, r := range records {
    related := r.Edges.Related
}
```

#### Batch Operations

Use `pq.Array` for efficient `IN` clause queries with large ID sets:

```go
import "github.com/lib/pq"

result, err := r.sql.ExecContext(ctx,
    "UPDATE entities SET status = $1, updated_at = NOW() WHERE id = ANY($2) AND deleted_at IS NULL",
    newStatus, pq.Array(ids),
)
```

Use CASE expressions for batch updates with different values per row:

```go
caseSQL := "UPDATE entities SET priority = CASE id"
for i, id := range ids {
    caseSQL += fmt.Sprintf(" WHEN $%d THEN $%d", i*2+1, i*2+2)
    args = append(args, id, priorities[i])
}
caseSQL += " END, updated_at = NOW() WHERE id = ANY($?) AND deleted_at IS NULL"
args = append(args, pq.Array(ids))
```

---

## Migrations

### Migration Philosophy

SQL migrations in `migrations/` are the **authoritative schema source**. Do NOT use Ent's auto-migrate in production.

### Creating Migrations

1. Create a new SQL file in `migrations/` following the naming convention: `XXX_description.sql`
2. Use descriptive names: `004_add_account_scheduler_fields.sql`
3. Include both UP (forward) migration
4. Use `CREATE TABLE IF NOT EXISTS` and `CREATE INDEX IF NOT EXISTS` for idempotency
5. For partial unique indexes with soft delete, use `WHERE deleted_at IS NULL`:

```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_entities_email_active
    ON entities(email) WHERE deleted_at IS NULL;
```

### Running Migrations

Migrations run automatically on startup:

```go
if err := applyMigrationsFS(migrationCtx, drv.DB(), migrations.FS); err != nil {
    return nil, nil, err
}
```

### After Schema Changes

When modifying Ent schemas:

```bash
cd backend
go generate ./ent
# Commit both the schema changes AND generated files
```

---

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Table names | snake_case, plural | `accounts`, `user_groups` |
| Column names | snake_case | `created_at`, `rate_multiplier` |
| Index names | `idx_<table>_<columns>` | `idx_accounts_platform_priority` |
| Foreign keys | `<table>_<referenced_table>_fkey` | `accounts_proxy_id_fkey` |
| Unique indexes (partial) | `idx_<table>_<columns>_<qualifier>` | `idx_entities_email_active` |

### Field Naming in Ent Schemas

```go
// Good: descriptive, consistent
field.String("platform")           // snake_case
field.Int("concurrency")           // singular
field.Time("rate_limit_reset_at")  // _at suffix for timestamps
field.JSON("credentials", map[string]any{})  // JSONB for complex data

// Bad: inconsistent with database conventions
field.String("platformName")       // PascalCase
field.Int("maxConcurrency")        // Hungarian notation
```

---

## Schema Design Patterns

### Mixins

Use mixins for cross-cutting schema concerns:

**TimeMixin** — automatic `created_at` / `updated_at`:

```go
func (TimeMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Time("created_at").
            Immutable().
            Default(time.Now).
            SchemaType(map[string]string{dialect.Postgres: "timestamptz"}),
        field.Time("updated_at").
            Default(time.Now).
            UpdateDefault(time.Now).
            SchemaType(map[string]string{dialect.Postgres: "timestamptz"}),
    }
}
```

**SoftDeleteMixin** — automatic `deleted_at` with query interceptors and delete hooks:

```go
func (SoftDeleteMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Time("deleted_at").
            Optional().
            Nillable().
            SchemaType(map[string]string{dialect.Postgres: "timestamptz"}),
    }
}
```

### JSONB Fields

Use JSONB for flexible, schema-less data:

```go
field.JSON("credentials", map[string]any{}).
    Default(func() map[string]any { return map[string]any{} }).
    SchemaType(map[string]string{dialect.Postgres: "jsonb"})

field.JSON("extra", map[string]any{}).
    Default(func() map[string]any { return map[string]any{} }).
    SchemaType(map[string]string{dialect.Postgres: "jsonb"})
```

### Edges and Relationships

Define edges for relationships, using `Through` for many-to-many with join tables:

```go
// Many-to-many through a join table
edge.To("groups", Group.Type).
    Through("entity_groups", EntityGroup.Type),

// Optional one-to-one with explicit field
edge.To("proxy", Proxy.Type).
    Field("proxy_id").
    Unique(),
```

### Indexes

Define indexes in Ent schemas for readability; create actual indexes (including partial indexes) via SQL migrations:

```go
func (Entity) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("status"),
        index.Fields("platform", "priority"),  // composite
        index.Fields("deleted_at"),             // soft delete filter
    }
}
```

---

## Connection Pooling

Configure database connection pool settings for production:

```go
db.SetMaxOpenConns(maxOpen)
db.SetMaxIdleConns(maxIdle)
db.SetConnMaxLifetime(lifetime)
db.SetConnMaxIdleTime(idleTime)
```

---

## Common Mistakes

### 1. Forgetting Soft Delete Filter

Always use Ent's query methods which apply soft delete filters automatically. If writing raw SQL:

```sql
-- Good: include deleted_at filter
WHERE deleted_at IS NULL

-- Bad: missing soft delete filter
WHERE id = $1
```

### 2. N+1 Queries with Edges

```go
// Bad: N+1 query
for _, record := range records {
    related, _ := client.Related.Query().Where(related.IDEQ(record.RelatedID)).Only(ctx)
}

// Good: preload edges
records, _ := client.Entity.Query().WithRelated().All(ctx)
for _, record := range records {
    related := record.Edges.Related  // no additional query
}
```

### 3. Mutating JSONB Incorrectly

```go
// Bad: overwrites entire JSONB field
UPDATE entities SET extra = '{"key": "value"}' WHERE id = $1

// Good: merges atomically using || operator
UPDATE entities SET extra = COALESCE(extra, '{}'::jsonb) || '{"key": "value"}'::jsonb WHERE id = $1

// Good: removes a specific key atomically
UPDATE entities SET extra = COALESCE(extra, '{}'::jsonb) - 'obsolete_key' WHERE id = $1
```

### 4. Not Using Transactions for Multi-Table Operations

```go
// Good: wrap related operations in transaction
tx, err := client.Tx(ctx)
if err != nil && !errors.Is(err, ent.ErrTxStarted) {
    return err
}
defer tx.Rollback()

// operations...

return tx.Commit()
```

Handle nested transactions by checking for `ErrTxStarted` and reusing the existing client.

### 5. Forgetting to Handle Optional/Nillable Fields on Clear

When updating optional fields via Ent's builder, use `Clear<Field>()` to set them back to NULL:

```go
builder := client.Entity.UpdateOneID(id)
if value != nil {
    builder.SetOptionalField(*value)
} else {
    builder.ClearOptionalField()
}
```

---

## Field Type Mapping

| Go Type | PostgreSQL Type | Ent SchemaType |
|---------|-----------------|----------------|
| `string` | varchar/text | Default |
| `int` | integer | Default |
| `int64` | bigint | Default |
| `float64` | decimal | `SchemaType(map[string]string{dialect.Postgres: "decimal(10,4)"})` |
| `bool` | boolean | Default |
| `time.Time` | timestamptz | `SchemaType(map[string]string{dialect.Postgres: "timestamptz"})` |
| `map[string]any` | jsonb | `SchemaType(map[string]string{dialect.Postgres: "jsonb"})` |
| `[]byte` | bytea | Default |

---

## Error Handling

Translate database errors to domain errors at the repository boundary:

```go
result, err := builder.Save(ctx)
if err != nil {
    return translatePersistenceError(err, service.ErrNotFound, nil)
}
```

Use `ent.IsNotFound(err)` to check for missing records:

```go
if ent.IsNotFound(err) {
    return nil, nil
}
```
