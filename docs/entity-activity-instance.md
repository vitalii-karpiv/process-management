# Entity: ActivityInstance

## Overview

The **ActivityInstance** entity represents a single runtime execution of an activity within a process. It tracks the state, data, and execution attempts of an activity as it progresses.

---

## Purpose

- Track individual activity execution within a process
- Store activity-specific input and output data
- Manage retry attempts and failure handling
- Enable activity-level debugging and observability

---

## Attributes

### `id` (UUID, Primary Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Unique identifier for the activity instance
- **Generation**: Auto-generated on creation
- **Usage**: Used for correlation and idempotency

### `processInstanceId` (UUID, Foreign Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Reference to the parent process instance
- **Foreign Key**: `process_instance(id)`
- **On Delete**: CASCADE

### `activityDefinitionId` (UUID, Foreign Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Reference to the activity definition template
- **Foreign Key**: `activity_definition(id)`
- **On Delete**: RESTRICT

### `status` (Enum)
- **Type**: String (Enum)
- **Required**: Yes
- **Default**: `PENDING`
- **Description**: Current execution status of the activity
- **Allowed Values**:
  - `PENDING` - Activity queued but not started
  - `RUNNING` - Activity is currently executing
  - `WAITING` - Activity is waiting for external event/input
  - `COMPLETED` - Activity finished successfully
  - `FAILED` - Activity failed after all retries

### `inputData` (JSON)
- **Type**: JSONB
- **Required**: No
- **Description**: Input data for this activity execution
- **Example**:
```json
{
  "customerId": "cust_123",
  "documentType": "passport",
  "documentUrl": "https://..."
}
```
- **Source**: Derived from process input or previous activity output

### `outputData` (JSON)
- **Type**: JSONB
- **Required**: No (nullable)
- **Description**: Result data produced by the activity
- **Example**:
```json
{
  "verificationStatus": "verified",
  "confidence": 0.98,
  "verificationId": "ver_456"
}
```
- **Population**: Set when activity completes successfully

### `retryCount` (Integer)
- **Type**: Integer
- **Required**: Yes
- **Default**: 0
- **Description**: Number of times this activity has been retried
- **Constraints**: Must be >= 0
- **Usage**: Compared against maxRetries to determine if retries are exhausted

### `maxRetries` (Integer)
- **Type**: Integer
- **Required**: Yes
- **Description**: Maximum number of retry attempts allowed
- **Source**: Copied from ActivityDefinition.retryPolicy.maxRetries
- **Default**: 3

### `timeout` (Integer)
- **Type**: Integer
- **Required**: Yes
- **Description**: Maximum execution time in milliseconds
- **Source**: Copied from ActivityDefinition.timeout
- **Default**: 30000 (30 seconds)

### `createdAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: Yes
- **Description**: When the activity instance was created
- **Generation**: Auto-generated on creation

### `updatedAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: Yes
- **Description**: When the activity instance was last modified
- **Generation**: Auto-updated on modification

### `startedAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: No (nullable)
- **Description**: When the activity started executing
- **Population**: Set when status changes to RUNNING

### `completedAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: No (nullable)
- **Description**: When the activity reached a terminal state
- **Population**: Set when status changes to COMPLETED or FAILED

### `failureReason` (Text)
- **Type**: Text
- **Required**: No (nullable)
- **Description**: Error message if activity failed
- **Example**: `"HTTP 500: Payment service unavailable after 3 retries"`
- **Population**: Set when activity fails

### `lastExecutionId` (String)
- **Type**: String
- **Required**: No (nullable)
- **Description**: Idempotency key of the last execution attempt
- **Usage**: Prevents duplicate processing of retry attempts
- **Format**: `${activityInstanceId}:${retryCount}:${timestamp}`

---

## Relationships

### Many-to-One: ActivityInstance → ProcessInstance
- Many activity instances belong to one process instance
- **Foreign Key**: `processInstanceId`

### Many-to-One: ActivityInstance → ActivityDefinition
- Many activity instances reference one activity definition
- **Foreign Key**: `activityDefinitionId`

---

## Constraints

1. **Foreign Key Constraint**: `processInstanceId` must exist in `process_instance`
2. **Foreign Key Constraint**: `activityDefinitionId` must exist in `activity_definition`
3. **Check Constraint**: `retryCount >= 0`
4. **Check Constraint**: `maxRetries >= 0`
5. **Check Constraint**: `timeout > 0`
6. **Check Constraint**: `retryCount <= maxRetries` (business rule, not DB constraint)
7. **Enum Constraint**: Status must be one of allowed values

---

## Indexes

