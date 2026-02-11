# Process Management Platform

A **process orchestration platform** for building, executing, and monitoring long-running business processes using an API-first, event-driven architecture.

This project is designed as a **real, sellable backend product**, not a demo or toy example.

---

## 🚀 What is this?

The Process Management Platform allows teams to:

- Define business **processes**
- Orchestrate nested **activities**
- Execute long-running workflows (minutes → days)
- Integrate with external systems via APIs and events
- Handle retries, failures, and timeouts safely
- Track full execution history for audit and debugging

The platform focuses on **orchestration**, not business logic.

> Your systems do the work.  
> This platform guarantees the process runs reliably.

---

## 🧠 Key Concepts

### Process
A **process** represents a single execution of a business workflow  
(e.g. customer onboarding, order fulfillment, data synchronization).

### Activity
An **activity** is an atomic orchestration unit within a process.  
A process consists of one or more ordered or conditional activities.

Examples:
- Call an external API
- Wait for a callback
- Require manual approval
- Trigger an event
- Make a decision

---

## ✨ Features

- API-first process creation and querying
- Asynchronous activity execution
- Event-driven architecture (RabbitMQ)
- Safe retries and idempotent execution
- Persistent process and activity state
- Full audit trail and execution history
- Scalable API and Worker services
- Designed for future multi-tenancy

---

## 🏗️ Architecture Overview

The system is built as a **Modular Monolith with Workers**:

- **API Service**
  - Handles HTTP requests
  - Creates and queries processes
  - Accepts external events and manual actions

- **Worker Service**
  - Executes activities asynchronously
  - Handles retries, timeouts, and failures
  - Advances process state

- **PostgreSQL**
  - Source of truth for all process state

- **RabbitMQ**
  - Asynchronous job and event delivery

- **Redis**
  - Distributed locks
  - Idempotency
  - Lightweight caching

API and Worker are deployed and scaled independently.

---

## 🛠️ Technology Stack

- **Runtime:** Node.js (TypeScript)
- **Framework:** NestJS
- **Database:** PostgreSQL
- **Message Broker:** RabbitMQ
- **Cache / Locks:** Redis
- **Containerization:** Docker
- **Architecture Style:** Event-driven, API + Workers

---

## 🔄 Execution Model (Simplified)

1. Client creates a process via API
2. Process state is persisted in PostgreSQL
3. Activity execution jobs are published to RabbitMQ
4. Workers consume jobs and execute activities
5. State transitions are persisted and audited
6. Process continues until completion or failure

All executions are **at-least-once** with idempotency guarantees.

---

## 📦 Use Cases

- Customer onboarding
- Order fulfillment
- Payment or billing workflows
- Data synchronization between systems
- Approval and compliance flows
- Long-running integrations

---

## 📊 Observability

The platform provides:
- Full process and activity history
- Execution timestamps
- Failure reasons
- Retry counts

Designed to integrate with logging and metrics systems.

---

## 🚧 Project Status

**Current phase:** Structure setup complete, beginning implementation

Completed:
- [x] Process & activity definition schema
- [x] API specification (REST endpoints)
- [x] Docker-based local setup
- [x] Project structure (monorepo with workspaces)

In Progress:
- [ ] Domain entities implementation
- [ ] Database schema (TypeORM)
- [ ] Core execution engine
- [ ] API controllers
- [ ] Worker executors

Planned:
- [ ] Integration tests
- [ ] MVP release

---

## 🔮 Roadmap

- Visual process designer (future)
- Multi-tenancy support
- SLA tracking and analytics
- Advanced activity types
- Role-based access control

---

## 🚀 Quick Start

### Prerequisites
- Node.js 18+ and npm 9+
- Docker and Docker Compose

### Setup

```bash
# 1. Install dependencies
npm install

# 2. Start infrastructure services
docker-compose up -d

# 3. Copy environment variables
cp .env.example .env

# 4. Build all packages
npm run build

# 5. Start API service
npm run start:api:dev

# 6. Start Worker service (in another terminal)
npm run start:worker:dev
```

API will be available at: `http://localhost:3000/api/v1`

---

## 📄 Documentation

- `docs/business-requirements.md` — Business Requirements
- `docs/technical-requirements.md` — Technical Requirements
- `docs/project-structure.md` — Codebase architecture
- `docs/api-endpoints.md` — REST API specification
- `docs/entity-*.md` — Entity definitions
- `STRUCTURE.md` — Project structure guide


