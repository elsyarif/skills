---
name: architecture-ddd
description: Generic DDD & Clean Architecture guide (Eric Evans tactical patterns — Entity, Value Object, Aggregate Root, Domain Event, Repository, Feature Flags, Error Taxonomy, Testing Standards). Use when adding external systems (infrastructure/external/<system_name>), database repositories, message brokers (RMQ/Kafka), usecases, domain services, aggregates, feature flags, or applying the Cache Decorator Pattern across any project.
---

# Architecture DDD (Domain-Driven Design) — Universal Standard

## Overview
This skill is a **generic, project-agnostic architecture contract** for Clean/Hexagonal Architecture (Ports & Adapters), enriched with **Eric Evans' tactical DDD patterns** (Entity, Value Object, Aggregate, Domain Event, Repository). Replace placeholders like `<system_name>`, `<feature>`, `<entity>`, `<Action>` with your actual domain names.

---

## 1. Universal Layer Mapping & Import Flow (Top-Down Only)

```
delivery/{grpc, http, rmqp, kafka}                   (Transport Adapters)
        ↓
application/usecase                                  (Workflow Orchestration / Application Service)
        ↓
domain/service                                       (Pure Business Rules — cross-aggregate logic)
        ↓
domain/{aggregate, entity, valueobject, event, enum, exceptions, gateway}
        ↑ implemented by
infrastructure/external/<system_name>, infrastructure/persistence/<db>, infrastructure/gateway/   (Adapters)
```

**Golden Rules:**
- `domain/entity` → Lowest layer. Zero imports from this repo.
- `domain/valueobject` → Immutable value structs. Zero repo imports (can import `domain/entity` or `domain/exceptions` errors only, or stdlib).
- `domain/enum` → Type-safe constants (feature flags, statuses). Zero repo imports.
- `domain/exceptions` → Canonical domain error constants. Stdlib only (`errors`).
- `domain/aggregate` → Multi-entity transactional units. Imports only `domain/entity`, `domain/valueobject`, and `domain/event`.
- `domain/event` → Business event payload structs. Imports `domain/entity` and `domain/valueobject` (past-tense naming).
- `domain/gateway` → Interface ports only. Imports `domain/entity`, `domain/valueobject`, `domain/enum`, and `domain/event` (for publisher ports).
- `domain/service` → Pure rules. Imports only the domain layers above. NEVER import `os`, `viper`, `net/http`, `database/sql`, `pkg/config`.
- `application/usecase` → Orchestrator. Imports `domain/*`, `pkg/logger`, and `application/service` adapters. NEVER import external DTOs, delivery packages, or `pkg/config` directly (use `application/service/ConfigAdapter`).
- `application/service` → (Optional) Adapters only: transport claims → domain entity, config → domain value object, cross-usecase helpers. NO business rules, NO DB/API I/O, NO crypto libs.
- `infrastructure/external` → Outbound adapters (HTTP/gRPC surrounding systems). Implements `domain/gateway` ports. NEVER import `application` or `delivery`.
- `infrastructure/persistence` → DB adapters. Implements `domain/gateway` repository ports. NEVER import `application` or `delivery`.
- `infrastructure/gateway` → Surrounding tool adapters (e.g. `growthbook.go` feature flags). Implements `domain/gateway` ports.
- `infrastructure/security` → (Optional) Crypto/token adapters implementing `domain/gateway` ports (`TokenSigner`, `Hasher`). Wraps low-level libs (jwt, bcrypt, aes); usecases depend on the port, never the lib.
- `infrastructure/**` (any other) → Same rule: implements a domain port, NEVER imports `application` or `delivery`.
- `internal/**` → (Optional) Project-private legacy utilities, response mappers (`internal/helper/`), or aliases (`internal/exception/`). New code prefers `infrastructure/` and `domain/exceptions/`.
- `delivery/**` → Inbound entry points. Parse transport payload → call `usecase.Execute()` → return transport response.
- Reverse direction = cyclic import = FORBIDDEN.

---

## 2. Standard Folder Structure (This Repo)

Full annotated structure: read `references/go-ddd-structure.md` (layer placement rules + import directions).
Raw scaffold tree for copy-paste: `templates/domain-tree.txt`.

