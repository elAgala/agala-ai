---
name: nuxt-server-route
description: >
  Add a typed API endpoint in a Nuxt 3 project under server/api/ with Zod validation,
  createError for HTTP errors, and a typed response. Triggered by phrases like
  "add a server route", "create an API endpoint in Nuxt", "add a POST endpoint",
  "create server/api/ handler", "add a Nuxt API route".
---

## What It Does

Creates a Nuxt 3 server route with:
- Zod schema for request body or query validation
- `createError` for typed HTTP error responses
- Explicit TypeScript return type

---

## When to Use It

- Adding a new API endpoint to a Nuxt 3 project
- The endpoint lives under `server/api/`

---

## Steps

### File Location

```
server/api/<resource>[/<id>].ts
server/api/<resource>.post.ts   ← method-specific file
server/api/<resource>.get.ts
```

### Example — POST endpoint with body validation

`server/api/orders.post.ts`

```typescript
import { z } from 'zod'

const createOrderSchema = z.object({
  productId: z.string().uuid(),
  quantity: z.number().int().min(1),
})

type CreateOrderResponse = {
  id: string
  productId: string
  quantity: number
  createdAt: string
}

export default defineEventHandler(async (event): Promise<CreateOrderResponse> => {
  const body = await readValidatedBody(event, createOrderSchema.parse)

  const order = await createOrder(body)

  return {
    id: order.id,
    productId: order.productId,
    quantity: order.quantity,
    createdAt: order.createdAt.toISOString(),
  }
})
```

### Example — GET endpoint with query param validation

`server/api/orders.get.ts`

```typescript
import { z } from 'zod'

const querySchema = z.object({
  status: z.enum(['pending', 'completed', 'cancelled']).optional(),
  page: z.coerce.number().int().min(1).default(1),
})

type OrderListResponse = {
  items: { id: string; status: string }[]
  total: number
}

export default defineEventHandler(async (event): Promise<OrderListResponse> => {
  const query = await getValidatedQuery(event, querySchema.parse)

  const { items, total } = await listOrders(query)

  return { items, total }
})
```

### Throwing HTTP errors

```typescript
import { createError } from 'h3'

const order = await findOrder(id)
if (!order) {
  throw createError({ statusCode: 404, statusMessage: 'Order not found' })
}
```

### Validation error — let Zod propagate

If `readValidatedBody` or `getValidatedQuery` throws (Zod parse error), Nuxt automatically
returns a 400. Do not wrap validation in try/catch to swallow this.

---

## What to Avoid

- Using `readBody` without validation — always use `readValidatedBody`
- Returning untyped objects from event handlers
- Throwing plain `new Error()` instead of `createError`
- Putting business logic directly in the handler — call a service or utility function
- Using `any` for request or response types
