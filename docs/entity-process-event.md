# Entity: ProcessEvent

## Overview

The **ProcessEvent** entity represents domain events emitted during process execution. These events enable event-driven architecture, support external integrations, and provide an event log for process reconstruction.

---

## Purpose

- Enable event-driven integrations with external systems
- Support webhooks and notifications
- Provide event sourcing capability
- Enable process replay and debugging
- Facilitate asynchronous communication

---

## Attributes

### `id` (UUID, Primary Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Unique identifier for the event
- **Generation**: Auto-generated on creation
- **Usage**: Used for event deduplication and ordering

### `processInstanceId` (UUID, Foreign Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Reference to the process instance that emitted this event
- **Foreign Key**: `process_instance(id)`
- **On Delete**: CASCADE (or archive)

### `activityInstanceId` (UUID, Foreign Key)
- **Type**: UUID
- **Required**: No (nullable)
- **Description**: Reference to the activity instance that triggered this event (if applicable)
- **Foreign Key**: `activity_instance(id)`
- **Usage**: NULL for process-level events, populated for activity-level events

### `eventType` (Enum)
- **Type**: String (Enum)
- **Required**: Yes
- **Description**: Type of event that occurred
- **Allowed Values**:
  - **Process Events:**
    - `PROCESS_CREATED` - Process instance was created
    - `PROCESS_STARTED` - Process execution began
    - `PROCESS_COMPLETED` - Process finished successfully
    - `PROCESS_FAILED` - Process failed
    - `PROCESS_CANCELLED` - Process was cancelled
    - `PROCESS_WAITING` - Process is waiting for external input
  - **Activity Events:**
    - `ACTIVITY_STARTED` - Activity execution started
    - `ACTIVITY_COMPLETED` - Activity finished successfully
    - `ACTIVITY_FAILED` - Activity failed
    - `ACTIVITY_RETRYING` - Activity is being retried
    - `ACTIVITY_WAITING` - Activity waiting for external event

### `payload` (JSON)
- **Type**: JSONB
- **Required**: Yes
- **Description**: Event data containing relevant context and information
- **Structure**: Varies by event type (see examples below)
- **Usage**: Contains all information needed for consumers to react to the event

### `timestamp` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: Yes
- **Description**: When the event was created
- **Generation**: Auto-generated on creation
- **Precision**: Microseconds for ordering
- **Usage**: Critical for event ordering and replay

### `published` (Boolean)
- **Type**: Boolean
- **Required**: Yes
- **Default**: false
- **Description**: Whether this event has been published to external systems
- **Usage**: Supports eventual consistency and retry logic for external publishing

### `publishedAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: No (nullable)
- **Description**: When the event was successfully published externally
- **Population**: Set when external publication succeeds

---

## Relationships

### Many-to-One: ProcessEvent → ProcessInstance
- Many events belong to one process instance
- **Foreign Key**: `processInstanceId`

### Many-to-One: ProcessEvent → ActivityInstance
- Many events may reference one activity instance
- **Foreign Key**: `activityInstanceId`

---

## Constraints

1. **Foreign Key Constraint**: `processInstanceId` must exist in `process_instance`
2. **Foreign Key Constraint**: `activityInstanceId` must exist in `activity_instance` (if not null)
3. **Check Constraint**: `eventType` must be a valid enum value
4. **Check Constraint**: `payload` must not be empty
5. **Check Constraint**: If `published = true`, then `publishedAt` must not be null
6. **Immutability**: Events should never be updated after insertion (append-only)

---

## Indexes

```sql
CREATE INDEX idx_process_event_process_id ON process_event(process_instance_id);
CREATE INDEX idx_process_event_activity_id ON process_event(activity_instance_id);
CREATE INDEX idx_process_event_type ON process_event(event_type);
CREATE INDEX idx_process_event_timestamp ON process_event(timestamp DESC);
CREATE INDEX idx_process_event_process_timestamp ON process_event(process_instance_id, timestamp);
CREATE INDEX idx_process_event_published ON process_event(published) WHERE published = false;
CREATE INDEX idx_process_event_type_timestamp ON process_event(event_type, timestamp DESC);
```

