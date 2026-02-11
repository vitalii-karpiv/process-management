# Entity: ProcessInstance

## Overview

The **ProcessInstance** entity represents a single runtime execution of a process definition. It maintains the current state, progress, and data for an active or completed process.

---

## Purpose

- Track the lifecycle of a running process
- Store process state and execution context
- Enable process recovery after failures
- Provide visibility into process progress

---

## Attributes

### `id` (UUID, Primary Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Unique identifier for the process instance
- **Generation**: Auto-generated on creation
- **Usage**: Used for correlation across logs, events, and API calls

### `processDefinitionId` (UUID, Foreign Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Reference to the process definition this instance is based on
- **Foreign Key**: `process_definition(id)`
- **On Delete**: RESTRICT (cannot delete definitions with active instances)

### `status` (Enum)
- **Type**: String (Enum)
- **Required**: Yes
- **Default**: `CREATED`
- **Description**: Current lifecycle status of the process
- **Allowed Values**:
  - `CREATED` - Process created but not started
  - `IN_PROGRESS` - Process is actively executing
  - `WAITING` - Process is paused waiting for external input/event
  - `COMPLETED` - Process finished successfully
  - `FAILED` - Process failed and cannot continue
  - `CANCELLED` - Process was manually cancelled

### `currentActivityInstanceId` (UUID, Foreign Key)
- **Type**: UUID
- **Required**: No (nullable)
- **Description**: Reference to the currently executing activity instance
- **Foreign Key**: `activity_instance(id)`
- **Usage**: Indicates where the process is in its execution flow

### `inputPayload` (JSON)
- **Type**: JSONB
- **Required**: Yes
- **Description**: Initial input data provided when process was created
- **Example**:
```json
{
  "customerId": "cust_123",
  "orderItems": [...],
  "deliveryAddress": {...}
}
```
- **Immutability**: Should not be modified after creation

### `outputPayload` (JSON)
- **Type**: JSONB
- **Required**: No (nullable)
- **Description**: Final result data after process completion
- **Example**:
```json
{
  "orderId": "order_456",
  "status": "confirmed",
  "trackingNumber": "TRACK123"
}
```
- **Population**: Set when process reaches COMPLETED status

### `metadata` (JSON)
- **Type**: JSONB
- **Required**: No
- **Default**: `{}`
- **Description**: Additional context and runtime data
- **Example**:
```json
{
  "correlationId": "req_789",
  "source": "web-api",
  "priority": "high",
  "tags": ["urgent", "vip-customer"]
}
```

### `version` (Integer)
- **Type**: Integer
- **Required**: Yes
- **Default**: 1
- **Description**: Optimistic locking version to prevent concurrent updates
- **Usage**: Incremented on each state change
- **Pattern**: Check version before update, fail if mismatch

### `createdAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: Yes
- **Description**: When the process instance was created
- **Generation**: Auto-generated on creation

### `updatedAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: Yes
- **Description**: When the process instance was last modified
- **Generation**: Auto-updated on any modification

### `startedAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: No (nullable)
- **Description**: When the process started executing (transitioned to IN_PROGRESS)
- **Population**: Set when status changes from CREATED to IN_PROGRESS

### `completedAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: No (nullable)
- **Description**: When the process reached a terminal state
- **Population**: Set when status changes to COMPLETED, FAILED, or CANCELLED
- **Usage**: Calculate process duration

### `failureReason` (Text)
- **Type**: Text
- **Required**: No (nullable)
- **Description**: Detailed error message if process failed
- **Example**: `"Activity 'verify-payment' failed after 3 retries: Connection timeout to payment service"`
- **Population**: Set when status changes to FAILED

---

## Relationships

### Many-to-One: ProcessInstance → ProcessDefinition
- Many process instances belong to one process definition
- **Foreign Key**: `processDefinitionId`

### One-to-Many: ProcessInstance → ActivityInstance
- One process instance contains multiple activity instances
- **Cascade**: Delete activity instances when process instance is deleted (or use soft delete)

### One-to-Many: ProcessInstance → ProcessStateHistory
- One process instance has multiple state history records
- **Cascade**: Delete history when process instance is deleted (or archive)

