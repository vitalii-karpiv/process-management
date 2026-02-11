# Entity: ProcessStateHistory

## Overview

The **ProcessStateHistory** entity provides an immutable audit trail of all state transitions for a process instance. It records every status change with context about why and when the transition occurred.

---

## Purpose

- Maintain complete audit trail for compliance
- Enable debugging and troubleshooting
- Track process lifecycle for analytics
- Support process replay and forensics

---

## Attributes

### `id` (UUID, Primary Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Unique identifier for this history record
- **Generation**: Auto-generated on creation

### `processInstanceId` (UUID, Foreign Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Reference to the process instance
- **Foreign Key**: `process_instance(id)`
- **On Delete**: CASCADE (or archive to separate table)

### `fromStatus` (Enum)
- **Type**: String (Enum)
- **Required**: No (nullable for initial state)
- **Description**: Previous status before transition
- **Allowed Values**: Same as ProcessInstance.status enum
- **Usage**: NULL for the first history entry (CREATED state)

### `toStatus` (Enum)
- **Type**: String (Enum)
- **Required**: Yes
- **Description**: New status after transition
- **Allowed Values**: Same as ProcessInstance.status enum
- **Constraint**: Must follow valid state transition rules

### `reason` (String)
- **Type**: String
- **Required**: Yes
- **Max Length**: 500
- **Description**: Human-readable reason for the state transition
- **Examples**:
  - `"Process started by API request"`
  - `"All activities completed successfully"`
  - `"Activity 'verify-payment' failed after max retries"`
  - `"Process cancelled by user"`
  - `"Waiting for external event 'payment.confirmed'"`

### `triggeredBy` (String)
- **Type**: String
- **Required**: No
- **Max Length**: 255
- **Description**: Identifier of who/what triggered the transition
- **Examples**:
  - `"system"` - Automatic system transition
  - `"user:john@example.com"` - Manual user action
  - `"worker:worker-instance-3"` - Worker service
  - `"api:webhook"` - External webhook
  - `"scheduler:timeout"` - Timeout scheduler

### `metadata` (JSON)
- **Type**: JSONB
- **Required**: No
- **Default**: `{}`
- **Description**: Additional contextual information about the transition
- **Example**:
```json
{
  "activityInstanceId": "uuid-of-activity",
  "activityName": "verify-payment",
  "errorCode": "TIMEOUT",
  "errorDetails": "Payment service did not respond within 30s",
  "retryCount": 3,
  "correlationId": "req_789"
}
```

### `timestamp` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: Yes
- **Description**: When the state transition occurred
- **Generation**: Auto-generated on creation
- **Precision**: Microseconds for ordering

---

## Relationships

### Many-to-One: ProcessStateHistory → ProcessInstance
- Many history records belong to one process instance
- **Foreign Key**: `processInstanceId`
- **Cascade**: History is typically deleted with the process (or archived)

---

## Constraints

1. **Foreign Key Constraint**: `processInstanceId` must exist in `process_instance`
2. **Check Constraint**: `toStatus` must be a valid enum value
3. **Check Constraint**: `fromStatus` must be a valid enum value (if not null)
4. **Check Constraint**: `reason` must not be empty
5. **Immutability**: Records should never be updated after insertion (append-only)

---

## Indexes

```sql
CREATE INDEX idx_process_state_history_process_id ON process_state_history(process_instance_id);
CREATE INDEX idx_process_state_history_timestamp ON process_state_history(timestamp DESC);
CREATE INDEX idx_process_state_history_process_timestamp ON process_state_history(process_instance_id, timestamp DESC);
CREATE INDEX idx_process_state_history_to_status ON process_state_history(to_status);
CREATE INDEX idx_process_state_history_triggered_by ON process_state_history(triggered_by);
```

---

## Business Rules

1. **Immutability**: History records must never be updated or deleted (append-only)
2. **Atomicity**: State transition and history creation must be in same transaction
3. **Completeness**: Every process status change must create a history record
4. **Ordering**: Timestamp must be monotonically increasing for a given process
5. **Validation**: State transitions must follow valid transition rules

---

## Valid State Transitions

```
NULL → CREATED (Initial creation)

CREATED → IN_PROGRESS
CREATED → CANCELLED

IN_PROGRESS → WAITING
IN_PROGRESS → COMPLETED
IN_PROGRESS → FAILED
IN_PROGRESS → CANCELLED

WAITING → IN_PROGRESS
WAITING → FAILED
WAITING → CANCELLED

No transitions from terminal states:
- COMPLETED
- FAILED
- CANCELLED
```

---

## Usage Examples

### Recording initial creation
```typescript
await processStateHistoryRepository.insert({
  processInstanceId: processId,
  fromStatus: null,
  toStatus: 'CREATED',
  reason: 'Process created via API',
  triggeredBy: 'api:POST /processes',
  metadata: {
    correlationId: 'req_123',
    source: 'web-checkout'
  },
  timestamp: new Date()
});
```

