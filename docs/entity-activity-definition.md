# Entity: ActivityDefinition

## Overview

The **ActivityDefinition** entity defines an atomic unit of work within a process. It serves as a template for creating activity instances and specifies the configuration, type, and behavior of an activity.

---

## Purpose

- Define reusable activity templates within a process
- Specify activity type and execution configuration
- Configure retry policies and timeouts
- Establish activity execution order

---

## Attributes

### `id` (UUID, Primary Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Unique identifier for the activity definition
- **Generation**: Auto-generated on creation

### `processDefinitionId` (UUID, Foreign Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Reference to the parent process definition
- **Foreign Key**: `process_definition(id)`
- **On Delete**: CASCADE

### `name` (String)
- **Type**: String
- **Required**: Yes
- **Max Length**: 255
- **Description**: Human-readable name of the activity
- **Example**: `"verify-identity"`, `"send-notification"`, `"wait-for-approval"`
- **Constraints**: Must be unique within a process definition

### `type` (Enum)
- **Type**: String (Enum)
- **Required**: Yes
- **Description**: Type of activity executor to use
- **Allowed Values**:
  - `API_CALL` - HTTP call to external/internal API
  - `EVENT_TRIGGER` - Publish event or webhook
  - `WAIT` - Pause until event, timeout, or manual action
  - `DECISION` - Conditional branching
  - `TRANSFORMATION` - Data mapping/transformation
  - `MANUAL` - Require human action

### `order` (Integer)
- **Type**: Integer
- **Required**: Yes
- **Description**: Execution sequence number within the process
- **Constraints**: Must be positive
- **Usage**: Determines activity execution order (lower numbers execute first)

### `configuration` (JSON)
- **Type**: JSONB
- **Required**: Yes
- **Description**: Activity-specific configuration based on type
- **Structure**: Varies by activity type (see examples below)

### `retryPolicy` (JSON)
- **Type**: JSONB
- **Required**: No
- **Default**:
```json
{
  "maxRetries": 3,
  "backoffStrategy": "exponential",
  "initialDelayMs": 1000,
  "maxDelayMs": 30000
}
```
- **Description**: Retry configuration for failed activities
- **Fields**:
  - `maxRetries`: Maximum number of retry attempts
  - `backoffStrategy`: `fixed` | `linear` | `exponential`
  - `initialDelayMs`: Initial delay before first retry
  - `maxDelayMs`: Maximum delay between retries

### `timeout` (Integer)
- **Type**: Integer
- **Required**: No
- **Default**: 30000 (30 seconds)
- **Description**: Maximum execution time in milliseconds
- **Constraints**: Must be positive
- **Usage**: Activity fails if execution exceeds this duration

### `createdAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: Yes
- **Description**: When this definition was created
- **Generation**: Auto-generated on creation

### `updatedAt` (Timestamp)
- **Type**: Timestamp with timezone
- **Required**: Yes
- **Description**: When this definition was last modified
- **Generation**: Auto-updated on modification

---

## Relationships

### Many-to-One: ActivityDefinition → ProcessDefinition
- Many activity definitions belong to one process definition
- **Foreign Key**: `processDefinitionId`

### One-to-Many: ActivityDefinition → ActivityInstance
- One activity definition can spawn multiple activity instances
- **Cascade**: None (instances reference the definition)

---

## Constraints

1. **Foreign Key Constraint**: `processDefinitionId` must exist in `process_definition`
2. **Unique Constraint**: `(processDefinitionId, name)` must be unique
3. **Check Constraint**: `order > 0`
4. **Check Constraint**: `timeout > 0`
5. **Enum Constraint**: `type` must be one of allowed values

---

## Indexes

```sql
CREATE INDEX idx_activity_definition_process_id ON activity_definition(process_definition_id);
CREATE UNIQUE INDEX idx_activity_definition_process_name ON activity_definition(process_definition_id, name);
CREATE INDEX idx_activity_definition_order ON activity_definition(process_definition_id, order);
CREATE INDEX idx_activity_definition_type ON activity_definition(type);
```

---

## Configuration Examples by Activity Type

### API_CALL Activity
```json
{
  "url": "https://api.example.com/verify",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer ${env.API_TOKEN}",
    "Content-Type": "application/json"
  },
  "body": {
    "customerId": "${input.customerId}",
    "documentType": "passport"
  },
  "successCondition": "response.status === 'verified'"
}
```

### EVENT_TRIGGER Activity
```json
{
  "eventType": "customer.verified",
  "webhookUrl": "https://webhook.example.com/notifications",
  "payload": {
    "customerId": "${input.customerId}",
    "status": "${output.verificationStatus}",
    "timestamp": "${now()}"
  }
}
```

### WAIT Activity
```json
{
  "waitType": "EXTERNAL_EVENT",
  "eventCondition": {
    "eventType": "payment.received",
    "filter": {
      "orderId": "${input.orderId}"
    }
  },
  "timeoutMs": 86400000,
  "timeoutAction": "FAIL"
}
```

### DECISION Activity
```json
{
  "conditions": [
    {
      "expression": "${output.creditScore >= 700}",
      "nextActivity": "approve-application"
    },
    {
      "expression": "${output.creditScore >= 600}",
      "nextActivity": "manual-review"
    }
  ],
  "defaultActivity": "reject-application"
}
```

### TRANSFORMATION Activity
```json
{
  "mappings": [
    {
      "source": "${output.user.firstName}",
      "target": "customerFirstName"
    },
    {
      "source": "${output.user.lastName}",
      "target": "customerLastName"
    },
    {
      "source": "${concat(output.user.firstName, ' ', output.user.lastName)}",
      "target": "customerFullName"
    }
  ]
}
```

### MANUAL Activity
```json
{
  "assignedTo": "compliance-team",
  "approvalType": "SINGLE",
  "instructions": "Review customer documents and approve or reject the application",
  "actions": [
    {
      "id": "approve",
      "label": "Approve",
      "nextActivity": "complete-onboarding"
    },
    {
      "id": "reject",
      "label": "Reject",
      "nextActivity": "send-rejection-email"
    }
  ]
}
```

---

## Business Rules

1. **Type-Specific Validation**: Configuration must match the activity type
2. **Order Uniqueness**: Order numbers should be unique within a process definition
3. **Timeout Constraints**: Timeout should be reasonable for the activity type
4. **Retry Limits**: MaxRetries should not exceed system limits

---

## Usage Example

```typescript
const activityDefinition = {
  processDefinitionId: 'uuid-of-process-definition',
  name: 'verify-customer-identity',
  type: 'API_CALL',
  order: 1,
  configuration: {
    url: 'https://api.verification-service.com/verify',
    method: 'POST',
    headers: {
      'Authorization': 'Bearer ${env.VERIFICATION_API_KEY}'
    },
    body: {
      customerId: '${input.customerId}',
      documentType: 'passport'
    }
  },
  retryPolicy: {
    maxRetries: 3,
    backoffStrategy: 'exponential',
    initialDelayMs: 1000,
    maxDelayMs: 30000
  },
  timeout: 60000
};
```

---

## Migration Strategy

### Phase 1 (MVP)
- Support all 6 activity types
- Basic configuration validation
- Simple retry policies

### Phase 2 (Future)
- Advanced conditional logic
- Parallel activity execution
- Sub-process activities
- Custom activity types

---

## Related Entities

- **ProcessDefinition**: Parent process template
- **ActivityInstance**: Runtime instances created from this definition