---

## Event Payload Examples

### PROCESS_CREATED
```json
{
  "processDefinitionId": "uuid-of-definition",
  "processDefinitionName": "customer-onboarding",
  "processDefinitionVersion": 1,
  "inputPayload": {
    "customerId": "cust_123",
    "email": "customer@example.com"
  },
  "metadata": {
    "correlationId": "req_456",
    "source": "web-api"
  }
}
```

### PROCESS_STARTED
```json
{
  "processDefinitionName": "customer-onboarding",
  "startedAt": "2026-02-11T10:00:00Z",
  "firstActivityId": "uuid-of-activity",
  "firstActivityName": "verify-identity"
}
```

### PROCESS_COMPLETED
```json
{
  "processDefinitionName": "customer-onboarding",
  "completedAt": "2026-02-11T10:05:30Z",
  "durationMs": 330000,
  "outputPayload": {
    "accountId": "acc_789",
    "status": "active"
  },
  "totalActivities": 5,
  "completedActivities": 5
}
```

### PROCESS_FAILED
```json
{
  "processDefinitionName": "customer-onboarding",
  "failedAt": "2026-02-11T10:03:15Z",
  "durationMs": 195000,
  "failureReason": "Activity 'verify-identity' failed after 3 retries",
  "failedActivityId": "uuid-of-activity",
  "failedActivityName": "verify-identity",
  "errorDetails": "Connection timeout to identity service"
}
```

### ACTIVITY_STARTED
```json
{
  "activityDefinitionId": "uuid-of-definition",
  "activityName": "verify-identity",
  "activityType": "API_CALL",
  "startedAt": "2026-02-11T10:00:05Z",
  "inputData": {
    "customerId": "cust_123",
    "documentType": "passport"
  },
  "retryCount": 0
}
```

### ACTIVITY_COMPLETED
```json
{
  "activityName": "verify-identity",
  "activityType": "API_CALL",
  "completedAt": "2026-02-11T10:00:08Z",
  "durationMs": 3000,
  "outputData": {
    "verificationStatus": "verified",
    "confidence": 0.98
  },
  "retryCount": 0
}
```

### ACTIVITY_FAILED
```json
{
  "activityName": "charge-payment",
  "activityType": "API_CALL",
  "failedAt": "2026-02-11T10:02:30Z",
  "failureReason": "Payment declined",
  "errorCode": "INSUFFICIENT_FUNDS",
  "errorDetails": "Card has insufficient funds",
  "retryCount": 3,
  "maxRetries": 3,
  "willRetry": false
}
```

### ACTIVITY_WAITING
```json
{
  "activityName": "wait-for-approval",
  "activityType": "WAIT",
  "waitingFor": "MANUAL_ACTION",
  "waitingSince": "2026-02-11T10:03:00Z",
  "timeoutAt": "2026-02-12T10:03:00Z",
  "assignedTo": "compliance-team",
  "approvalUrl": "https://app.example.com/approvals/uuid"
}
```

---

## Business Rules

1. **Immutability**: Events must never be updated or deleted (append-only log)
2. **Ordering**: Events must be ordered by timestamp within a process
3. **Atomicity**: Event creation must be atomic with state changes
4. **Idempotency**: Duplicate event detection using event ID
5. **Publishing**: External publication failures should not prevent event storage
6. **Retry**: Failed publications should be retried asynchronously

---

## Usage Examples

### Creating a process event
```typescript
await processEventRepository.insert({
  processInstanceId: processId,
  activityInstanceId: null,
  eventType: 'PROCESS_CREATED',
  payload: {
    processDefinitionId: definitionId,
    processDefinitionName: 'customer-onboarding',
    processDefinitionVersion: 1,
    inputPayload: processInput,
    metadata: requestMetadata
  },
  timestamp: new Date(),
  published: false
});
```

