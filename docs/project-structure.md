# Project Structure
## Process Management Platform

This document describes the logical and physical structure of the **Process Management Platform** codebase.

The project follows a **monorepo architecture** with:
- multiple runtime applications
- shared libraries
- strict separation of concerns

The structure is designed to support:
- API + Worker architecture
- long-running process orchestration
- future evolution to microservices
- high maintainability and testability

---

## High-Level Overview

process-management/
├── apps/
│   ├── api/
│   └── worker/
├── libs/
│   ├── domain/
│   ├── infrastructure/
│   └── messaging/
└── package.json


### Core Principles
- **apps/** contain runtime entry points
- **libs/** contain reusable logic
- Business logic lives only in **domain**
- Infrastructure concerns are isolated
- No direct coupling between API and Worker

---

## apps/

The `apps/` directory contains executable applications.  
Each application has its own runtime, lifecycle, and deployment model.

Applications must **never contain core business logic**.

### apps/api/

The **API Service** is responsible for synchronous interactions with external clients.

#### Responsibilities
- Process creation
- Process querying
- Manual activity actions
- External event ingestion
- Request validation
- Authentication and authorization (future)

#### Non-Responsibilities
- Activity execution
- Long-running tasks
- Retry logic
- State transitions

#### Typical Structure

apps/api/
├── src/
│ ├── main.ts # Application entry point
│ ├── app.module.ts # Root NestJS module
│ ├── controllers/ # HTTP controllers
│ ├── dto/ # Request/response DTOs
│ ├── modules/ # API-specific modules
│ └── config/ # API configuration
└── package.json


#### Rules
- Controllers call domain services only
- No direct database access
- No message consumption
- No business rules

---

### apps/worker/

The **Worker Service** executes activities asynchronously.

#### Responsibilities
- Activity execution
- Retry handling
- Timeout handling
- Process state transitions
- Event consumption

#### Non-Responsibilities
- HTTP request handling
- Authentication
- Direct client interaction

#### Typical Structure

apps/worker/
├── src/
│ ├── main.ts # Worker entry point
│ ├── worker.module.ts # Root worker module
│ ├── consumers/ # RabbitMQ consumers
│ ├── executors/ # Activity executors
│ ├── schedulers/ # Timeout / delayed jobs
│ └── config/ # Worker configuration
└── package.json


#### Rules
- Workers must be idempotent
- Workers must acquire locks before execution
- Workers must persist all state changes
- Workers communicate only via DB and messaging

---

## libs/

The `libs/` directory contains shared libraries used by both API and Worker.

Libraries **must not depend on applications**.

---

### libs/domain/

The **domain library** is the heart of the system.

It contains:
- business rules
- process orchestration logic
- activity state machines

This library is **framework-agnostic** (no HTTP, no MQ specifics).

#### Responsibilities
- Process lifecycle management
- Activity lifecycle management
- State transition validation
- Domain events
- Business invariants

#### Typical Structure

libs/domain/
├── src/
│ ├── process/
│ │ ├── entities/ # Process & activity entities
│ │ ├── enums/ # Status enums
│ │ ├── services/ # Domain services
│ │ ├── events/ # Domain events
│ │ ├── state-machine/ # State transition logic
│ │ └── process.module.ts
│ └── index.ts
└── package.json


#### Rules
- No NestJS controllers
- No infrastructure details
- Depends only on abstractions
- Fully unit-testable

---

### libs/infrastructure/

The **infrastructure library** provides concrete implementations for external systems.

#### Responsibilities
- Database access
- Redis integration
- Transaction management
- Distributed locks
- External system clients

#### Typical Structure

libs/infrastructure/
├── src/
│ ├── database/ # ORM, repositories
│ ├── redis/ # Cache & locks
│ ├── transactions/ # Transaction helpers
│ ├── config/ # Infrastructure config
│ └── infrastructure.module.ts
└── package.json


#### Rules
- Implements interfaces defined by domain
- No business logic
- No process orchestration
- Can be swapped or extended

---

### libs/messaging/

The **messaging library** abstracts message-based communication.

#### Responsibilities
- Event publishing
- Message consumption base classes
- Message schemas and contracts
- RabbitMQ configuration

#### Typical Structure

libs/messaging/
├── src/
│ ├── publishers/ # Event publishers
│ ├── consumers/ # Base consumer abstractions
│ ├── schemas/ # Message schemas
│ ├── constants/ # Routing keys, queues
│ └── messaging.module.ts
└── package.json


#### Rules
- No business logic
- No DB access
- Messaging contracts must be versioned
- Consumers delegate to domain services

---

## Root-Level Files

### package.json
- Declares workspace configuration
- Defines shared dependencies
- Defines build and lint scripts

---

## Dependency Rules (Very Important)

Allowed dependencies:

apps → libs
domain → nothing
infrastructure → domain
messaging → domain

Forbidden dependencies:

domain → infrastructure
domain → messaging
libs → apps
api → worker
worker → api

---

## Design Benefits

This structure provides:
- Clear ownership of responsibilities
- Safe parallel development
- Easy testability
- Independent scaling of API and Worker
- Clean migration path to microservices

---

## Summary

The project structure enforces:
- Separation of concerns
- Explicit boundaries
- Product-grade maintainability

This structure is intentionally strict to prevent architectural drift as the system grows.
