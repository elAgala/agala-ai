---
name: gin-endpoint
description: >
  Add a complete endpoint to a Go + Gin backend following handler -> service ->
  repository architecture, with domain interfaces and typed errors. Triggered by
  phrases like "add a Go endpoint", "create a Gin route", "add a backend endpoint",
  "create a handler in Go", "add a REST endpoint to the Go backend".
---

## What It Does

Implements a full endpoint across all layers:
- Handler: binds and validates input, calls service, writes response
- Service: orchestrates business logic, calls repository via interface
- Repository: executes the database query
- Domain: defines the interface and typed errors
- Router: registers the route

---

## When to Use It

- Adding a new endpoint to the Go + Gin backend
- The project follows handler → service → repository architecture

---

## Project Structure

```
backend/
  cmd/main.go
  config/config.go
  internal/
    domain/
      order.go          <- types, interfaces, errors
    handler/
      order_handler.go
    service/
      order_service.go
    repository/
      order_repository.go
  router/
    router.go
```

---

## Steps

### 1. Domain layer

`internal/domain/order.go`

```go
package domain

import "errors"

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

`internal/repository/order_repository.go`

```go
package repository

import (
	"context"
	"database/sql"
	"errors"

	"github.com/yourorg/yourapp/internal/domain"
)

type orderRepository struct {
	db *sql.DB
}

func NewOrderRepository(db *sql.DB) domain.OrderRepository {
	return &orderRepository{db: db}
}

func (r *orderRepository) Create(ctx context.Context, input domain.CreateOrderInput) (*domain.Order, error) {
	row := r.db.QueryRowContext(ctx,
		`INSERT INTO orders (user_id, product_id, quantity, status)
		 VALUES ($1, $2, $3, 'pending')
		 RETURNING id, user_id, product_id, quantity, status, created_at`,
		input.UserID, input.ProductID, input.Quantity,
	)

	var order domain.Order
	if err := row.Scan(&order.ID, &order.UserID, &order.ProductID, &order.Quantity, &order.Status, &order.CreatedAt); err != nil {
		return nil, err
	}

	return &order, nil
}

func (r *orderRepository) FindByID(ctx context.Context, id string) (*domain.Order, error) {
	row := r.db.QueryRowContext(ctx,
		`SELECT id, user_id, product_id, quantity, status, created_at FROM orders WHERE id = $1`, id,
	)

	var order domain.Order
	if err := row.Scan(&order.ID, &order.UserID, &order.ProductID, &order.Quantity, &order.Status, &order.CreatedAt); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, domain.ErrOrderNotFound
		}
		return nil, err
	}

	return &order, nil
}
```

### 3. Service

`internal/service/order_service.go`

```go
package service

import (
	"context"

	"github.com/yourorg/yourapp/internal/domain"
)

type orderService struct {
	repo domain.OrderRepository
}

func NewOrderService(repo domain.OrderRepository) domain.OrderService {
	return &orderService{repo: repo}
}

func (s *orderService) CreateOrder(ctx context.Context, input domain.CreateOrderInput) (*domain.Order, error) {
	if input.Quantity < 1 {
		return nil, domain.ErrInvalidInput
	}

	return s.repo.Create(ctx, input)
}

func (s *orderService) GetOrder(ctx context.Context, id string) (*domain.Order, error) {
	return s.repo.FindByID(ctx, id)
}
```

### 4. Handler

`internal/handler/order_handler.go`

```go
package handler

import (
	"errors"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/yourorg/yourapp/internal/domain"
)

type OrderHandler struct {
	service domain.OrderService
}

func NewOrderHandler(service domain.OrderService) *OrderHandler {
	return &OrderHandler{service: service}
}

type createOrderRequest struct {
	ProductID string `json:"productId" binding:"required,uuid"`
	Quantity  int    `json:"quantity" binding:"required,min=1"`
}

type orderResponse struct {
	ID        string `json:"id"`
	ProductID string `json:"productId"`
	Quantity  int    `json:"quantity"`
	Status    string `json:"status"`
	CreatedAt string `json:"createdAt"`
}

func (h *OrderHandler) CreateOrder(c *gin.Context) {
	var req createOrderRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	userID := c.GetString("userID") // set by auth middleware

	order, err := h.service.CreateOrder(c.Request.Context(), domain.CreateOrderInput{
		UserID:    userID,
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
		ID:        order.ID,
		ProductID: order.ProductID,
		Quantity:  order.Quantity,
		Status:    order.Status,
		CreatedAt: order.CreatedAt.Format(time.RFC3339),
	})
}

func (h *OrderHandler) GetOrder(c *gin.Context) {
	id := c.Param("id")

	order, err := h.service.GetOrder(c.Request.Context(), id)
	if err != nil {
		if errors.Is(err, domain.ErrOrderNotFound) {
			c.JSON(http.StatusNotFound, gin.H{"error": "order not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
		return
	}

	c.JSON(http.StatusOK, orderResponse{
		ID:        order.ID,
		ProductID: order.ProductID,
		Quantity:  order.Quantity,
		Status:    order.Status,
		CreatedAt: order.CreatedAt.Format(time.RFC3339),
	})
}
```

### 5. Register the route

`router/router.go`

```go
orderRepo := repository.NewOrderRepository(db)
orderSvc := service.NewOrderService(orderRepo)
orderHandler := handler.NewOrderHandler(orderSvc)

api := r.Group("/api/v1")
{
	orders := api.Group("/orders")
	orders.Use(authMiddleware)
	{
		orders.POST("", orderHandler.CreateOrder)
		orders.GET("/:id", orderHandler.GetOrder)
	}
}
```

---

## What to Avoid

- Business logic in handlers — handlers only bind input, call service, write response
- Database queries in services — queries belong in the repository
- Returning raw `error.Error()` strings from internal errors to the client
- Skipping the domain interface — always depend on the interface, not the concrete type
- Using `_` to discard errors