### Creating an activity event
```typescript
await processEventRepository.insert({
  processInstanceId: processId,
  activityInstanceId: activityId,
  eventType: 'ACTIVITY_COMPLETED',
  payload: {
    activityName: 'verify-identity',
    activityType: 'API_CALL',
    completedAt: new Date(),
    durationMs: 3000,
    outputData: activityOutput,
    retryCount: 0
  },
  timestamp: new Date(),
  published: false
});
```

### Publishing events externally
```typescript
// Find unpublished events
const unpublishedEvents = await processEventRepository.find({
  where: { published: false },
  order: { timestamp: 'ASC' },
  take: 100
});

// Publish to message broker
for (const event of unpublishedEvents) {
  await messageBroker.publish('process.events', event);
  
  // Mark as published
  await processEventRepository.update(
    { id: event.id },
    { published: true, publishedAt: new Date() }
  );
}
```

---

## Event Publishing Patterns

### 1. Transactional Outbox Pattern
- Events stored in DB within same transaction as state changes
- Background worker publishes events to message broker
- Ensures eventual consistency

### 2. Event Streaming
- Events published to Kafka/RabbitMQ
- External systems subscribe to event topics
- Supports real-time integrations

### 3. Webhook Delivery
- Events trigger HTTP webhooks to registered endpoints
- Retry logic for failed deliveries
- Signature verification for security

---

## Queries

### Get all events for a process
```sql
SELECT * FROM process_event
WHERE process_instance_id = :processId
ORDER BY timestamp ASC;
```

### Get unpublished events
```sql
SELECT * FROM process_event
WHERE published = false
  AND timestamp < NOW() - INTERVAL '1 minute'
ORDER BY timestamp ASC
LIMIT 100;
```

### Count events by type
```sql
SELECT 
  event_type,
  COUNT(*) as event_count
FROM process_event
WHERE timestamp >= NOW() - INTERVAL '7 days'
GROUP BY event_type
ORDER BY event_count DESC;
```

### Find processes with failures
```sql
SELECT DISTINCT process_instance_id
FROM process_event
WHERE event_type IN ('PROCESS_FAILED', 'ACTIVITY_FAILED')
  AND timestamp >= CURRENT_DATE;
```

---

## Observability

### Key Metrics
- Event creation rate by type
- Publishing lag (time between creation and publication)
- Unpublished event count
- Failed publication attempts

### Analytics
```sql
-- Average time between process creation and completion
SELECT 
  AVG(
    EXTRACT(EPOCH FROM (completed.timestamp - created.timestamp))
  ) / 60 as avg_duration_minutes
FROM process_event created
JOIN process_event completed ON created.process_instance_id = completed.process_instance_id
WHERE created.event_type = 'PROCESS_CREATED'
  AND completed.event_type = 'PROCESS_COMPLETED';
```

---

## Data Retention

### Strategy
1. **Hot storage** (PostgreSQL): Last 30 days
2. **Warm storage** (Archive DB): 31-90 days
3. **Cold storage** (S3/Data Lake): Older than 90 days
4. **Compression**: Parquet format for cold storage
5. **Retention**: Indefinite for compliance (may compress/archive)

---

## Migration Strategy

### Phase 1 (MVP)
- Basic event storage
- Simple publishing flag
- Core event types

### Phase 2 (Future)
- Add `version` field for event schema versioning
- Add `causationId` and `correlationId` for event tracing
- Add `aggregateVersion` for event sourcing
- Support for custom event types
- Event replay capability

---

## Related Entities

- **ProcessInstance**: Process that emitted the event
- **ActivityInstance**: Activity that triggered the event (if applicable)
- **ProcessStateHistory**: Related but distinct (history is for audit, events are for integration)
