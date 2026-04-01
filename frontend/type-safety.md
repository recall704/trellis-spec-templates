# Type Safety

> Type safety patterns in this project.

---

## Overview

This project uses **TypeScript** with **strict mode** enabled across the entire codebase. The Vue 3 Composition API (`<script setup lang="ts">`) provides full type inference for reactive state, props, and emits. There is no runtime validation library — type safety is enforced at compile time via TypeScript, with `noUnusedLocals`, `noUnusedParameters`, and `noFallthroughCasesInSwitch` all enabled.

**Key decisions:**
- `strict: true` — full strict mode including `strictNullChecks`, `strictFunctionTypes`, etc.
- All Vue components use `<script setup lang="ts">`
- API layer wraps responses in typed generics: `apiClient.get<T>()`, `apiClient.post<T>()`
- All business domain types live in `src/types/index.ts` (single source of truth)
- Global type augmentation via `src/types/global.d.ts` and `src/vite-env.d.ts`

---

## Type Organization

### Central Type Registry (`src/types/index.ts`)

All shared types are exported from a single barrel file, organized by domain:

| Section | Types |
|---------|-------|
| Common | `SelectOption`, `BasePaginationResponse<T>`, `FetchOptions` |
| User & Auth | `User`, `AdminUser`, `LoginRequest`, `RegisterRequest`, `AuthResponse`, `PublicSettings` |
| Subscription | `Subscription`, `UserSubscription`, `SubscriptionProgress` |
| Announcement | `Announcement`, `UserAnnouncement`, `AnnouncementTargeting` |
| Proxy Node | `ProxyNode`, `ConversionRequest`, `ConversionResult` |
| API Key & Group | `ApiKey`, `Group`, `AdminGroup`, `CreateApiKeyRequest` |
| Account & Proxy | `Account`, `Proxy`, `AccountPlatform`, `AccountType`, `CreateAccountRequest` |
| Usage & Redeem | `UsageLog`, `AdminUsageLog`, `RedeemCode`, `UsageCleanupTask` |
| Dashboard & Statistics | `DashboardStats`, `TrendDataPoint`, `ModelStat`, `UserSpendingRankingResponse` |
| Admin User Management | `UpdateUserRequest`, `ChangePasswordRequest` |
| Query Parameters | `UsageQueryParams` |
| User Attributes | `UserAttributeDefinition`, `UserAttributeValue` |
| Promo Code | `PromoCode`, `PromoCodeUsage` |
| TOTP (2FA) | `TotpStatus`, `TotpSetupRequest`, `TotpLoginResponse` |
| Scheduled Test | `ScheduledTestPlan`, `ScheduledTestResult` |
| UI State | `Toast`, `ToastType`, `AppState`, `ValidationError` |
| Table/List | `SortConfig`, `FilterConfig`, `PaginationConfig` |

### Global Augmentations

- **`src/types/global.d.ts`** — Augments `Window` with `__APP_CONFIG__?: PublicSettings` for server-injected config
- **`src/vite-env.d.ts`** — Declares `ImportMetaEnv` with `VITE_API_BASE_URL` and `BASE_URL`, plus `*.vue` module declarations

### Local Types

Types scoped to a single component or composable are defined inline:

- **Composables**: callback signatures and options interfaces (e.g., `OnboardingOptions` in `useOnboardingTour.ts`)
- **Components**: event payload types, form state shapes
- **API modules**: request/response types specific to a single endpoint (when not shared)

### Shared vs Local Rule

If a type is referenced by **more than one file**, it belongs in `src/types/index.ts`. If it is used by only one file, keep it local.

---

## Validation

This project does **not** use a runtime validation library (no Zod, Yup, or io-ts). Validation is handled at two levels:

1. **Compile-time**: TypeScript types enforce shape at call sites and in component props
2. **Server-side**: The Go backend validates all inputs; the frontend displays returned `ApiError` messages

### Error Type Handling

The API client (`src/api/client.ts`) unwraps the standard `{ code, message, data }` response envelope via interceptors. On error, it rejects with a structured object:

