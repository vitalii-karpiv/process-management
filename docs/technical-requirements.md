# Technical Requirements Document (TRD)
## Process Management Platform

---

## 1. Purpose

This document defines the technical requirements for implementing the **Process Management Platform** described in the Business Requirements Document (BRD).

The goal is to translate business needs into concrete technical decisions, system architecture, and implementation constraints.

---

## 2. Assumptions

- Runtime: Node.js (TypeScript)
- Message Broker: RabbitMQ
- Multi-tenancy: Phase 2 (single-tenant in v1)
- Average activities per process (v1): **5–20 activities**
- Processes may run from seconds to days
- PostgreSQL is the system of record

---

## 3. System Overview

The system is composed of two main runtime components:

1. **API Service**
2. **Worker Service**

These components communicate via:
- RabbitMQ (events & jobs)
- PostgreSQL (state persistence)
- Redis (locking & caching)

---

## 4. Architecture

### 4.1 Components

#### API Service
Responsibilities:
- Process creation
- Process querying
- Manual activity actions
- External event ingestion
- Authentication (basic in v1)

Constraints:
- Must not execute long-running logic
- Must remain stateless

---

#### Worker Service
Responsibilities:
- Activity execution
- Retry handling
- Timeout handling
- Process state transitions

Constraints:
- Must be horizontally scalable
- Must be idempotent

---

#### Message Broker (RabbitMQ)

Usage:
- Dispatch activity execution jobs
- Resume suspended processes
- Decouple API and Worker lifecycles

Delivery semantics:
- At-least-once delivery
- Explicit acknowledgements
- Dead-letter queues for failures

---

#### Database (PostgreSQL)

Role:
- Source of truth for all process state
- Durable recovery after crashes

Characteristics:
- Strong consistency
- Transactional guarantees
- Schema migrations required

---

#### Cache (Redis)

Usage:
- Distributed locks
- Idempotency keys
- Hot process state caching
- Retry counters

Redis is **not** a source of truth.

---

## 5. Technology Stack

| Layer | Technology |
|-----|-----------|
| Runtime | Node.js (LTS) |
| Language | TypeScript |
| Framework | NestJS |
| Database | PostgreSQL |
| Message Broker | RabbitMQ |
| Cache | Redis |
| ORM | TypeORM |
| Containerization | Docker |
| Deployment | Container-based |

---

## 6. Execution Model

### 6.1 Process Lifecycle

Process states:
- CREATED
- IN_PROGRESS
- WAITING
- COMPLETED
- FAILED
- CANCELLED

State transitions:
- Must be explicit
- Must be persisted
- Must be validated

---

### 6.2 Activity Lifecycle

Activity states:
- PENDING
- RUNNING
- WAITING
- COMPLETED
- FAILED

Each activity execution must:
- Be idempotent
- Persist execution result
- Record start and end timestamps

---

### 6.3 Activity Execution

Execution flow:
1. Worker consumes activity job
2. Acquire distributed lock
3. Load process and activity state
4. Execute activity
5. Persist result
6. Emit next event

Failures:
- Trigger retries (configurable)
- Escalate to process failure if unrecoverable

---

## 7. Activity Types (Technical View)

Supported executors:
- HTTP Call Executor
- Event Trigger Executor
- Wait Executor
- Decision Executor
- Transformation Executor
- Manual Executor

Executor selection is based on activity type.

---

## 8. Retry & Idempotency Strategy

- Each activity execution has a unique execution ID
- Duplicate deliveries must be safely ignored
- Retries must use exponential backoff
- Max retry count configurable per activity

Idempotency mechanisms:
- Execution ID persistence
- Redis idempotency keys
- Optimistic locking via DB versioning

---

## 9. Locking Strategy

- Distributed locks via Redis
- Lock scoped to process ID
- Lock TTL required to avoid deadlocks
- Lock acquisition mandatory before activity execution

---

## 10. Data Model (High-Level)

Core entities:
- Process
- Activity Definition
- Activity Execution
- Process State History
- Process Event

Constraints:
- Process state must be consistent with activity states
- History must be append-only
- No destructive updates to audit data

---

## 11. API Design Principles

- REST-based
- JSON payloads
- Idempotent endpoints where applicable
- Clear error contracts

Example endpoints:
- POST /processes
- GET /processes/{id}
- GET /processes/{id}/history
- POST /processes/{id}/activities/{activityId}/resume

---

## 12. Error Handling

Error categories:
- Validation errors
- Execution errors
- External system failures
- Timeout errors

All errors must:
- Be logged
- Be associated with process ID
- Be persisted for audit

---

## 13. Observability

### Logging
- Structured logs (JSON)
- Correlation by process ID
- Log activity start/end

### Metrics
- Process duration
- Activity duration
- Retry counts
- Failure rates

### Tracing (Phase 2)
- Distributed tracing across services

---

## 14. Security

v1:
- Basic API authentication
- Secrets injected via environment variables

Phase 2:
- Tenant isolation
- Role-based access control
- Encrypted payload fields
- Audit access controls

---

## 15. Scalability Requirements

- API and Worker services must scale independently
- Workers must support parallel execution
- RabbitMQ must support horizontal scaling

---

## 16. Non-Goals (v1)

- Visual process designer
- Embedded scripting engine
- SLA enforcement
- Billing and usage tracking

---

## 17. Risks & Mitigations

| Risk | Mitigation |
|----|-----------|
| Stuck processes | Timeouts + retries |
| Duplicate execution | Idempotency |
| External system failure | Retry & wait activities |
| Worker crashes | At-least-once delivery |

---

## 18. MVP Scope (v1)

Included:
- API-based process creation
- Core activity types
- Reliable execution
- Full audit trail

Excluded:
- UI
- Multi-tenancy
- Advanced analytics

