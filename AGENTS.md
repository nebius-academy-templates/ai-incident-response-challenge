# Project Architecture and Development Guidelines

## Overview

This project follows **Hexagonal Architecture** (also known as Ports and Adapters).
The core principle is that business logic is completely isolated from infrastructure
concerns. The domain never knows about HTTP, MongoDB, or any external framework —
it only works with interfaces and pure Go.

## Project Structure

/cmd/api/main.go                        → Dependency injection, service wiring, server startup
/internal/                              → Business logic, owned by the domain, no external imports
/internal/order/                        → Order domain — models, services, handlers
/internal/catalog/                      → Catalog domain — models, services, handlers
/pkg/                                   → Shared infrastructure, framework agnostic
/pkg/types/order.go                     → Order domain types: Order, Item, Report, OrderStatus, NotificationChannel
/pkg/types/catalog.go                   → Catalog domain types: Product
/pkg/database/database.go               → Order and Catalog port interfaces
/pkg/database/mongo/client.go           → MongoDB client initialization and lifecycle
/pkg/database/mongo/order.go            → MongoDB adapter implementing the Order port
/pkg/database/mongo/catalog.go          → MongoDB adapter implementing the Catalog port
/pkg/database/mock/order.go             → Mock adapter implementing the Order port for testing
/pkg/database/mock/catalog.go           → Mock adapter implementing the Catalog port for testing
/pkg/server/server.go                   → Server interface
/pkg/server/gin/server.go               → Gin server implementation
/pkg/server/mock/server.go              → Mock server for testing
/pkg/telemetry/otel/context.go          → Context helpers: WithDomain, WithOperation, TraceID, KV
/pkg/telemetry/otel/logs.go             → OpenTelemetry logger provider setup
/pkg/telemetry/otel/metrics.go          → OpenTelemetry meter provider setup
/pkg/telemetry/otel/traces.go           → OpenTelemetry tracer provider setup
/pkg/telemetry/otel/setup.go            → Combined telemetry bootstrap (logs + metrics + traces)
/scripts/                               → Simulation scripts for load testing and issue reproduction
/docker/                                → Configuration files for all observability stack containers

## Layer Responsibilities

### `internal/<domain>/<domain>.go`
Domain model definitions — structs, value objects, domain errors, and the interface
that the database layer must implement. This is the heart of the domain. Imports from
`pkg/types` are allowed; no external framework imports.

### `internal/<domain>/service/`
One file per use case (`create.go`, `get.go`, `update.go`, `delete.go`). The service
receives the database interface via constructor injection and orchestrates the business
logic. No HTTP, no MongoDB, no Gin — only pure business logic and the domain interface.

### `internal/<domain>/handler/rest/`
`handler.go` defines the handler struct and its constructor, receiving the service via
injection. One file per HTTP verb (`post.go`, `get.go`, `put.go`, `delete.go`).
Handlers translate HTTP requests into service calls and service responses into HTTP
responses. The only place Gin is imported inside the domain boundary.

### `pkg/database/nosql/`
MongoDB adapter implementations. These implement the interfaces defined in
`pkg/database/database.go`. The adapter fulfills the contract; it never defines it.

### `pkg/server/server.go`
Gin server setup. Receives handler structs from `main.go` and registers routes.
Knows nothing about business logic.

### `cmd/api/main.go`
The only place where everything is wired together. Initializes telemetry, database
connections, adapters, services, and handlers in the correct dependency order.
Registers handlers to the server and starts it.

## Go Practices

### Error Handling
- Always wrap errors with context using `fmt.Errorf("operation description: %w", err)`
- Never swallow errors silently
- Domain errors are defined in the domain package and checked by handlers to map to
  HTTP status codes

### Context Propagation
- `context.Context` is always the first argument of every function that does I/O or
  calls another service
- Never store context in a struct
- Use the helpers in `pkg/telemetry` (`WithDomain`, `WithOperation`, `TraceID`) to
  attach telemetry metadata to `context.Context` for downstream logging

### Interfaces
- Define interfaces where they are consumed, not where they are implemented
- Database port interfaces live in `pkg/database/database.go`; the domain imports
  them via a type alias. This prevents adapter-package churn from touching the domain.
- Keep interfaces small — one method is fine

### Constructors
- Always use constructor functions (`NewService`, `NewHandler`, `NewAdapter`) over
  bare struct literals
- Constructors validate their dependencies and return errors if required fields are
  missing

### Typed Enums
- `pkg/types/order.go` defines `OrderStatus` and `NotificationChannel` as `string`-based
  custom types; always use the typed constants (`types.StatusPending`, `types.ChannelEmail`,
  etc.) rather than raw strings
- When passing these values to functions that expect `string`, cast explicitly: `string(status)`

### No Global State
- No package-level variables that are mutated at runtime
- No `init()` functions
- Everything is passed explicitly via constructors

## OpenTelemetry Conventions

- Every handler creates a span at the start of the request using the route as the
  span name
- Every service method propagates the context and creates a child span
- Errors are always recorded on the span with `span.RecordError(err)` before returning
- Metrics, logs, and trace names follow the pattern `<domain>.<operation>`
  (e.g. `order.create`, `catalog.get`)
- Structured log fields always include `domain`, `operation`, and `trace_id`

## Scripts Directory Policy

`/scripts/` contains the incident-response simulation scripts used for this exercise.
Do not open, read, or edit any file under `/scripts/` during research or investigation,
even if it seems relevant to explaining an incident. Do not summarize, quote, or infer
their contents from partial reads. Investigate the incident only through observability
tooling (Grafana, logs, traces, metrics) and application code outside `/scripts/`.
