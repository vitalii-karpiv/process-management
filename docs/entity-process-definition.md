# Entity: ProcessDefinition

## Overview

The **ProcessDefinition** entity serves as a template or blueprint for creating process instances. It defines the structure, flow, and configuration of a business process that can be instantiated multiple times.

---

## Purpose

- Define reusable process templates
- Version process configurations
- Enable process evolution without breaking existing instances
- Provide a single source of truth for process structure

---

## Attributes

### `id` (UUID, Primary Key)
- **Type**: UUID
- **Required**: Yes
- **Description**: Unique identifier for the process definition
- **Generation**: Auto-generated on creation

### `name` (String)
- **Type**: String
- **Required**: Yes
- **Max Length**: 255
- **Description**: Human-readable name of the process
- **Example**: `"customer-onboarding"`, `"order-fulfillment"`
- **Constraints**: Must be unique per version

### `version` (Integer)
- **Type**: Integer
- **Required**: Yes
- **Description**: Version number of this process definition
- **Default**: 1
- **Constraints**: Must be positive, immutable after creation
- **Usage**: Enables schema evolution and rollback capabilities

### `description` (Text)
- **Type**: Text
- **Required**: No
- **Description**: Detailed description of what this process does
- **Example**: `"Orchestrates the complete customer onboarding flow including identity verification, account setup, and welcome email"`

### `schema` (JSON)
- **Type**: JSONB
- **Required**: Yes
- **Description**: Complete process flow definition including activities, transitions, and conditions
- **Structure**:
```json
{
  "activities": [
    {
      "id": "activity-1",
      "name": "verify-identity",
      "type": "API_CALL",
      "next": "activity-2"
    }
  ],
  "transitions": [],
  "errorHandling": {}
}
```

### `isActive` (Boolean)
- **Type**: Boolean
- **Required**: Yes
- **Default**: true
- **Description**: Indicates if this definition can be used to create new process instances
- **Usage**: Allows deprecation without deletion

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

### One-to-Many: ProcessDefinition → ActivityDefinition
- One process definition contains multiple activity definitions
- **Cascade**: Delete activity definitions when process definition is deleted

### One-to-Many: ProcessDefinition → ProcessInstance
- One process definition can spawn multiple process instances
- **Cascade**: None (instances remain even if definition is deactivated)

---

## Constraints

1. **Unique Constraint**: `(name, version)` must be unique
2. **Check Constraint**: `version > 0`
3. **Check Constraint**: `schema IS NOT NULL AND schema != '{}'`
4. **Immutability**: Once created, `name` and `version` cannot be changed

---

## Indexes

```sql
CREATE UNIQUE INDEX idx_process_definition_name_version ON process_definition(name, version);
CREATE INDEX idx_process_definition_is_active ON process_definition(is_active);
CREATE INDEX idx_process_definition_created_at ON process_definition(created_at DESC);
```

---

## Business Rules

1. **Versioning**: Creating a new version requires incrementing the version number
2. **Activation**: Only one version of a process definition should typically be active
3. **Deprecation**: Deactivating a definition prevents new instances but doesn't affect existing ones
4. **Immutability**: Schema changes require creating a new version

---

## Usage Examples

### Creating a new process definition
```typescript
const processDefinition = {
  name: 'customer-onboarding',
  version: 1,
  description: 'Customer onboarding workflow',
  schema: {
    activities: [
      { id: 'verify-identity', type: 'API_CALL', next: 'create-account' },
      { id: 'create-account', type: 'API_CALL', next: 'send-welcome-email' },
      { id: 'send-welcome-email', type: 'EVENT_TRIGGER', next: null }
    ]
  },
  isActive: true
};
```

### Version evolution
```typescript
// Version 2 adds a new activity
const processDefinitionV2 = {
  name: 'customer-onboarding',
  version: 2,
  description: 'Customer onboarding workflow with credit check',
  schema: {
    activities: [
      { id: 'verify-identity', type: 'API_CALL', next: 'credit-check' },
      { id: 'credit-check', type: 'API_CALL', next: 'create-account' },
      { id: 'create-account', type: 'API_CALL', next: 'send-welcome-email' },
      { id: 'send-welcome-email', type: 'EVENT_TRIGGER', next: null }
    ]
  },
  isActive: true
};
```

---

## Migration Strategy

### Phase 1 (MVP)
- Single tenant
- Manual versioning
- Basic schema validation

### Phase 2 (Future)
- Tenant isolation via `tenantId` field
- Automatic schema validation
- Version comparison and diff tools

---

## Related Entities

- **ActivityDefinition**: Contains activity templates for this process
- **ProcessInstance**: Runtime instances created from this definition
