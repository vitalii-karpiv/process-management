# Project Structure - Created

## ✅ Folder Structure Created

```
process-management/
├── apps/
│   ├── api/                           # API Service
│   │   ├── src/
│   │   │   ├── controllers/           # HTTP controllers
│   │   │   ├── dto/                   # Request/response DTOs
│   │   │   ├── modules/               # API-specific modules
│   │   │   ├── config/                # API configuration
│   │   │   ├── main.ts                # Entry point
│   │   │   └── app.module.ts          # Root module
│   │   ├── test/                      # API tests
│   │   ├── package.json
│   │   ├── nest-cli.json
│   │   └── tsconfig.build.json
│   │
│   └── worker/                        # Worker Service
│       ├── src/
│       │   ├── consumers/             # RabbitMQ consumers
│       │   ├── executors/             # Activity executors
│       │   ├── schedulers/            # Timeout/delayed jobs
│       │   ├── config/                # Worker configuration
│       │   ├── main.ts                # Entry point
│       │   └── worker.module.ts       # Root module
│       ├── test/                      # Worker tests
│       ├── package.json
│       ├── nest-cli.json
│       └── tsconfig.build.json
│
├── libs/
│   ├── domain/                        # Domain Layer (Business Logic)
│   │   ├── src/
│   │   │   ├── process/
│   │   │   │   ├── entities/          # Process & activity entities
│   │   │   │   ├── enums/             # Status enums
│   │   │   │   ├── services/          # Domain services
│   │   │   │   ├── events/            # Domain events
│   │   │   │   ├── interfaces/        # Repository interfaces
│   │   │   │   ├── state-machine/     # State transition logic
│   │   │   │   └── process.module.ts
│   │   │   └── index.ts
│   │   ├── test/
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── infrastructure/                # Infrastructure Layer
│   │   ├── src/
│   │   │   ├── database/              # TypeORM, repositories
│   │   │   ├── redis/                 # Cache & locks
│   │   │   ├── transactions/          # Transaction helpers
│   │   │   ├── config/                # Infrastructure config
│   │   │   ├── infrastructure.module.ts
│   │   │   └── index.ts
│   │   ├── test/
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── messaging/                     # Messaging Layer
│       ├── src/
│       │   ├── publishers/            # Event publishers
│       │   ├── consumers/             # Base consumer abstractions
│       │   ├── schemas/               # Message schemas
│       │   ├── constants/             # Routing keys, queues
│       │   ├── messaging.module.ts
│       │   └── index.ts
│       ├── test/
│       ├── package.json
│       └── tsconfig.json
│
├── docs/                              # Documentation
│   ├── api-endpoints.md
│   ├── business-requirements.md
│   ├── technical-requirements.md
│   ├── project-structure.md
│   ├── entity-*.md                    # Entity documentation
│   └── ...
│
├── docker-compose.yml                 # Local development services
├── .env.example                       # Environment variables template
├── package.json                       # Root package (workspace)
├── tsconfig.json                      # Root TypeScript config
├── .gitignore
├── .prettierrc
├── .eslintrc.js
└── README.md
```

## 📦 Workspaces Configuration

The project uses **npm workspaces** for monorepo management:

- `apps/api` - API service
- `apps/worker` - Worker service  
- `libs/domain` - Domain layer
- `libs/infrastructure` - Infrastructure layer
- `libs/messaging` - Messaging layer

## 🏗️ Architecture Layers

### 1. **apps/api** - API Service
- HTTP REST endpoints
- Request validation
- Authentication
- Delegates to domain services
- **No business logic**

### 2. **apps/worker** - Worker Service
- Consumes messages from RabbitMQ
- Executes activities
- Handles retries and timeouts
- Updates process state
- **Asynchronous processing**

### 3. **libs/domain** - Domain Layer
- Core business logic
- Entity definitions
- State machines
- Domain events
- **Framework-agnostic**

### 4. **libs/infrastructure** - Infrastructure Layer
- Database access (TypeORM)
- Redis cache and locks
- External API clients
- Implements domain interfaces

### 5. **libs/messaging** - Messaging Layer
- Event publishing
- Message consumption
- RabbitMQ configuration
- Message schemas

## 🔗 Dependency Rules

```
✅ Allowed:
apps → libs
infrastructure → domain
messaging → domain

❌ Forbidden:
domain → infrastructure
domain → messaging
libs → apps
api → worker
worker → api
```

## 🚀 Getting Started

### 1. Install Dependencies
```bash
npm install
```

### 2. Start Infrastructure (Docker)
```bash
docker-compose up -d
```

This starts:
- PostgreSQL (port 5432)
- Redis (port 6379)
- RabbitMQ (port 5672, Management UI: 15672)

### 3. Configure Environment
```bash
cp .env.example .env
# Edit .env with your settings
```

### 4. Build All Packages
```bash
npm run build
```

### 5. Start API Service
```bash
npm run start:api:dev
```

### 6. Start Worker Service (in another terminal)
```bash
npm run start:worker:dev
```

## 📝 Available Scripts

### Root Level
```bash
npm run build              # Build all workspaces
npm run build:api          # Build API only
npm run build:worker       # Build Worker only
npm run start:api          # Start API (production)
npm run start:api:dev      # Start API (development)
npm run start:worker       # Start Worker (production)
npm run start:worker:dev   # Start Worker (development)
npm run test               # Run all tests
npm run lint               # Lint all code
npm run lint:fix           # Fix linting issues
npm run format             # Format code
```

## 🗃️ Database Setup

Database migrations will be added using TypeORM migrations.

```bash
# Generate migration
npm run migration:generate --workspace=@process-platform/infrastructure

# Run migrations
npm run migration:run --workspace=@process-platform/infrastructure
```

## 🧪 Testing

Each workspace has its own test suite:

```bash
# Test specific workspace
npm run test --workspace=@process-platform/domain
npm run test --workspace=@process-platform/api

# Test with coverage
npm run test:cov --workspace=@process-platform/domain
```

## 🐳 Docker Services

Access the services:

- **PostgreSQL**: `localhost:5432`
- **Redis**: `localhost:6379`
- **RabbitMQ Management**: http://localhost:15672
  - Username: `process_user`
  - Password: `process_password`

## 📊 Next Steps

1. **Implement Domain Entities** (libs/domain/src/process/entities/)
   - ProcessDefinition
   - ActivityDefinition
   - ProcessInstance
   - ActivityInstance
   - ProcessStateHistory
   - ProcessEvent

2. **Create Database Schema** (libs/infrastructure/src/database/)
   - TypeORM entities
   - Repositories
   - Migrations

3. **Implement API Controllers** (apps/api/src/controllers/)
   - ProcessDefinitionController
   - ProcessInstanceController
   - ActivityInstanceController

4. **Implement Worker Executors** (apps/worker/src/executors/)
   - ApiCallExecutor
   - EventTriggerExecutor
   - WaitExecutor
   - DecisionExecutor
   - TransformationExecutor
   - ManualExecutor

5. **Setup Messaging** (libs/messaging/src/)
   - Event publishers
   - Message consumers
   - Queue configuration

## 📚 Documentation

All entity and API documentation is in the `docs/` folder:
- Entity definitions: `entity-*.md`
- API endpoints: `api-endpoints.md`
- Requirements: `business-requirements.md`, `technical-requirements.md`

---

**Status**: ✅ Project structure created and ready for implementation
