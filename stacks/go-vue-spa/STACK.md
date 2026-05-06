# Stack: Go + Vue SPA (Monorepo)

## Detection

`/backend` and `/frontend` directories exist at the project root.

---

## Key paths — Backend

| Path | Purpose |
|---|---|
| `backend/internal/domain/` | Types, interfaces, typed errors |
| `backend/internal/handler/` | Gin HTTP handlers |
| `backend/internal/service/` | Business logic |
| `backend/internal/repository/` | Database queries |
| `backend/router/` | Route registration |
| `backend/config/config.go` | Typed config struct + godotenv |
| `backend/cmd/main.go` | Entry point |

## Key paths — Frontend

| Path | Purpose |
|---|---|
| `frontend/src/api/` | HTTP repository (all fetch calls) |
| `frontend/src/queries/` | TanStack Query factories |
| `frontend/src/components/` | Vue components |
| `frontend/src/views/` | Page-level components |
| `frontend/src/stores/` | Pinia stores |
| `frontend/src/router/` | Vue Router config |

---

## Backend architecture: handler → service → repository

Dependency direction: handler depends on service interface; service depends on repository interface. Concrete types are wired in main/router — never inside the layers themselves.

### 1. Domain layer

Defines types, interfaces, and typed errors. No logic here.

```go
// backend/internal/domain/order.go
package domain

import (
	"context"
	"errors"
	"time"
)

type Order struct {
	ID        string
	UserID    string
	ProductID string
	Quantity  int
	Status    string
	CreatedAt time.Time
}

type CreateOrderInput struct {
	UserID    string
	ProductID string
	Quantity  int
}

type OrderRepository interface {
	Create(ctx context.Context, input CreateOrderInput) (*Order, error)
	FindByID(ctx context.Context, id string) (*Order, error)
}

type OrderService interface {
	CreateOrder(ctx context.Context, input CreateOrderInput) (*Order, error)
	GetOrder(ctx context.Context, id string) (*Order, error)
}

var (
	ErrOrderNotFound = errors.New("order not found")
	ErrInvalidInput  = errors.New("invalid input")
)
```

### 2. Repository

Implements the domain interface. All DB queries live here.

```go
// backend/internal/repository/order_repository.go
package repository

import (
	"context"
	"database/sql"
	"errors"

	"github.com/yourorg/app/internal/domain"
)

type orderRepository struct{ db *sql.DB }

func NewOrderRepository(db *sql.DB) domain.OrderRepository {
	return &orderRepository{db: db}
}

func (r *orderRepository) FindByID(ctx context.Context, id string) (*domain.Order, error) {
	row := r.db.QueryRowContext(ctx,
		`SELECT id, user_id, product_id, quantity, status, created_at FROM orders WHERE id = $1`, id,
	)
	var o domain.Order
	if err := row.Scan(&o.ID, &o.UserID, &o.ProductID, &o.Quantity, &o.Status, &o.CreatedAt); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, domain.ErrOrderNotFound
		}
		return nil, err
	}
	return &o, nil
}
```

Rules:
- Constructor always returns the domain interface, not the concrete type
- Map `sql.ErrNoRows` → the relevant domain error
- Never skip error handling — no `_` discarding errors from fallible calls

### 3. Service

Orchestrates business logic. Calls the repository interface.

```go
// backend/internal/service/order_service.go
package service

import (
	"context"

	"github.com/yourorg/app/internal/domain"
)

type orderService struct{ repo domain.OrderRepository }

func NewOrderService(repo domain.OrderRepository) domain.OrderService {
	return &orderService{repo: repo}
}

func (s *orderService) CreateOrder(ctx context.Context, input domain.CreateOrderInput) (*domain.Order, error) {
	if input.Quantity < 1 {
		return nil, domain.ErrInvalidInput
	}
	return s.repo.Create(ctx, input)
}
```

Rules:
- No DB calls in the service — use the repository
- No HTTP concepts (status codes, JSON) — that belongs in the handler
- Return domain errors, not generic `errors.New` strings

### 4. Handler

Binds input, calls service, maps errors to HTTP responses.

```go
// backend/internal/handler/order_handler.go
package handler

import (
	"errors"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/yourorg/app/internal/domain"
)

type OrderHandler struct{ service domain.OrderService }

func NewOrderHandler(service domain.OrderService) *OrderHandler {
	return &OrderHandler{service: service}
}

type createOrderRequest struct {
	ProductID string `json:"productId" binding:"required,uuid"`
	Quantity  int    `json:"quantity"  binding:"required,min=1"`
}

type orderResponse struct {
	ID        string `json:"id"`
	ProductID string `json:"productId"`
	Quantity  int    `json:"quantity"`
	Status    string `json:"status"`
	CreatedAt string `json:"createdAt"`
}

func (h *OrderHandler) Create(c *gin.Context) {
	var req createOrderRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	order, err := h.service.CreateOrder(c.Request.Context(), domain.CreateOrderInput{
		UserID:    c.GetString("userID"),
		ProductID: req.ProductID,
		Quantity:  req.Quantity,
	})
	if err != nil {
		if errors.Is(err, domain.ErrInvalidInput) {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
		return
	}

	c.JSON(http.StatusCreated, orderResponse{
		ID: order.ID, ProductID: order.ProductID, Quantity: order.Quantity,
		Status: order.Status, CreatedAt: order.CreatedAt.Format(time.RFC3339),
	})
}
```

