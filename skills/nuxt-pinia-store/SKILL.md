---
name: nuxt-pinia-store
description: >
  Create a Pinia store in a Nuxt 3 project using setup store syntax, with computed
  getters and async actions. Triggered by phrases like "add a Pinia store",
  "create a store for X", "add state management for X", "set up a Pinia store",
  "I need a store".
---

## What It Does

Creates a Pinia store using setup store syntax (not options API), with:
- Typed reactive state via `ref` / `reactive`
- Computed getters via `computed`
- Async actions as plain async functions

---

## When to Use It

- Adding client-side state that needs to be shared across multiple components
- The project uses Nuxt 3 with Pinia

---

## Steps

### File Location

```
stores/<domain>.ts
```

### Example — User store

`stores/user.ts`

```typescript
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

type User = {
  id: string
  name: string
  email: string
  role: 'admin' | 'member'
}

export const useUserStore = defineStore('user', () => {
  const currentUser = ref<User | null>(null)
  const isLoading = ref(false)

  const isAuthenticated = computed(() => currentUser.value !== null)
  const isAdmin = computed(() => currentUser.value?.role === 'admin')

  async function fetchCurrentUser() {
    isLoading.value = true
    try {
      currentUser.value = await $fetch<User>('/api/users/me')
    } finally {
      isLoading.value = false
    }
  }

  function logout() {
    currentUser.value = null
  }

  return {
    currentUser,
    isLoading,
    isAuthenticated,
    isAdmin,
    fetchCurrentUser,
    logout,
  }
})
```

### Example — Cart store with collection state

`stores/cart.ts`

```typescript
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

type CartItem = {
  productId: string
  name: string
  quantity: number
  unitPrice: number
}

export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])

  const totalItems = computed(() =>
    items.value.reduce((sum, item) => sum + item.quantity, 0)
  )

  const totalPrice = computed(() =>
    items.value.reduce((sum, item) => sum + item.quantity * item.unitPrice, 0)
  )

  function addItem(item: CartItem) {
    const existing = items.value.find((i) => i.productId === item.productId)
    if (existing) {
      existing.quantity += item.quantity
      return
    }
    items.value.push(item)
  }

  function removeItem(productId: string) {
    items.value = items.value.filter((i) => i.productId !== productId)
  }

  function clear() {
    items.value = []
  }

  return {
    items,
    totalItems,
    totalPrice,
    addItem,
    removeItem,
    clear,
  }
})
```

### Using the store in a component

```vue
<script setup lang="ts">
const userStore = useUserStore()

onMounted(() => {
  userStore.fetchCurrentUser()
})
</script>

<template>
  <div v-if="userStore.isAuthenticated">
    Welcome, {{ userStore.currentUser?.name }}
  </div>
</template>
```

---

## What to Avoid

- Using options API syntax (`state`, `getters`, `actions` as object keys) — always use setup syntax
- Storing derived values as state instead of `computed`
- Exposing internal implementation details — only return what components need
- Calling `$fetch` or doing async work directly in components when a store action fits better
- Using `any` for state types