### One-to-Many: ProcessInstance → ProcessEvent
- One process instance emits multiple events
- **Cascade**: Delete events when process instance is deleted (or archive)

---

## Constraints

1. **Foreign Key Constraint**: `processDefinitionId` must exist in `process_definition`
2. **Foreign Key Constraint**: `currentActivityInstanceId` must exist in `activity_instance` (if not null)
3. **Check Constraint**: `version > 0`
4. **Check Constraint**: Status must be one of allowed enum values
5. **Business Rule**: `completedAt` must be NULL if status is not terminal
6. **Business Rule**: `failureReason` should only be set if status is FAILED

---

## Indexes

```sql
CREATE INDEX idx_process_instance_definition_id ON process_instance(process_definition_id);
CREATE INDEX idx_process_instance_status ON process_instance(status);
CREATE INDEX idx_process_instance_created_at ON process_instance(created_at DESC);
CREATE INDEX idx_process_instance_current_activity ON process_instance(current_activity_instance_id);
CREATE INDEX idx_process_instance_status_updated ON process_instance(status, updated_at);
```

---

## State Transitions

Valid state transitions:

```
CREATED → IN_PROGRESS
CREATED → CANCELLED

IN_PROGRESS → WAITING
IN_PROGRESS → COMPLETED
IN_PROGRESS → FAILED
IN_PROGRESS → CANCELLED

WAITING → IN_PROGRESS
WAITING → FAILED
WAITING → CANCELLED

Terminal states (no outgoing transitions):
- COMPLETED
- FAILED
- CANCELLED
```

---

## Business Rules

1. **Atomicity**: Status changes must be atomic with activity state changes
2. **Idempotency**: Multiple updates with same version should be rejected
3. **Auditability**: All status changes must be recorded in ProcessStateHistory
4. **Recovery**: Failed processes may be retried (creating a new instance or resuming)
5. **Immutability**: InputPayload should never be modified after creation
6. **Completion**: OutputPayload is only set upon successful completion

---

## Usage Examples

### Creating a new process instance
```typescript
const processInstance = {
  processDefinitionId: 'uuid-of-definition',
  status: 'CREATED',
  inputPayload: {
    customerId: 'cust_123',
    orderAmount: 99.99
  },
  metadata: {
    correlationId: 'web_request_456',
    source: 'web-checkout'
  },
  version: 1
};
```

### Updating process status
```typescript
// Must check version for optimistic locking
await processInstanceRepository.update(
  {
    id: processId,
    version: currentVersion
  },
  {
    status: 'IN_PROGRESS',
    startedAt: new Date(),
    version: currentVersion + 1
  }
);
```

### Completing a process
```typescript
await processInstanceRepository.update(
  { id: processId, version: currentVersion },
  {
    status: 'COMPLETED',
    completedAt: new Date(),
    outputPayload: {
      orderId: 'order_789',
      status: 'confirmed'
    },
    version: currentVersion + 1
  }
);
```

---

## Observability

### Key Metrics
- Process duration: `completedAt - startedAt`
- Creation rate: Count by `createdAt`
- Completion rate: Count where `status = 'COMPLETED'`
- Failure rate: Count where `status = 'FAILED'`
- Active processes: Count where `status IN ('IN_PROGRESS', 'WAITING')`

### Queries
```sql
-- Find stuck processes (no update in 1 hour)
SELECT * FROM process_instance
WHERE status IN ('IN_PROGRESS', 'WAITING')
  AND updated_at < NOW() - INTERVAL '1 hour';

-- Average process duration by definition
SELECT process_definition_id, 
       AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) as avg_duration_seconds
FROM process_instance
WHERE status = 'COMPLETED'
GROUP BY process_definition_id;
```

---

## Migration Strategy

### Phase 1 (MVP)
- Single tenant (no tenantId field)
- Basic status tracking
- Manual cancellation only

### Phase 2 (Future)
- Add `tenantId` for multi-tenancy
- Add `priority` for queue management
- Add `parentProcessInstanceId` for sub-processes
- Add soft delete via `deletedAt` timestamp

---

## Related Entities

- **ProcessDefinition**: Template this instance is based on
- **ActivityInstance**: Activities executed within this process
- **ProcessStateHistory**: Audit trail of status changes
- **ProcessEvent**: Events emitted during execution