Rules:
- Never leak internal error messages in 500 responses
- Map every domain error to an explicit status code
- No business logic in handlers
- Status codes: 200 / 201 / 204 / 400 / 401 / 403 / 404 / 409 / 500

### 5. Route registration

```go
// backend/router/router.go
orderRepo := repository.NewOrderRepository(db)
orderSvc  := service.NewOrderService(orderRepo)
orderH    := handler.NewOrderHandler(orderSvc)

api := r.Group("/api/v1")
orders := api.Group("/orders").Use(authMiddleware)
orders.POST("", orderH.Create)
orders.GET("/:id", orderH.GetByID)
```

### Config

```go
// backend/config/config.go
type Config struct {
	DatabaseURL string
	Port        string
	JWTSecret   string
}

func Load() (*Config, error) {
	_ = godotenv.Load()
	return &Config{
		DatabaseURL: os.Getenv("DATABASE_URL"),
		Port:        os.Getenv("PORT"),
		JWTSecret:   os.Getenv("JWT_SECRET"),
	}, nil
}
```

Never access `os.Getenv` outside the config package.

---

## Frontend architecture

### HTTP repository

All `fetch` calls live here. Components never call fetch directly.

```typescript
// frontend/src/api/orders.ts
const BASE = import.meta.env.VITE_API_URL

async function fetchJSON<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE}${path}`, {
    headers: { 'Content-Type': 'application/json' },
    ...init,
  })
  if (!res.ok) {
    const body = await res.json().catch(() => ({}))
    throw new Error(body.error ?? `HTTP ${res.status}`)
  }
  return res.json() as Promise<T>
}

export type Order = { id: string; productId: string; quantity: number; status: string; createdAt: string }
export type CreateOrderPayload = { productId: string; quantity: number }

export const ordersApi = {
  getById: (id: string) => fetchJSON<Order>(`/api/v1/orders/${id}`),
  create:  (payload: CreateOrderPayload) =>
    fetchJSON<Order>('/api/v1/orders', { method: 'POST', body: JSON.stringify(payload) }),
}
```

### TanStack Query factory

Query keys and query functions live in one place. Components import from here.

```typescript
// frontend/src/queries/orders.ts
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

### Component usage

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
  <div v-else>{{ order.status }}</div>
</template>
```

### Pinia store

```typescript
// frontend/src/stores/cart.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])
  const total = computed(() => items.value.reduce((s, i) => s + i.quantity * i.unitPrice, 0))

  function addItem(item: CartItem) {
    const existing = items.value.find((i) => i.productId === item.productId)
    if (existing) { existing.quantity += item.quantity; return }
    items.value.push(item)
  }

  return { items, total, addItem }
})
```

---

## Testing

### Go (backend)

```go
// backend/internal/service/order_service_test.go
func TestCreateOrder_ValidInput(t *testing.T) {
	repo := &mockOrderRepository{}
	svc := NewOrderService(repo)
	order, err := svc.CreateOrder(context.Background(), domain.CreateOrderInput{
		UserID: "u1", ProductID: "p1", Quantity: 2,
	})
	require.NoError(t, err)
	require.Equal(t, 2, order.Quantity)
}

func TestCreateOrder_InvalidQuantity(t *testing.T) {
	svc := NewOrderService(&mockOrderRepository{})
	_, err := svc.CreateOrder(context.Background(), domain.CreateOrderInput{Quantity: 0})
	require.ErrorIs(t, err, domain.ErrInvalidInput)
}
```

Run with: `go test ./...` from `backend/`.

### Frontend (Vitest + Vue Test Utils)

```typescript
// frontend/src/components/__tests__/OrderCard.test.ts
import { mount } from '@vue/test-utils'
import { describe, it, expect } from 'vitest'
import OrderCard from '../OrderCard.vue'

describe('OrderCard', () => {
  it('renders the order status', () => {
    const wrapper = mount(OrderCard, { props: { orderId: '1', status: 'pending' } })
    expect(wrapper.text()).toContain('pending')
  })
})
```

---

## Conventions

- No `any` without an inline comment
- No `continue` or `break` — use early returns or `filter`/`map`/`find`
- No `console.log` / `fmt.Println` in delivered code
- No hardcoded secrets or API keys
- No new dependency without explicit user approval
- No `os.Getenv` outside `backend/config/`