Top-level layout: `api/` (exposed protos) · `protoc/` (consumed vendor protos) · `domain/` (entity, valueobject, aggregate, event, enum, exceptions, gateway, service) · `application/` (usecase, service) · `delivery/` (grpc, http, rmqp, kafka) · `infrastructure/` (external, persistence, gateway, security) · `pkg/` (cross-cutting) · `internal/` (optional legacy utilities) · `cmd/` (apps DI, server main).

---

## 3. Eric Evans Tactical Patterns (domain/ Sub-Folders)

### A. `domain/valueobject/` — Value Objects
Immutable objects with no identity, defined by their **attributes**. Two value objects with the same values are interchangeable. Always validate on construction.

```go
// domain/valueobject/money.go
package valueobject

import "errors"

type Money struct {
    Amount   int64
    Currency string
}

func NewMoney(amount int64, currency string) (Money, error) {
    if amount < 0 {
        return Money{}, errors.New("money: amount cannot be negative")
    }
    if currency == "" {
        return Money{}, errors.New("money: currency is required")
    }
    return Money{Amount: amount, Currency: currency}, nil
}

func (m Money) Add(other Money) (Money, error) {
    if m.Currency != other.Currency {
        return Money{}, errors.New("money: currency mismatch")
    }
    return Money{Amount: m.Amount + other.Amount, Currency: m.Currency}, nil
}
```

**When to use:** Money, Address, Email, Phone, DateRange — anything with validation rules and no DB identity.

### B. `domain/aggregate/` — Aggregates & Aggregate Root
A cluster of entities + value objects treated as **ONE transactional unit** with invariants. Exactly **one Entity is the Aggregate Root**; the outside world may only modify the aggregate **through the root's methods**.

```go
// domain/aggregate/order_aggregate.go
package aggregate

import (
    "errors"
    "<module>/domain/entity"
    "<module>/domain/valueobject"
)

type OrderAggregate struct {
    Order   entity.Order          // Aggregate ROOT
    Items   []entity.OrderItem    // Child entity
    Total   valueobject.Money     // Value object
}

// Constructor enforces invariants at creation
func NewOrder(order entity.Order, items []entity.OrderItem) (*OrderAggregate, error) {
    if len(items) == 0 {
        return nil, errors.New("order aggregate: cannot create empty order")
    }
    return &OrderAggregate{Order: order, Items: items}, nil
}

// Business transition guarded by invariants — only via root
func (a *OrderAggregate) Pay(paid valueobject.Money) error {
    if a.Order.Status != "PENDING" {
        return errors.New("order aggregate: only pending orders can be paid")
    }
    if paid.Amount < a.Total.Amount {
        return errors.New("order aggregate: underpaid")
    }
    a.Order.Status = "PAID"
    return nil
}

func (a *OrderAggregate) Cancel(reason string) error {
    if a.Order.Status == "SHIPPED" {
        return errors.New("order aggregate: shipped order cannot be cancelled")
    }
    a.Order.Status = "CANCELLED"
    return nil
}
```

**When to use:** Only when multiple entities MUST change atomically with strict invariants. For simple 1-entity data, plain `domain/entity` is enough (YAGNI).

### C. `domain/event/` — Domain Events
Immutable records of **something important that happened in the domain**, named in past tense (`OrderCompleted`, `PaymentFailed`). Payload only — the delivery mechanism lives in `domain/gateway` (`EventPublisher`) and `infrastructure/external/{rmqp,kafka}`.

```go
// domain/event/order_completed.go
package event

import "time"

type OrderCompleted struct {
    OrderID     string    `json:"order_id"`
    CustomerID  string    `json:"customer_id"`
    TotalAmount int64     `json:"total_amount"`
    OccurredAt  time.Time `json:"occurred_at"`
}
```

```go
// domain/event/order_cancelled.go
package event

import "time"

type OrderCancelled struct {
    OrderID    string    `json:"order_id"`
    Reason     string    `json:"reason"`
    OccurredAt time.Time `json:"occurred_at"`
}
```

**Division of labor:**

| Layer                                  | Responsibility                                        |
|----------------------------------------|-------------------------------------------------------|
| `domain/event/`                        | Event **payload struct** (data only, past-tense name) |
| `domain/gateway/event.go`              | **Port** `EventPublisher.Publish(ctx, topic, event)`  |
| `infrastructure/external/{rmqp,kafka}` | **Transport** (JSON marshal, broker protocol)         |
| `application/usecase`                  | **Decides when** to publish (after DB write succeeds) |

### D. `domain/repository/` vs `domain/gateway/` — Naming Conventions
Two accepted mazhab. Pick ONE per project and stay consistent:

