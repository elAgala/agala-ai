---
name: vue-repository-method
description: >
  Add a frontend repository method and expose it via TanStack Query factory pattern
  in a Vue 3 SPA. Triggered by phrases like "add a frontend API call", "fetch data
  from the backend in Vue", "add a TanStack Query query", "create a useQuery for X",
  "add a repository method to the frontend".
---

## What It Does

Adds a typed API call to a repository module and wires it into a TanStack Query
query factory, so components never call `fetch` directly.

---

## When to Use It

- Fetching or mutating data from the Go backend in the Vue 3 SPA
- The project is a monorepo with `/frontend` using Vite + Vue 3 + TanStack Query

---

## Project Structure

```
frontend/
  src/
    api/
      orders.ts         <- repository: raw HTTP calls
    queries/
      orders.ts         <- TanStack Query factory
    components/
      OrderDetail.vue
```

---

## Steps

### 1. Repository — typed HTTP call

`frontend/src/api/orders.ts`

```typescript
const BASE_URL = import.meta.env.VITE_API_URL

export type Order = {
  id: string
  productId: string
  quantity: number
  status: string
  createdAt: string
}

export type CreateOrderPayload = {
  productId: string
  quantity: number
}

async function fetchJSON<T>(path: string, init?: RequestInit): Promise<T> {
  const response = await fetch(`${BASE_URL}${path}`, {
    headers: { 'Content-Type': 'application/json' },
    ...init,
  })

  if (!response.ok) {
    const body = await response.json().catch(() => ({}))
    throw new Error(body.error ?? `HTTP ${response.status}`)
  }

  return response.json() as Promise<T>
}

export const ordersApi = {
  getById: (id: string) =>
    fetchJSON<Order>(`/api/v1/orders/${id}`),

  create: (payload: CreateOrderPayload) =>
    fetchJSON<Order>('/api/v1/orders', {
      method: 'POST',
      body: JSON.stringify(payload),
    }),
}
```

### 2. Query factory

`frontend/src/queries/orders.ts`

```typescript
import { queryOptions, useMutation, useQueryClient } from '@tanstack/vue-query'
import { ordersApi, type CreateOrderPayload } from '@/api/orders'

export const orderQueries = {
  detail: (id: string) =>
    queryOptions({
      queryKey: ['orders', id],
      queryFn: () => ordersApi.getById(id),
      staleTime: 1000 * 60,
    }),
}

export function useCreateOrder() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (payload: CreateOrderPayload) => ordersApi.create(payload),
    onSuccess: (order) => {
      queryClient.setQueryData(['orders', order.id], order)
    },
  })
}
```

### 3. Using the query in a component

`frontend/src/components/OrderDetail.vue`

```vue
<script setup lang="ts">
import { useQuery } from '@tanstack/vue-query'
import { orderQueries } from '@/queries/orders'

const props = defineProps<{ orderId: string }>()

const { data: order, isPending, isError } = useQuery(orderQueries.detail(props.orderId))
</script>

<template>
  <div v-if="isPending">Loading...</div>
  <div v-else-if="isError">Failed to load order.</div>
  <div v-else>
    <p>Order #{{ order.id }}</p>
    <p>Status: {{ order.status }}</p>
  </div>
</template>
```

### 4. Using the mutation in a component

```vue
<script setup lang="ts">
import { useCreateOrder } from '@/queries/orders'

const { mutate: createOrder, isPending } = useCreateOrder()

function submit() {
  createOrder({ productId: 'abc-123', quantity: 2 })
}
</script>
```

---

## What to Avoid

- Calling `fetch` directly inside components or Pinia stores
- Putting query keys as raw strings in components — always use the factory
- Skipping error handling in the repository — always check `response.ok`
- Using `any` for API response types
- Duplicating query key arrays across files — define them once in the factory
