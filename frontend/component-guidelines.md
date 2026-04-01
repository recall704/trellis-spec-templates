# Component Guidelines

> How components are built in this project.

---

## Overview

Components follow Vue 3 `<script setup lang="ts">` conventions with strict TypeScript typing for props, emits, and slots. Each component adheres to the single responsibility principle and is designed for reusability through configurable props/emits and composable logic extraction.

### Naming Conventions

- **Component names**: PascalCase, e.g., `AccountsView.vue`
- **File names**: Must match component name exactly
- **Variables**: camelCase, e.g., `isLoading`, `userList`
- **Constants**: UPPER_SNAKE_CASE, e.g., `API_BASE_URL`, `DEFAULT_LOCALE`
- **Types**: PascalCase, e.g., `PaginatedResponse<T>`
- **Events**: camelCase with `@` prefix, e.g., `@update:filters`, `@create`

---

## Component Structure

Standard component file layout:

```vue
<template>
  <!-- Template area -->
  <div class="component-name">
    <slot name="before"></slot>
    <slot></slot>
    <slot name="after"></slot>
  </div>
</template>

<script setup lang="ts">
// Imports
import { ref, computed } from 'vue'
import { useI18n } from 'vue-i18n'

// Props definition
interface Props {
  loading: boolean
  visible: boolean
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  visible: true
})

// Emits definition
interface Emits {
  (e: 'refresh'): void
  (e: 'create', data?: any): void
}

const emit = defineEmits<Emits>()

// Composables
const { t } = useI18n()

// Reactive data
const count = ref(0)
const isLoading = ref(props.loading)

// Computed
const showButton = computed(() => !props.loading && props.visible)

// Methods
function onClick() {
  if (props.loading) return
  emit('refresh')
}

// Lifecycle
onMounted(() => {
  // Initialization logic
})

onUnmounted(() => {
  // Cleanup logic
})
</script>

<style scoped>
/* Component styles must use scoped */
.component-name {
  padding: 16px;
}
</style>
```

### Section Order

1. Imports
2. Props definition (with `withDefaults`)
3. Emits definition
4. Composables
5. Reactive data (`ref`, `reactive`)
6. Computed properties
7. Methods (plain functions)
8. Lifecycle hooks

---

## Props Conventions

- **Must define types**: All props require explicit TypeScript interface definitions
- **Use `withDefaults`**: Provide default values for optional props
- **Avoid `any`**: Use specific types for all prop values

```vue
<script setup lang="ts">
interface Props {
  loading?: boolean    // Optional prop
  visible?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  visible: true
})
</script>
```

---

## Emits Conventions

- **Must define event types**: All emits require explicit TypeScript interface definitions
- **Avoid `any`**: Use specific types for event payloads

```vue
<script setup lang="ts">
interface Emits {
  (e: 'refresh'): void
  (e: 'sync', data?: any): void
}

const emit = defineEmits<Emits>()
</script>
```

---

## Slot Conventions

- **Default slot**: Every component must support a `default` slot
- **Named slots**: Use named slots for additional content (`before`, `after`, `header`, `footer`)
- **Scoped slots**: Pass data via `slot-scope="scope"`

```vue
<template>
  <div>
    <slot name="before"></slot>
    <slot></slot>
    <slot name="after"></slot>
  </div>
</template>
```

---

## Styling Patterns

### Global Styles

- **TailwindCSS**: Use utility classes for styling
- **Custom global styles**: `src/style.css` using CSS Modules or scoped

### Component Styles

- **Must use `scoped`**: All component `<style>` blocks must include the `scoped` attribute
- **BEM-like naming**: Use `.component-name__element` pattern for nested elements

```css
/* Must use scoped */
.component-name {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.component-name__header {
  font-size: 16px;
  font-weight: 600;
  color: var(--text-primary);
}
```

### Responsive Breakpoints

| Breakpoint | Size   | Target     |
|------------|--------|------------|
| `sm`       | 640px  | Mobile     |
| `md`       | 768px  | Tablet     |
| `lg`       | 1024px | Desktop    |
| `xl`       | 1280px | Large screen |
| `2xl`      | 1536px | Extra large |

---

## Accessibility

- Use semantic HTML elements
- Ensure keyboard navigability
- Provide proper ARIA attributes where needed

---

## Performance Optimization

### Component Optimization

- **Async components**: Use `() => import(...)` for lazy loading
- **Critical rendering**: Use `<script setup>` for optimized rendering
- **Reactive optimization**: Avoid unnecessary reactive data

### List Optimization

- **Virtual scrolling**: Use `@tanstack/vue-virtual` for large data lists
- **Pagination**: Limit page range with `handlePageChange`
- **Lazy loading**: Images and charts should use lazy loading

---

## Common Mistakes

### Event Bus (Deprecated)

```typescript
// ❌ Avoid using event bus (deprecated in Vue 3)
// ✅ Use Composables or Pinia state management instead
```

### Complex Computation in Templates

```vue
<!-- ❌ Avoid complex computation in templates -->
<template>
  <div v-if="!data">Loading...</div>
</template>

<!-- ✅ Use computed properties -->
<script setup>
const isLoading = computed(() => !data.value)
</script>

<template>
  <div v-if="isLoading">Loading...</div>
</template>
```

### Circular Dependencies

```typescript
// Avoid circular imports
import type { User } from '@/types'  // Use type import

// When necessary, use async import
const { default: User } = await import('@/types')
```

---

## Best Practices

- **Single Responsibility**: Each component should do one thing
- **Reusability**: Configure behavior through Props/Emits
- **Composability**: Extract logic into Composables