| Mazhab A (this repo's default)                                                   | Mazhab B (strict Eric Evans)                                                                                     |
|----------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| ALL ports in `domain/gateway/`: `co.go`, `repository.go`, `event.go`, `cache.go` | Split by kind: `domain/repository/` (DB persistence interfaces) + `domain/gateway/` (surrounding API interfaces) |
| Simpler, fewer folders                                                           | More explicit separation of persistence vs external                                                              |

```go
// Mazhab B example: domain/repository/order_repository.go
package repository

type OrderRepository interface {
    Save(ctx context.Context, aggregate *aggregate.OrderAggregate) error
    FindByID(ctx context.Context, id string) (*aggregate.OrderAggregate, error)
}
```

### E. `domain/entity` vs `domain/valueobject` vs `domain/aggregate` — Quick Decision Table
| Question                                    | Answer                                      |
|---------------------------------------------|---------------------------------------------|
| Does it have a unique identity + lifecycle? | → `domain/entity`                           |
| No identity, defined by values, immutable?  | → `domain/valueobject`                      |
| Multi-entity group with atomic invariants?  | → `domain/aggregate` (1 root entity inside) |
| "Something happened" record?                | → `domain/event`                            |
| Pure rule crossing 2+ aggregates?           | → `domain/service`                          |

---

## 4. Standard Pattern: External Surrounding & Internal Systems (`infrastructure/external/<system_name>/`)

Every external integration (HTTP REST surrounding system or internal gRPC microservice) MUST follow this file structure inside `infrastructure/external/<system_name>/`:

```
infrastructure/external/<system_name>/
├── client.go          # Struct, Constructor, compile check: var _ gateway.<System>Gateway = (*Client)(nil)
├── request.go         # Transport logic: auth headers, retry, structured log builder, status envelope extraction
├── dto.go             # 100% mirror of external JSON/proto contract (NEVER leaks outside this package)
├── mapper.go          # Anti-Corruption Layer: DTO -> domain entity mapper
├── <action>.go        # 1 method per endpoint (separate file if the file gets large)
└── cached_client.go   # (Optional) Cache Decorator wrapping client.go
```

### Generic Client Template:
```go
// infrastructure/external/<system_name>/client.go
package <system_name>

import (
    "context"
    "<module>/domain/entity"
    "<module>/domain/gateway"
    "<module>/pkg/config"
    "github.com/go-resty/resty/v2"
)

var _ gateway.<System>Gateway = (*Client)(nil)

type Client struct {
    cfg    *config.Config
    client *resty.Client
}

func NewClient(cfg *config.Config, client *resty.Client) *Client {
    return &Client{cfg: cfg, client: client}
}
```

### Rules:
1. Base URL / endpoint paths live in config (`cfg.Service.*`), never hardcoded in the client.
2. Envelope errors from upstream become typed errors (`Err<System>API{StatusCode, StatusDesc}`) wrapped with `%w`.
3. Business status validation (e.g. `order.Status == "COMPLETED"`) belongs in usecase/domain, NOT in the adapter.
4. Polymorphic POST payloads: one master DTO with `omitempty` + pointer fields (`*bool`, `*int`).

---

## 5. Generic Cache Decorator Pattern (`infrastructure/external/<system_name>/cached_client.go`)

When caching an external call or DB query, **DO NOT put cache logic in usecases**. Wrap the gateway adapter with a Decorator in `infrastructure/`. The usecase calls the plain port interface without knowing Redis exists.

```
application/usecase
       ↓ calls <System>Gateway.<GetMethod>()
gateway.<System>Gateway (Interface — unchanged!)
       ↓ implemented by
infrastructure/external/<system_name>/cached_client.go (Decorator)
       ├── 1. Check CacheGateway first
       ├── 2. Hit  → return cached entity (skip downstream network call!)
       └── 3. Miss → call real <system_name>.Client → store to CacheGateway → return
```

### Generic Decorator Template:
```go
// infrastructure/external/<system_name>/cached_client.go
package <system_name>

import (
    "context"
    "fmt"
    "time"
    "<module>/domain/entity"
    "<module>/domain/gateway"
)

var _ gateway.<System>Gateway = (*CachedClient)(nil)

type CachedClient struct {
    realClient gateway.<System>Gateway
    cache      gateway.CacheGateway
    ttl        time.Duration
}

func NewCachedClient(realClient gateway.<System>Gateway, cache gateway.CacheGateway, ttl time.Duration) *CachedClient {
    return &CachedClient{realClient: realClient, cache: cache, ttl: ttl}
}

// Decorate READ methods only:
func (c *CachedClient) Get<Entity>Detail(ctx context.Context, id string) (*entity.<Entity>, error) {
    cacheKey := fmt.Sprintf("<system_name>:<entity>:%s", id)

    if c.cache != nil {
        var cached entity.<Entity>
        if err := c.cache.Get(ctx, cacheKey, &cached); err == nil {
            return &cached, nil // Cache HIT — usecase never knows
        }
    }

    data, err := c.realClient.Get<Entity>Detail(ctx, id)
    if err != nil {
        return nil, err
    }

    if c.cache != nil && data != nil {
        _ = c.cache.Set(ctx, cacheKey, data, c.ttl)
    }

    return data, nil
}

// WRITE methods pass through (never cache writes):
func (c *CachedClient) Submit<Entity>(ctx context.Context, param *entity.<Param>) (*entity.<Result>, error) {
    return c.realClient.Submit<Entity>(ctx, param)
}
```

### Decorator Rules:
1. The decorator lives NEXT TO the real adapter, NOT in the usecase.
2. It implements the SAME port interface (`var _ gateway.<System>Gateway = (*CachedClient)(nil)`).
3. Usecase `Deps` must NOT have a `Cache` field when caching is handled by the decorator.
4. Wrap at wiring time in `cmd/apps/`:
```go
rawClient := <system_name>.NewClient(cfg, restyClient)
gatewayInstance := <system_name>.NewCachedClient(rawClient, cacheGateway, 5*time.Minute)
// To disable caching: inject rawClient directly — zero usecase changes.
```
5. Only decorate READ methods (Get/Query). Never cache writes (Submit/Create/Update).

---

## 6. Generic Single-Action Usecase (Command Handler)

Every usecase follows: **1 file = 1 struct = 1 `Deps` struct = 1 `Execute()` method**, named `<feature>_<action>_usecase.go`.

```go
// application/usecase/<feature>_<action>_usecase.go
package usecase

import (
    "context"
    "<module>/domain/entity"
    "<module>/domain/gateway"
    "<module>/pkg/logger"
)

type <Feature><Action>Deps struct {
    Log             logger.Logger
    <System>Gateway gateway.<System>Gateway
    // OrderRepo gateway.<Feature>Repository
}

type <Feature><Action>Usecase struct {
    deps <Feature><Action>Deps
}

func New<Feature><Action>Usecase(deps <Feature><Action>Deps) *<Feature><Action>Usecase {
    return &<Feature><Action>Usecase{deps: deps}
}

func (u *<Feature><Action>Usecase) Execute(ctx context.Context, param *entity.<Param>) (*entity.<Result>, error) {
    u.deps.Log.Info("executing usecase", logger.Fields{
        "usecase": "<Feature><Action>Usecase",
        "param":   param,
    })

    // 1. Validation
    if err := validateParam(param); err != nil {
        u.deps.Log.Error("validation failed", logger.Fields{"error": err.Error()})
        return nil, err
    }

    // 2. Gateway Orchestration
    res, err := u.deps.<System>Gateway.<Action>(ctx, param)
    if err != nil {
        u.deps.Log.Error("gateway call failed", logger.Fields{"error": err.Error()})
        return nil, err
    }

    u.deps.Log.Info("usecase succeeded", logger.Fields{"result": res})
    return res, nil
}
```

---

## 7. Generic Event-Driven Messaging (RabbitMQ / Kafka)

1. **Port Interface (`domain/gateway/event.go`)**:
   ```go
   type EventPublisher interface {
       Publish(ctx context.Context, topicOrRoutingKey string, event any) error
   }
   ```
2. **Event payloads** live in `domain/event/` (see Section 3C) — never inline anonymous structs in usecases.
3. **Publisher Adapters**: `infrastructure/external/rmqp` & `infrastructure/external/kafka` (shared clients in `pkg/rmqp`, `pkg/kafka`).
4. **Consumer Delivery**: `delivery/{rmqp,kafka}/consumer/<feature>_consumer.go`:
   Decode payload → inject `support_id`/`co_order_id` into ctx → call `usecase.Execute()` → Ack/Commit on success, Nack/DLQ (or skip offset commit) on failure.
5. Publish failure after a successful DB write must NOT fail the usecase — log it instead.

---

## 8. Wiring (`cmd/apps/`) — Zero Merge Conflict

- `app.go` holds shared singletons (`sharedDeps`: config, logger, resty, gateways, db, publisher, cache, kafka client). LOCKED file: new features add 1 line per protocol.
- Each feature owns `wire_<feature>.go` with `wire<Feature>GRpc`, `wire<Feature>HTTP`, `wire<Feature>Consumer` / `wire<Feature>KafkaConsumer`.

---

## 9. Feature Flag Pattern (GrowthBook / Unleash / LaunchDarkly)

Feature flags control **release workflows**, NOT business logic. Therefore, evaluation is strictly constrained by layer:

| Layer                                  | Allowed?          | Reason                                                                   |
|----------------------------------------|-------------------|--------------------------------------------------------------------------|
| `delivery/`                            | Rare              | Only when routing to different handler/protocol versions.                |
| `application/usecase/`                 | **YES (Primary)** | Usecase orchestrates flow: calls old vs new branch based on flag.        |
| `domain/` (Entity, Aggregate, Service) | **FORBIDDEN**     | Domain is pure rules; it must never know release tools or rollout state. |
| `infrastructure/` (Adapters)           | No                | Infrastructure executes operations, does not decide business flow.       |

### Structure & Flow:

```
domain/enum/feature_flag.go        (Type-safe flag names)
        ↓
domain/gateway/feature_flag_gateway.go (Outbound port interface)
        ↓ implemented by
infrastructure/gateway/growthbook.go (primary; legacy projects may use internal/ instead)
        ↓ consumed by
application/usecase/               (Evaluates flag via port, branches flow)
```

### 1. Port & Enum Definition:

```go
// domain/enum/feature_flag.go
package enum

type FeatureFlag string

func (f FeatureFlag) ToString() string { return string(f) }

const (
    FeatureNewCheckoutFlow FeatureFlag = "FF_NEW_CHECKOUT_FLOW"
    FeatureWhitelistOnly   FeatureFlag = "FF_CHECKOUT_WHITELIST_ONLY"
)
```

```go
// domain/gateway/feature_flag_gateway.go
package gateway

import (
    "context"
    "<module>/domain/enum"
)

type FeatureFlagGateway interface {
    IsEnabled(ctx context.Context, feature enum.FeatureFlag) bool
    IsWhitelisted(ctx context.Context, path string, identifier string) bool
}
```

### 2. Usage in Usecase:

```go
// application/usecase/<feature>_<action>_usecase.go
// Path whitelist MUST come from config via ConfigAdapter, NEVER hardcoded string in usecase!
cfg := u.deps.ConfigAdapter.ToAppConfig()

if u.deps.FeatureFlag.IsEnabled(ctx, enum.FeatureNewCheckoutFlow) {
    result.UseNewFlow = true

    if u.deps.FeatureFlag.IsEnabled(ctx, enum.FeatureWhitelistOnly) {
        if u.deps.FeatureFlag.IsWhitelisted(ctx, cfg.WhitelistFilePath, req.UserID) {
            result.UseNewFlow = false
        }
    }
}
```

### Golden Rules:
1. NEVER import GrowthBook / SDK directly in usecase — always depend on `domain/gateway.FeatureFlagGateway`.
2. File paths (whitelists, configs) MUST pass through `ConfigAdapter` into a value object, NEVER hardcoded literals in usecase.
3. Every flag branch MUST have unit test coverage: flag ON, flag OFF, and whitelist match / non-match.

---

## 10. Error Taxonomy & HTTP Mapping

Errors must be **strongly typed at domain boundaries** so callers (handlers, other services) can use `errors.Is()` reliably without string matching.

### Layer Responsibility:

```
domain/exceptions/          # Canonical error definitions (errors.New constants)
       ↓ aliased by
internal/exception/         # Backward-compatibility alias for older non-DDD code
       ↓ returned by
application/usecase/        # Returns canonical domain errors (NEVER inline errors.New)
       ↓ mapped by
internal/helper/response.go # MapHttpStatusCode(err) translates domain error -> (statusCode, httpCode)
       ↓ consumed by
delivery/http/handler/      # AbortWithStatusJSON using mapped codes; branches with errors.Is
```

### 1. Domain Error Definition:

```go
// domain/exceptions/errors.go
package exceptions

import "errors"

var (
    ErrNotFound               = errors.New("error - data not found")
    ErrForbiddenAction        = errors.New("error - forbidden action")
    ErrInvalidPaymentMethodId = errors.New("Invalid or unsupported payment_method_id")
    ErrNoRows                 = errors.New("error - no record found")
    ErrTransactionIDNotFound  = errors.New("Transaction ID not Found")
)
```

### 2. Usecase Returns Canonical Error (No Inline errors.New):

```go
// BAD: handler cannot errors.Is, test must string-match, log vs return drift
log.Error(event, exceptions.ErrInvalidPaymentMethodId, 400)
return nil, errors.New("invalid payment method unsupported")

// GOOD: single source of truth
log.Error(event, exceptions.ErrInvalidPaymentMethodId, 400)
return nil, exceptions.ErrInvalidPaymentMethodId
```

### 3. Handler Mapping & Branching:

```go
// delivery/http/handler/<feature>_handler.go
result, err := h.usecase.Execute(c.Request.Context(), req)
if err != nil {
    statusCode, httpCode := helper.MapHttpStatusCode(err)
    if errors.Is(err, exceptions.ErrForbiddenAction) {
        c.AbortWithStatusJSON(http.StatusBadRequest, helper.HttpErrorResponse(statusCode, err, supportID, nil))
        return
    }
    c.AbortWithStatusJSON(httpCode, helper.HttpErrorResponse(statusCode, err, supportID, nil))
    return
}
```

### 4. Mapping Rule:
- Business errors (`ErrInvalidPaymentMethodId`, `ErrBadRequest`) map to `statusCode = 400`.
- System / unexpected errors map to `statusCode = 500`.
- If repo uses envelope pattern (HTTP 200 with body status 4xx), let `MapHttpStatusCode` handle the split — do NOT hardcode it in handler.

---

## 11. Testing Standards (Table-Driven Pattern)

All usecases, handlers, and gateways MUST follow the **Table-Driven Test** pattern with `testify` and `gomock`. No sprawling multi-function test files.

### 1. Usecase Table-Driven Template:

```go
// application/usecase/<feature>_<action>_usecase_test.go
package usecase

import (
    "context"
    "testing"

    "<module>/pkg/logger"

    "github.com/golang/mock/gomock"
    "github.com/stretchr/testify/assert"
)

// Minimal stub logger for zero-allocation test runs
type stubLogger struct{}
func (s *stubLogger) Info(msg string, fields logger.Fields)   {}
func (s *stubLogger) Error(msg string, fields logger.Fields)  {}
func (s *stubLogger) Warn(msg string, fields logger.Fields)   {}
func (s *stubLogger) Debug(msg string, fields logger.Fields)  {}
func (s *stubLogger) Fatal(msg string, fields logger.Fields)  {}
func (s *stubLogger) Panic(msg string, fields logger.Fields)  {}

func Test<Feature><Action>Usecase_Execute(t *testing.T) {
    tests := []struct {
        name       string
        req        *entity.<Request>
        mockSetup  func(deps <Feature><Action>Deps)
        wantResult func(t *testing.T, res *entity.<Result>)
        wantErr    string // or error for errors.Is comparison
    }{
        {
            name: "happy path - success",
            req:  validRequest(),
            mockSetup: func(deps <Feature><Action>Deps) {
                deps.Gateway.(*mocks.MockGateway).EXPECT().
                    GetData(gomock.Any(), "id-123").
                    Return(expectedData, nil)
            },
            wantResult: func(t *testing.T, res *entity.<Result>) {
                assert.Equal(t, "expected", res.Field)
            },
        },
        {
            name: "gateway error - returns wrapped error",
            req:  validRequest(),
            mockSetup: func(deps <Feature><Action>Deps) {
                deps.Gateway.(*mocks.MockGateway).EXPECT().
                    GetData(gomock.Any(), "id-123").
                    Return(nil, errors.New("upstream timeout"))
            },
            wantErr: "upstream timeout",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            ctrl := gomock.NewController(t)
            defer ctrl.Finish()

            deps := <Feature><Action>Deps{
                Log:     &stubLogger{},
                Gateway: mocks.NewMockGateway(ctrl),
            }

            if tt.mockSetup != nil {
                tt.mockSetup(deps)
            }

            u := New<Feature><Action>Usecase(deps)
            result, err := u.Execute(context.Background(), tt.req)

            if tt.wantErr != "" {
                assert.Nil(t, result)
                assert.EqualError(t, err, tt.wantErr)
            } else {
                assert.NoError(t, err)
                assert.NotNil(t, result)
                if tt.wantResult != nil {
                    tt.wantResult(t, result)
                }
            }
        })
    }
}
```

### 2. Handler Table-Driven Template:

```go
// delivery/http/handler/<feature>_handler_test.go
func Test<Feature>Handler(t *testing.T) {
    gin.SetMode(gin.TestMode)

    tests := []struct {
        name       string
        env        string
        claims     *helper.Claims
        mockSetup  func(deps <Feature>Deps)
        wantStatus int
        wantBody   []string
    }{
        {
            name:       "short-circuit nft - returns mock without calling usecase",
            env:        "nft",
            claims:     validClaims(),
            wantStatus: http.StatusOK,
            wantBody:   []string{"mock response fragment"},
        },
        {
            name:       "success prod - calls usecase and returns 200",
            env:        "prod",
            claims:     validClaims(),
            mockSetup:  func(deps <Feature>Deps) { /* mock expectations */ },
            wantStatus: http.StatusOK,
            wantBody:   []string{`"status":"success"`},
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            ctrl := gomock.NewController(t)
            defer ctrl.Finish()

            // setup deps + handler...
            w := httptest.NewRecorder()
            c, _ := gin.CreateTestContext(w)
            if tt.claims != nil {
                c.Set("claims", tt.claims)
            }
            c.Request = httptest.NewRequest(http.MethodGet, "/path", nil)

            h.Handle(c)

            assert.Equal(t, tt.wantStatus, w.Code)
            for _, expected := range tt.wantBody {
                assert.Contains(t, w.Body.String(), expected)
            }
        })
    }
}
```

### Testing Rules:
1. **Controller isolation:** `ctrl := gomock.NewController(t)` MUST be inside `t.Run()` — prevents mock leakage across subtests.
2. **Context matcher:** Use `gomock.Any()` for `context.Context` parameters when OpenTelemetry / tracing wraps the context.
3. **Timezone safety:** For date assertions formatted with local time, use `assert.Contains(t, res.Date, "Nov 2023")` instead of exact day strings like `"14 Nov 2023"` (UTC vs GMT+7 shifts day).
4. **Mock regeneration:** When interface changes, immediately update mock (via `mockgen` or sed) before running tests.

---

## 12. External Libraries & Interface Boundary Rule

Not all external libraries require an interface wrapper. Follow the **Pragmatic Go Boundary Rule**:

### When an Interface (Port) is MANDATORY:
Wrap behind `domain/gateway/` when the dependency is **I/O-bound, non-deterministic, stateful, or must be swapped/mocked**:

| Dependency Kind | Reason | Example Port (`domain/gateway/`) |
|---|---|---|
| **Database** (MySQL, Postgres) | No live DB in unit tests | `repository.PaymentRepository` |
| **Cache** (Redis) | Decouple usecase from cache backend | `gateway.CacheGateway` |
| **HTTP Surrounding** (CO, CRM, Elisa) | External network dependency | `repository.ElisaRepository` |
| **gRPC Client** (Link, Product) | Microservice network dependency | `gateway.ProductDetailer` |
| **Message Broker** (RabbitMQ, Kafka) | I/O streaming | `gateway.EventPublisher` |
| **Feature Flags** (GrowthBook) | Dynamic remote state | `gateway.FeatureFlagGateway` |
| **Logging** (logrus, zap, slog, logmanager) | I/O + test isolation | `logger.Logger` (lives in `pkg/logger` — cross-cutting, not domain) |
| **Crypto / Token Signing** (JWT, AES, bcrypt) | Test determinism & swappable algorithms | `gateway.TokenSigner`, `gateway.Hasher` |
| **Time / Clock** *(if business-critical)* | Test time-travel without sleeping | `gateway.Clock` |

### When to DIRECT IMPORT (No Interface Needed):
Direct import is preferred for **pure, stateless, deterministic computation**. Wrapping these is an anti-pattern (interface bloat):

| Library / Tool | Reason |
|---|---|
| **JSON / XML Serializers** (`encoding/json`, `sonic`) | Deterministic input-output transformation |
| **String / Regex** (`strings`, `regexp`) | Pure CPU computation, zero side-effects |
| **Math / Decimals** (`shopspring/decimal`) | Value calculations |
| **Standard Library** (`time`, `errors`, `fmt`, `context`) | Language primitives |

### Go Idiom: Consumer-Owned Interfaces
- Interfaces are defined **where they are consumed** (`domain/gateway/`), NOT where they are implemented (`pkg/` or `infrastructure/`).
- **Exception (cross-cutting utilities):** Interfaces for technical infrastructure shared across all layers (`logger.Logger`) live in their respective `pkg/` package (`pkg/logger/`), NOT `domain/gateway/`. Backend adapters (logrus, zap, slog, logmanager) implement it in `pkg/logger/<backend>.go`.
- Define only the methods the consumer actually calls (Interface Segregation Principle).
- Single-method interfaces use `-er` suffix (`ProductDetailer`, `Encrypter`). Multi-method use descriptive names (`LinkService`).

---

## 13. Role of `application/service/` (Optional Layer)

`application/service/` is **OPTIONAL**. In pure Eric Evans DDD, `application/usecase` IS the Application Service.

### When to use `application/service/`:
1. **Transport Adapters (Anti-Corruption mini):** Translating transport-specific structs (Gin context, claims) to domain entities without polluting the usecase (e.g. `ClaimAdapter`).
2. **Configuration Adapters:** Converting infrastructure configuration (`*config.Config`) into domain value objects (e.g. `ConfigAdapter`).
3. **Re-usable orchestration helpers** shared across multiple usecases.

### What NEVER goes into `application/service/`:
- **Business rules:** Calculations, discounts, domain transitions belong to `domain/service/` or `domain/aggregate/`.
- **Database / API calls:** I/O operations belong to `infrastructure/` (legacy projects may use `internal/repository/`).
- **External libraries / Crypto:** Belongs to `infrastructure/` implementing a `domain/gateway/` port.

---

## 14. Anti-Patterns Checklist

| Anti-Pattern | Symptom | Fix |
|---|---|---|
| **Cyclic Import** | `domain/service` imports `usecase` or `infra` | Move shared spec/result struct to `domain/entity/` |
| **DTO Leakage** | Usecase sees JSON tags or Protobuf structs | Transform DTO → entity inside `infrastructure/external/<system_name>/mapper.go` |
| **Cache in Usecase** | Usecase `Deps` has `Cache` field + if-else hit/miss | Use **Cache Decorator Pattern** in `infrastructure/external/<system_name>/` |
| **Config in Domain** | `domain/service` imports `viper` / `pkg/config` | Invert via `domain/gateway/` interface or pass as parameter |
| **Business Rule in Infra** | Adapter checks `if data.Status != "COMPLETED"` | Adapter returns data as-is; usecase/domain validates |
| **God Usecase** | 1 usecase struct with 10 methods | 1 file = 1 single-action usecase = 1 `Execute()` |
| **Anemic Aggregate** | Aggregate struct is just a bag of fields; all logic in usecase | Move invariants into aggregate root methods (`aggregate.Pay()`, `aggregate.Cancel()`) |
| **Event Payload in Usecase** | Anonymous struct inline in `Publish(ctx, topic, struct{...})` | Define payload in `domain/event/` with past-tense name |
| **Entity/VO Confusion** | Mutable struct with identity treated as value object, or VO given a DB ID | Identity + lifecycle → `entity`; immutable values → `valueobject` |
| **Feature Flag in Domain** | `domain/aggregate` or `domain/service` checks GrowthBook | Move flag evaluation to `application/usecase/` via `gateway.FeatureFlagGateway` |
| **Hardcoded File Path in Usecase** | `IsWhitelisted(ctx, "file/foo.json", id)` in usecase | Pass path from config via `ConfigAdapter` into value object |
| **Inline errors.New in Usecase** | `return nil, errors.New("...")` in usecase | Return canonical `domain/exceptions.Err*` constant so handler can `errors.Is` |
| **Leaky Mock Controller** | `ctrl := gomock.NewController(t)` outside `t.Run()` loop | Create `ctrl` INSIDE `t.Run()` with `defer ctrl.Finish()` for test isolation |
| **Exact Date Assertion** | `assert.Equal(t, "14 Nov 2023", date)` fails on CI | Assert `Contains(t, date, "Nov 2023")` or mock/freeze time |
| **Interface Bloat** | Interfaces created for `json.Marshal` or `strings` | Use direct imports for pure deterministic functions; interface only for I/O and state |
| **Crypto in Application Layer** | Direct import of `bcrypt` / `crypto/aes` in usecase or `application/service` | Define port in `domain/gateway/`, implement in `infrastructure/` or `pkg/crypto/` |