```sql
CREATE INDEX idx_activity_instance_process_id ON activity_instance(process_instance_id);
CREATE INDEX idx_activity_instance_definition_id ON activity_instance(activity_definition_id);
CREATE INDEX idx_activity_instance_status ON activity_instance(status);
CREATE INDEX idx_activity_instance_process_status ON activity_instance(process_instance_id, status);
CREATE INDEX idx_activity_instance_created_at ON activity_instance(created_at DESC);
CREATE INDEX idx_activity_instance_execution_id ON activity_instance(last_execution_id);
```

---

## State Transitions

Valid state transitions:

```
PENDING → RUNNING
PENDING → FAILED (if validation fails)

RUNNING → COMPLETED
RUNNING → FAILED
RUNNING → WAITING
RUNNING → PENDING (retry)

WAITING → RUNNING
WAITING → FAILED (timeout)

Terminal states:
- COMPLETED
- FAILED
```

---

## Business Rules

1. **Retry Logic**: 
   - If activity fails and `retryCount < maxRetries`, retry by transitioning back to PENDING
   - If `retryCount >= maxRetries`, transition to FAILED

2. **Idempotency**:
   - Use `lastExecutionId` to detect duplicate execution attempts
   - Workers must check this before executing

3. **Timeout Handling**:
   - If execution exceeds `timeout`, mark as FAILED
   - Apply timeout at activity level, not just HTTP request level

4. **State Consistency**:
   - Activity state changes must be atomic with process state changes
   - Failed activities should update parent process status

5. **Data Flow**:
   - Output of activity N becomes part of input for activity N+1
   - Store both raw and transformed data

---

## Usage Examples

### Creating a new activity instance
```typescript
const activityInstance = {
  processInstanceId: 'uuid-of-process',
  activityDefinitionId: 'uuid-of-activity-def',
  status: 'PENDING',
  inputData: {
    customerId: 'cust_123',
    documentType: 'passport'
  },
  retryCount: 0,
  maxRetries: 3,
  timeout: 60000
};
```

### Starting activity execution
```typescript
const executionId = `${activityInstanceId}:${retryCount}:${Date.now()}`;

await activityInstanceRepository.update(
  { id: activityInstanceId },
  {
    status: 'RUNNING',
    startedAt: new Date(),
    lastExecutionId: executionId
  }
);
```

### Completing activity successfully
```typescript
await activityInstanceRepository.update(
  { id: activityInstanceId },
  {
    status: 'COMPLETED',
    completedAt: new Date(),
    outputData: {
      verificationStatus: 'verified',
      verificationId: 'ver_789'
    }
  }
);
```

### Handling failure with retry
```typescript
const shouldRetry = activity.retryCount < activity.maxRetries;

if (shouldRetry) {
  await activityInstanceRepository.update(
    { id: activityInstanceId },
    {
      status: 'PENDING',
      retryCount: activity.retryCount + 1,
      failureReason: error.message
    }
  );
} else {
  await activityInstanceRepository.update(
    { id: activityInstanceId },
    {
      status: 'FAILED',
      completedAt: new Date(),
      failureReason: error.message
    }
  );
}
```

---

## Execution Flow

1. **Creation**: Activity instance created with PENDING status
2. **Queue**: Activity job published to message queue
3. **Execution**: Worker picks up job
   - Check idempotency using `lastExecutionId`
   - Acquire distributed lock
   - Update status to RUNNING
   - Execute activity logic
4. **Completion**: Update status to COMPLETED/FAILED
5. **Retry** (if failed): Update status back to PENDING with incremented retryCount

---

## Observability

### Key Metrics
- Activity duration: `completedAt - startedAt`
- Success rate: `COUNT(COMPLETED) / COUNT(COMPLETED + FAILED)`
- Retry rate: Average of `retryCount`
- Timeout rate: Count of timeouts

### Queries
```sql
-- Activities pending for too long (stuck in queue)
SELECT * FROM activity_instance
WHERE status = 'PENDING'
  AND created_at < NOW() - INTERVAL '5 minutes';

-- Activities running longer than timeout
SELECT * FROM activity_instance
WHERE status = 'RUNNING'
  AND started_at < NOW() - (timeout || ' milliseconds')::INTERVAL;

-- Average activity duration by type
SELECT ad.type, 
       AVG(EXTRACT(EPOCH FROM (ai.completed_at - ai.started_at))) as avg_duration_seconds
FROM activity_instance ai
JOIN activity_definition ad ON ai.activity_definition_id = ad.id
WHERE ai.status = 'COMPLETED'
GROUP BY ad.type;
```

---

## Migration Strategy

### Phase 1 (MVP)
- Basic retry logic with exponential backoff
- Simple timeout handling
- Status tracking

### Phase 2 (Future)
- Add `executionHistory` JSONB field for detailed attempt tracking
- Add `priority` for activity queue management
- Add `scheduledFor` for delayed execution
- Support for activity compensation/rollback

---

## Related Entities

- **ProcessInstance**: Parent process containing this activity
- **ActivityDefinition**: Template this instance is based on
- **ProcessEvent**: Events emitted during activity execution