```typescript
{
  status: number,
  code?: string | number,
  message: string,
  detail?: string
}
```

Components catch errors with `catch (error: any)` — this is the primary location where `any` is tolerated, because the rejection shape is not formally typed.

### Form Validation

Form validation is handled by UI component libraries (e.g., Naive UI form rules) with inline rule objects. The `ValidationError` type in `src/types/index.ts` describes the shape of server-side validation errors:

```typescript
export interface ValidationError {
  field: string
  message: string
}
```

---

## Common Patterns

### Generic API Responses

All API functions use generic type parameters to ensure typed responses:

```typescript
async function list(page: number, pageSize: number): Promise<PaginatedResponse<Group>> {
  const { data } = await apiClient.get<PaginatedResponse<Group>>('/admin/groups', { params })
  return data
}
```

### Discriminated Unions for Status Fields

Status fields use string literal unions for exhaustive type narrowing:

```typescript
type AccountPlatform = 'anthropic' | 'openai' | 'gemini' | 'antigravity' | 'sora'
type AccountType = 'oauth' | 'setup-token' | 'apikey' | 'upstream' | 'bedrock' | 'chat_api'
type AnnouncementStatus = 'draft' | 'active' | 'archived'
```

### Interface Extension for Admin Views

Admin-specific types extend base types with additional fields:

```typescript
export interface AdminUser extends User {
  notes: string
  group_rates?: Record<number, number>
  current_concurrency?: number
}

export interface AdminGroup extends Group {
  model_routing: Record<string, number[]> | null
  model_routing_enabled: boolean
}
```

### Type-Safe Store Returns

Pinia setup stores explicitly return typed state and actions:

```typescript
return {
  // State
  sidebarCollapsed,
  loading,
  // Computed
  hasActiveToasts,
  // Actions
  toggleSidebar,
  setLoading,
}
```

### Request/Response Type Pairs

Every CRUD entity has matching `CreateXRequest` / `UpdateXRequest` types with optional fields on update:

```typescript
export interface CreateGroupRequest {
  name: string              // required
  platform?: GroupPlatform  // optional with default
  // ...
}

export interface UpdateGroupRequest {
  name?: string             // all optional
  platform?: GroupPlatform
  // ...
}
```

### Readonly for Protected State

Store state that should not be mutated externally is wrapped with `readonly()`:

```typescript
return {
  runMode: readonly(runMode),
  // ...
}
```

### `markRaw` for Non-Reactive Objects

Non-serializable objects stored in reactive state use `markRaw` to prevent Vue from making them reactive:

```typescript
driverInstance.value = driver ? markRaw(driver) : null
```

---

## Forbidden Patterns

### `any` — Minimize, Never in Shared Types

`any` is **forbidden** in `src/types/index.ts` and all shared type definitions. It is tolerated only in:

- Test files (`__tests__/*.spec.ts`) — for mock setup
- Catch clauses (`catch (err: any)`) — because rejection shape is untyped
- Temporary API params (`Record<string, any>`) — when building dynamic query objects

The only intentional `any` in shared types is `[key: string]: any` on `SelectOption` to support custom template properties.

### Type Assertions (`as`) — Use Sparingly

`as` assertions are discouraged. Prefer:

- Proper typing at the source (e.g., typed API responses)
- Type guards for narrowing
- `satisfies` operator for validating object shapes (when available)

Current acceptable uses:
- `as any` in catch blocks
- `as const` for literal type narrowing
- DOM element assertions in tests

### `!` Non-Null Assertion — Avoid

The non-null assertion operator (`!`) is not used in production code. Use optional chaining (`?.`) or explicit null checks instead.

### Untyped `ref()` / `reactive()`

Always provide explicit type parameters or let TypeScript infer from the initial value:

```typescript
// Good — explicit
const user = ref<User | null>(null)

// Good — inferred
const loading = ref(false)

// Bad — implicit any
const data = ref()
```

### Missing `lang="ts"` on Script Blocks

All Vue components must use `<script setup lang="ts">`. Plain `<script>` blocks are not allowed.
