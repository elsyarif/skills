# Go DDD Standard Structure — Reference

Annotated folder structure for the Clean/Hexagonal Architecture contract in SKILL.md.
Raw scaffold tree: `templates/domain-tree.txt`.

Replace placeholders (`<feature>`, `<system_name>`, `<entity>`, `<Action>`, `<mechanism>`) with actual domain names.

## Layer Placement Rules

| Adding... | Goes to | File naming |
|---|---|---|
| New proto contract (server) | `api/<feature>/v1/` | `<feature>.proto` + generated pb |
| Consumed vendor proto | `protoc/<service>/v1/` | compile via `protoc/gen.sh` |
| Business data struct | `domain/entity/` | `<entity>.go` |
| Immutable validated value | `domain/valueobject/` | `<value>.go` |
| Multi-entity invariant unit | `domain/aggregate/` | `<entity>_aggregate.go` |
| Business event payload | `domain/event/` | `<entity>_<ActionPastTense>.go` |
| Domain enum constants | `domain/enum/` | `<concept>.go` (e.g. `feature_flag.go`) |
| Domain error definitions | `domain/exceptions/` | `errors.go` |
| Port interface | `domain/gateway/` | `<system>.go`, `repository.go`, `cache.go`, `event.go`, `feature_flag_gateway.go` |
| Cross-entity pure rule | `domain/service/` | `<topic>_service.go` |
| Single-action workflow | `application/usecase/` | `<feature>_<action>_usecase.go` (1 file = 1 usecase = 1 `Execute`) |
| Transport / config adapter | `application/service/` | `<transport>_adapter.go`, `config_adapter.go` |
| Outbound HTTP/gRPC client | `infrastructure/external/<system_name>/` | `client.go`, `request.go`, `dto.go`, `mapper.go`, `cached_client.go` |
| DB adapter | `infrastructure/persistence/<db>/` | implements `domain/gateway` repository port |
| Tool adapter (flags, metrics) | `infrastructure/gateway/` | `growthbook.go` (implements `domain/gateway` ports) |
| Crypto/token adapter | `infrastructure/security/<mechanism>/` | implements TokenSigner/Hasher/Encrypter ports |
| Handler / route / consumer | `delivery/{grpc,http,rmqp,kafka}/` | `<feature>_handler.go`, `<feature>_route.go`, `<feature>_consumer.go` |
| DI wiring | `cmd/apps/` | `app.go` (shared) + `wire_<feature>.go` |
| (Optional) Legacy utilities | `internal/{exception,helper}/` | legacy compatibility; primary layer is `infrastructure/` + `domain/exceptions/` |

## Import Direction (Top-Down Only)

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

- `domain/entity` → lowest layer, zero imports from this repo.
- `domain/valueobject` → immutable; only `domain/entity` / `domain/exceptions` errors, or stdlib.
- `domain/enum` → type-safe constants, zero repo imports.
- `domain/exceptions` → canonical error constants, stdlib only.
- `domain/aggregate` → imports only `domain/entity`, `domain/valueobject`, `domain/event`.
- `domain/event` → imports only `domain/entity`, `domain/valueobject`.
- `domain/gateway` → interface ports only; imports `domain/entity`, `domain/valueobject`, `domain/enum`, `domain/event` (publisher ports).
- `domain/service` → pure rules, imports domain layers above, NEVER infrastructure. NEVER import `os`, `viper`, `net/http`, `database/sql`, `pkg/config`.
- `application/usecase` → imports `domain/*`, `pkg/logger`, `application/service` adapters; NEVER delivery, NEVER `pkg/config` directly (use `ConfigAdapter`), NEVER infrastructure directly (only via gateway ports).
- `infrastructure/*` → implements a domain port, NEVER imported by domain/application.
- `delivery/*` → parses transport → calls usecase → returns transport. No business logic.