### Recording process start
```typescript
await processStateHistoryRepository.insert({
  processInstanceId: processId,
  fromStatus: 'CREATED',
  toStatus: 'IN_PROGRESS',
  reason: 'Process execution started',
  triggeredBy: 'system',
  metadata: {
    firstActivityId: 'uuid-of-first-activity',
    firstActivityName: 'verify-identity'
  },
  timestamp: new Date()
});
```

### Recording failure
```typescript
await processStateHistoryRepository.insert({
  processInstanceId: processId,
  fromStatus: 'IN_PROGRESS',
  toStatus: 'FAILED',
  reason: 'Activity failed after maximum retries',
  triggeredBy: 'worker:worker-instance-2',
  metadata: {
    failedActivityId: 'uuid-of-activity',
    failedActivityName: 'charge-payment',
    errorCode: 'PAYMENT_DECLINED',
    errorMessage: 'Insufficient funds',
    retryCount: 3,
    maxRetries: 3
  },
  timestamp: new Date()
});
```

### Recording manual cancellation
```typescript
await processStateHistoryRepository.insert({
  processInstanceId: processId,
  fromStatus: 'WAITING',
  toStatus: 'CANCELLED',
  reason: 'Process cancelled by administrator',
  triggeredBy: 'user:admin@example.com',
  metadata: {
    cancellationReason: 'Customer request',
    adminNotes: 'Customer changed mind'
  },
  timestamp: new Date()
});
```

---

## Queries

### Get complete process history
```sql
SELECT * FROM process_state_history
WHERE process_instance_id = :processId
ORDER BY timestamp ASC;
```

### Get latest state for a process
```sql
SELECT * FROM process_state_history
WHERE process_instance_id = :processId
ORDER BY timestamp DESC
LIMIT 1;
```

### Find processes that failed today
```sql
SELECT DISTINCT process_instance_id
FROM process_state_history
WHERE to_status = 'FAILED'
  AND timestamp >= CURRENT_DATE;
```

### Calculate state durations
```sql
SELECT 
  process_instance_id,
  to_status,
  EXTRACT(EPOCH FROM (
    LEAD(timestamp) OVER (PARTITION BY process_instance_id ORDER BY timestamp) - timestamp
  )) as duration_seconds
FROM process_state_history
WHERE process_instance_id = :processId;
```

### Count transitions by type
```sql
SELECT 
  from_status,
  to_status,
  COUNT(*) as transition_count
FROM process_state_history
WHERE from_status IS NOT NULL
  AND timestamp >= NOW() - INTERVAL '7 days'
GROUP BY from_status, to_status
ORDER BY transition_count DESC;
```

---

## Observability

### Key Metrics
- Transition frequency by type
- Average time in each state
- Failure rate (transitions to FAILED)
- Cancellation rate (transitions to CANCELLED)
- Manual intervention rate (triggeredBy = user)

### Analytics Examples
```sql
-- Average time spent in each state
SELECT 
  to_status,
  AVG(
    EXTRACT(EPOCH FROM (
      LEAD(timestamp) OVER (PARTITION BY process_instance_id ORDER BY timestamp) - timestamp
    ))
  ) / 60 as avg_minutes_in_state
FROM process_state_history
GROUP BY to_status;

-- Most common failure reasons
SELECT 
  reason,
  COUNT(*) as failure_count
FROM process_state_history
WHERE to_status = 'FAILED'
  AND timestamp >= NOW() - INTERVAL '30 days'
GROUP BY reason
ORDER BY failure_count DESC
LIMIT 10;
```

---

## Data Retention

### Strategy
1. **Hot storage** (PostgreSQL): Last 90 days
2. **Cold storage** (Archive): Older than 90 days
3. **Compression**: Archive as Parquet or similar
4. **Deletion**: Never delete (archive indefinitely for compliance)

### Archival Query
```sql
-- Move old records to archive table
INSERT INTO process_state_history_archive
SELECT * FROM process_state_history
WHERE timestamp < NOW() - INTERVAL '90 days';

-- Then optionally delete from hot storage
DELETE FROM process_state_history
WHERE timestamp < NOW() - INTERVAL '90 days';
```

---

## Migration Strategy

### Phase 1 (MVP)
- Basic state tracking
- Simple reason messages
- Manual triggeredBy tracking

### Phase 2 (Future)
- Add `changeset` field to track specific attribute changes
- Add `userAgent` for API-triggered changes
- Add `ipAddress` for security audit
- Add `previousMetadata` and `newMetadata` for detailed diff
- Support for state branching (parallel processes)

---

## Related Entities

- **ProcessInstance**: The process being tracked
- **ActivityInstance**: Activities that trigger state changes
