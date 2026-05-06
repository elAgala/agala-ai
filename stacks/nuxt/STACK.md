# Stack: Nuxt Fullstack

## Detection

`nuxt.config.ts` exists at the project root.

---

## Key paths

| Path | Purpose |
|---|---|
| `server/api/` | API endpoints |
| `stores/` | Pinia stores |
| `components/` | Vue components |
| `pages/` | File-based routing |
| `composables/` | Shared Vue composables |
| `middleware/` | Route middleware |
| `nuxt.config.ts` | Framework config |

---

## Server routes

File name encodes the HTTP method: `orders.get.ts`, `orders.post.ts`, `orders/[id].delete.ts`.

### Validated POST endpoint

```typescript
// server/api/orders.post.ts
import { z } from 'zod'

const schema = z.object({
  productId: z.string().uuid(),
  quantity: z.number().int().min(1),
})

type OrderResponse = {
  id: string
  productId: string
  quantity: number
  createdAt: string
}

export default defineEventHandler(async (event): Promise<OrderResponse> => {
  const body = await readValidatedBody(event, schema.parse)
  const order = await createOrder(body)
  return { id: order.id, productId: order.productId, quantity: order.quantity, createdAt: order.createdAt.toISOString() }
})
```

### Validated GET endpoint with query params

```typescript
// server/api/orders.get.ts
import { z } from 'zod'

const querySchema = z.object({
  status: z.enum(['pending', 'completed']).optional(),
  page: z.coerce.number().int().min(1).default(1),
})

export default defineEventHandler(async (event) => {
  const query = await getValidatedQuery(event, querySchema.parse)
  return listOrders(query)
})
```

### HTTP errors

```typescript
throw createError({ statusCode: 404, statusMessage: 'Order not found' })
```

Rules:
- Always use `readValidatedBody` / `getValidatedQuery` — never raw `readBody`
- Always declare an explicit TypeScript return type on the handler
- Let Zod parse errors propagate — Nuxt returns 400 automatically
- Status codes: 200 / 201 / 204 / 400 / 401 / 403 / 404 / 409 / 500

---

## Pinia stores

Setup syntax only — no options API.

```typescript
// stores/cart.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

type CartItem = { productId: string; name: string; quantity: number; unitPrice: number }

export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])

  const total = computed(() =>
    items.value.reduce((sum, item) => sum + item.quantity * item.unitPrice, 0)
  )

  function addItem(item: CartItem) {
    const existing = items.value.find((i) => i.productId === item.productId)
    if (existing) { existing.quantity += item.quantity; return }
    items.value.push(item)
  }

  function removeItem(productId: string) {
    items.value = items.value.filter((i) => i.productId !== productId)
  }

  return { items, total, addItem, removeItem }
})
```

---

## Vue components

```vue
<script setup lang="ts">
type Props = {
  orderId: string
  status: 'pending' | 'completed'
}

const props = defineProps<Props>()
const emit = defineEmits<{ updated: [id: string] }>()
</script>
```

Rules:
- `<script setup>` always
- Named exports only — no default exports on components
- No business logic in templates — use `computed` or composables
- No inline complex expressions in `v-if` / `v-for`
- All `v-for` must have a `:key` with a stable unique value

---

## Data fetching

- Pages / layouts: use `useFetch` or `useAsyncData` (SSR-aware)
- Interactive client components: use TanStack Query (`@tanstack/vue-query`)
- Never call `$fetch` directly inside a component's logic — wrap in a composable or use `useAsyncData`

---

## Testing

- Runner: Vitest + `@nuxt/test-utils`
- Test files: `__tests__/<name>.test.ts` colocated with the source file
- Server route tests: mock the `h3` event or use `@nuxt/test-utils` test helpers
- Store tests: instantiate the store with `setActivePinia(createPinia())` in `beforeEach`

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useCartStore } from '@/stores/cart'

describe('useCartStore', () => {
  beforeEach(() => { setActivePinia(createPinia()) })

  it('adds an item to the cart', () => {
    const store = useCartStore()
    store.addItem({ productId: 'p1', name: 'Widget', quantity: 2, unitPrice: 10 })
    expect(store.items).toHaveLength(1)
    expect(store.total).toBe(20)
  })
})
```

---

## Conventions

- No `any` without an inline comment explaining why
- No `continue` or `break` in loops — use `filter` / `map` / `find` or early returns
- No `console.log` in delivered code
- No hardcoded secrets or API keys
- No new dependency without explicit user approval
