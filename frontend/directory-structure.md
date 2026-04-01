# Directory Structure

> How frontend code is organized in this project.

---

## Overview

The frontend follows a feature-driven directory structure with clear separation of concerns. Components, composables, and API modules are organized by domain (admin, user, auth) to improve discoverability and maintainability.

---

## Directory Layout

```
frontend/
├── src/
│   ├── api/                     # API client calls
│   │   ├── admin/               # Admin API modules
│   │   ├── sora.ts              # Sora streaming API
│   │   ├── keys.ts              # Key management API
│   │   └── client.ts            # Axios client wrapper
│   ├── components/              # Vue components
│   │   ├── admin/               # Admin components
│   │   ├── user/                # User components
│   │   ├── auth/                # Auth components
│   │   ├── charts/              # Chart components
│   │   ├── common/              # Shared/common components
│   │   └── icons/               # Icon components
│   ├── composables/             # Composable functions
│   │   ├── useAccountOAuth.ts   # OAuth auth logic
│   │   ├── useTableLoader.ts    # Table data loading
│   │   └── useSwipeSelect.ts    # Swipe selection logic
│   ├── i18n/                    # Internationalization
│   │   ├── locales/             # Language packs (en, zh)
│   │   └── index.ts             # i18n configuration
│   ├── router/                  # Router configuration
│   │   ├── index.ts             # Route definitions
│   │   └── meta.d.ts            # Route meta types
│   ├── stores/                  # Pinia state management
│   │   ├── app.ts               # Global state
│   │   ├── auth.ts              # Auth state
│   │   └── subscriptions.ts     # Subscription state
│   ├── types/                   # TypeScript type definitions
│   ├── utils/                   # Utility functions
│   ├── views/                   # Page views
│   │   ├── admin/               # Admin pages
│   │   ├── user/                # User pages
│   │   └── auth/                # Auth pages
│   ├── App.vue                  # Root component
│   ├── main.ts                  # Entry file
│   └── style.css                # Global styles
```

---

## Module Organization

### Feature Modules

New features should be organized by domain prefix:

| Prefix  | Purpose              | Location                        |
|---------|----------------------|---------------------------------|
| `admin` | Admin panel features | `views/admin/`, `components/admin/`, `api/admin/` |
| `user`  | User-facing features | `views/user/`, `components/user/` |
| `auth`  | Authentication       | `views/auth/`, `components/auth/`, `stores/auth.ts` |

### Shared Resources

- **`components/common/`** — Reusable UI components used across features
- **`components/charts/`** — Chart-specific components
- **`components/icons/`** — Icon components
- **`composables/`** — Shared composable functions (e.g., `useTableLoader`, `useAccountOAuth`)
- **`stores/`** — Pinia stores (import via `@/stores`)
- **`types/`** — TypeScript types (import via `@/types`)
- **`utils/`** — Utility functions

### Adding a New Feature

1. Create views in `src/views/{domain}/FeatureView.vue`
2. Create components in `src/components/{domain}/FeatureComponent.vue`
3. Create API module in `src/api/{domain}/feature.ts`
4. Add route in `src/router/index.ts`
5. Add i18n keys in `src/i18n/locales/{en,zh}/index.ts`

---

## Naming Conventions

### File Naming

| Type            | Convention    | Example                        |
|-----------------|---------------|--------------------------------|
| Components      | PascalCase    | `AccountsView.vue`             |
| Composables     | camelCase     | `useTableLoader.ts`            |
| API modules     | camelCase     | `accounts.ts`                  |
| Stores          | camelCase     | `auth.ts`                      |
| Type files      | camelCase     | `api.ts`, `user.ts`            |
| Route meta types| camelCase     | `meta.d.ts`                    |

### Import Patterns

```typescript
// Correct
import { list, create, update } from '@/api/admin/accounts'
import { useTableLoader } from '@/composables/useTableLoader'
import { useAuthStore } from '@/stores'
import type { Account, PaginatedResponse } from '@/types'

// Incorrect
import accounts from '@/api/admin'        // Avoid importing entire modules
import { useApp } from '@/stores/app'     // Use barrel export from @/stores
import Account from '@/types/account'     // Use type import
```

---

## Examples

Well-structured modules to reference:

- **Accounts module**: `views/admin/AccountsView.vue` + `components/admin/` + `api/admin/accounts.ts`
- **Auth module**: `stores/auth.ts` + `composables/useAccountOAuth.ts` + `views/auth/`
