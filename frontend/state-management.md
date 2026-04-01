# State Management

> How state is managed in this project.

---

## Overview

This project uses **Pinia** (Composition API style) for global state management, **Vue 3 Composition API** (`ref`, `reactive`, `computed`) for local component state, and **composables** for reusable stateful logic. There is no external server-state library (e.g., Vue Query) — each store implements its own caching, throttling, and request deduplication.

**Key decisions:**
- All stores use the function-based `defineStore('name', () => { ... })` syntax (setup stores)
- Stores live in `src/stores/` with a central barrel export in `src/stores/index.ts`
- Reusable stateful logic lives in `src/composables/` (e.g., `useTableLoader`, `useForm`)
- `localStorage` is used for auth persistence, theme, and admin settings cache
- Server-injected config (`window.__APP_CONFIG__`) eliminates flash on initial load

---

## State Categories

### Global State (Pinia Stores)

| Store | Purpose | Persistence |
|-------|---------|-------------|
| `auth` | User authentication, token management, auto-refresh | `localStorage` (token, user, refresh token, expiry) |
| `app` | UI state (sidebar, loading, toasts), public settings, version info | None (UI resets); public settings cached in-memory + injected config |
| `adminSettings` | Admin-only feature flags, ops monitoring toggles, custom menu items | `localStorage` (boolean/string caches to reduce flicker) |
| `subscriptions` | User subscription data with polling | None (60s in-memory cache + 5min polling) |
| `announcements` | User announcements, popup queue, read status | None (20min throttle) |
| `onboarding` | Tour driver instance, control method callbacks | `localStorage` (per-user/per-role seen flag) |

### Local State (Component-Level)

- `ref` / `reactive` for component-internal state (forms, modals, dialogs)
- `computed` for derived state within components
- Pattern: declare state near usage, lift to store only when shared across unrelated components

### Server State

No dedicated server-state library. Each store implements its own patterns:

- **Cache-then-fetch**: `fetch(force)` pattern with in-memory cache + `loaded`/`loading` flags
- **Request deduplication**: Promise reuse for in-flight requests (see `subscriptions.ts`)
- **Throttling**: Time-based cooldown to prevent duplicate fetches (see `announcements.ts` with 20min throttle)
- **Generation counter**: Invalidates stale in-flight responses (see `subscriptions.ts`)
- **Auto-refresh polling**: `setInterval` for real-time data (see `auth.ts` 60s user refresh, `subscriptions.ts` 5min polling)

### URL State

- Vue Router (`src/router/`) manages navigation state
- Query params used for filters, pagination, and shareable views
- Route-level code splitting via dynamic `import()`

---

## When to Use Global State

Promote state to a Pinia store when **any** of the following apply:

1. **Shared across unrelated components** — e.g., auth state needed by navbar, sidebar, and guards
2. **Survives navigation** — e.g., announcement popup queue, onboarding tour state
3. **Requires persistence** — e.g., auth tokens, admin settings cache
4. **Has complex lifecycle** — e.g., auto-refresh polling, token refresh scheduling

Keep state local (`ref`/`reactive`) when:

1. Used by a single component or parent-child pair
2. Related to form input, modal visibility, or UI toggles
3. No need to survive navigation or page reload

---

## Server State

### Caching Patterns

**Standard cache-then-fetch** (used by `app.ts`, `adminSettings.ts`):

```typescript
async function fetch(force = false) {
  if (loaded.value && !force) return
  if (loading.value) return // prevent concurrent duplicates

  loading.value = true
  try {
    const data = await apiCall()
    // apply to state
    loaded.value = true
  } finally {
    loading.value = false
  }
}
```

**Injected config for zero-flash** (`app.ts`):

```typescript
function initFromInjectedConfig(): boolean {
  if (window.__APP_CONFIG__) {
    applySettings(window.__APP_CONFIG__)
    return true
  }
  return false
}
```

Called synchronously in `main.ts` before `app.mount()` to prevent flash of default values.

**Request deduplication with generation counter** (`subscriptions.ts`):

```typescript
let requestGeneration = 0

async function fetchActiveSubscriptions(force = false) {
  if (activePromise && !force) return activePromise

  const currentGeneration = ++requestGeneration
  const requestPromise = apiCall().then((data) => {
    if (currentGeneration === requestGeneration) {
      // only apply if this is still the latest request
      activeSubscriptions.value = data
    }
    return data
  })
  activePromise = requestPromise
  return activePromise
}
```

### Error Recovery

- On fetch failure, throttle timestamps are reset to allow immediate retry (`announcements.ts`)
- Cached/default values are retained on transient failures — UI does not flip based on failed fetches (`adminSettings.ts`)

---

## Common Mistakes

1. **Mutating store state directly in components** — always use store actions. Direct mutation breaks reactivity tracking and bypasses business logic.

2. **Forgetting to clean up intervals/timeouts** — stores with `setInterval` (auth auto-refresh, subscription polling) must have corresponding `stop` methods called on logout or unmount.

3. **Not handling concurrent requests** — always guard with `loading.value` checks or promise deduplication. Race conditions cause duplicate API calls and stale state.

4. **Mixing `ref` and `reactive` inconsistently** — use `ref` for primitives and objects that may be replaced entirely; use `reactive` for objects where individual properties are mutated (e.g., `params` in `useTableLoader`).

5. **Storing non-serializable data in localStorage** — only store JSON-serializable data. Driver.js instances, callbacks, and DOM references must use `shallowRef`/`markRaw` and never be persisted.

6. **Forgetting `toRaw()` when passing reactive state to APIs** — `useTableLoader` uses `toRaw(params)` to avoid sending Vue proxy objects to the backend.

7. **Not protecting `localStorage` access** — wrap in try/catch (see `adminSettings.ts` helpers) to handle private browsing mode or storage quota errors.

8. **Starting polling without stopping** — every `startPolling()` / `startAutoRefresh()` must have a corresponding `stopPolling()` / `stopAutoRefresh()` called on logout or component unmount.
