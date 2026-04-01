# Quality Guidelines

> Code quality standards for frontend development.

---

## Overview

This document defines the quality standards, forbidden patterns, testing requirements, and code review checklist for the frontend codebase. All PRs must meet these standards before merging.

---

## Forbidden Patterns

### Event Bus (Deprecated in Vue 3)

```typescript
// ❌ Never use event bus
// ✅ Use Composables or Pinia state management instead
```

### Importing Entire Modules

```typescript
// ❌ Avoid importing entire modules
import accounts from '@/api/admin'

// ✅ Import only what you need
import { list, create, update } from '@/api/admin/accounts'
```

### Using `any` Type

```typescript
// ❌ Avoid using `any`
interface Emits {
  (e: 'sync', data?: any): void
}

// ✅ Use specific types
interface Emits {
  (e: 'sync', data?: SyncPayload): void
}
```

### Complex Computation in Templates

```vue
<!-- ❌ Avoid complex computation in templates -->
<template>
  <div v-if="!data || data.length === 0">No data</div>
</template>

<!-- ✅ Use computed properties -->
<script setup>
const isEmpty = computed(() => !data.value || data.value.length === 0)
</script>
```

### Circular Dependencies

```typescript
// ❌ Avoid circular imports
import { User } from '@/types'

// ✅ Use type import to avoid circular dependency
import type { User } from '@/types'

// ✅ When necessary, use async import
const { default: User } = await import('@/types')
```

### Direct Store Access Instead of Barrel Export

```typescript
// ❌ Direct store path
import { useApp } from '@/stores/app'

// ✅ Barrel export
import { useAuthStore } from '@/stores'
```

### Non-Type Imports for Types

```typescript
// ❌ Regular import for types
import Account from '@/types/account'

// ✅ Use type import
import type { Account } from '@/types'
```

---

## Required Patterns

### Component Structure

- Must use `<script setup lang="ts">`
- Props must be defined with TypeScript interface and `withDefaults`
- Emits must be defined with TypeScript interface
- Styles must use `scoped` attribute

### API Functions

- Must use standard naming: `list`, `create`, `update`, `deleteById`
- Must include JSDoc comments with `@param` and `@returns`
- Must return typed promises (e.g., `Promise<PaginatedResponse<T>>`)

### Composables

- Must follow the `use{Name}` naming convention
- Must return reactive state and methods
- Must include cleanup in `onUnmounted` when necessary (cancel requests, clear timers)

### i18n

- All user-facing text must use `t()` function
- Translation keys must follow the pattern: `{domain}.{feature}.{key}`

### Async Operations

- Must use `async/await` instead of `.then()` chains
- Must handle errors with `try/catch`

```typescript
async function fetchData() {
  const data = await apiClient.get('/data')
  return data
}
```

---

## Testing Requirements

### Test File Locations

```
src/
├── __tests__/                    # Root-level tests
├── api/
│   └── __tests__/                # API tests
├── composables/
│   └── __tests__/                # Composable tests
├── components/
│   └── __tests__/                # Component tests
└── views/
    └── __tests__/                # Page tests
```

### Testing Tools

| Tool                  | Purpose                     |
|-----------------------|-----------------------------|
| **Vitest**            | Test framework              |
| **@vue/test-utils**   | Vue component testing       |
| **jsdom**             | DOM environment simulation  |

### Test Structure

```typescript
import { describe, it, expect } from 'vitest'
import { useTableLoader } from '@/composables/useTableLoader'

describe('useTableLoader', () => {
  it('should load data correctly', () => {
    const mockData = [{ id: 1 }, { id: 2 }]
    // Test implementation
  })
})
```

### Run Commands

```bash
pnpm test:run    # Run tests
```

---

## Linting Rules

### ESLint + TypeScript Strict Mode

- ESLint with TypeScript strict mode is enforced
- Use `pnpm` for package management (never npm)

### Run Commands

```bash
pnpm lint        # Fix ESLint issues
pnpm lint:check  # Check only (no fixes)
pnpm typecheck   # TypeScript type check
```

---

## Code Review Checklist

When reviewing frontend code, check the following:

1. **Type Safety**: Are TypeScript types used correctly? No `any` types?
2. **Reactivity**: Are `ref`, `reactive`, `computed` used appropriately?
3. **Performance**: Are unnecessary reactive updates avoided?
4. **Testability**: Is the code testable? Does it avoid external state dependencies?
5. **Accessibility**: Is semantic HTML used? Are ARIA attributes present where needed?
6. **i18n**: Are all user-facing strings using `t()` for translation?
7. **Component Structure**: Does the component follow the standard section order?
8. **Props/Emits**: Are props and emits properly typed with interfaces?
9. **Styles**: Are component styles using `scoped`?
10. **Imports**: Are imports following the correct patterns (type imports for types, specific imports)?

---

## Git Commit Conventions

### Format

```
type(scope): description
```

### Examples

```bash
fix(api): fix token refresh error
feat(components): add AccountsView component
docs(i18n): update Chinese translations
refactor(stores): optimize auth store performance
```

---

## Development Workflow Commands

```bash
# Install dependencies
pnpm install

# Development mode
pnpm dev

# Production build (includes type checking)
pnpm build

# Code quality
pnpm lint        # Fix ESLint issues
pnpm lint:check  # Check only
pnpm typecheck   # TypeScript check
pnpm test:run    # Run tests
```
