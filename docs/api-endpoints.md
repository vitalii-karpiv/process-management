# API Endpoints Specification

## Overview

This document defines all REST API endpoints for the **Process Management Platform** MVP. The API follows RESTful principles and uses JSON for all request/response payloads.

---

## Base URL

```
Development: http://localhost:3000/api/v1
Production: https://api.process-platform.com/v1
```

---

## Authentication

**Phase 1 (MVP)**: Basic API Key authentication
- Header: `X-API-Key: <api-key>`

**Phase 2**: Bearer token (JWT) with tenant isolation

---

## Common Response Patterns

### Success Response
```json
{
  "data": { ... },
  "meta": {
    "timestamp": "2026-02-11T10:00:00Z"
  }
}
```

### Error Response
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid process definition",
    "details": [
      {
        "field": "name",
        "message": "Name is required"
      }
    ]
  },
  "meta": {
    "timestamp": "2026-02-11T10:00:00Z",
    "requestId": "req_123"
  }
}
```

### Pagination
```json
{
  "data": [...],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 150,
    "totalPages": 8
  }
}
```

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Request validation failed |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Resource conflict (duplicate) |
| `PROCESS_NOT_FOUND` | 404 | Process instance not found |
| `INVALID_STATE_TRANSITION` | 400 | Invalid process state transition |
| `UNAUTHORIZED` | 401 | Missing or invalid authentication |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `INTERNAL_ERROR` | 500 | Internal server error |

---

# Endpoint Groups

## 1. Process Definitions

### 1.1 Create Process Definition

**Endpoint**: `POST /process-definitions`

**Description**: Create a new process definition template

**Request Body**:
```json
{
  "name": "customer-onboarding",
  "version": 1,
  "description": "Complete customer onboarding workflow",
  "schema": {
    "activities": [
      {
        "id": "verify-identity",
        "name": "Verify Customer Identity",
        "type": "API_CALL",
        "order": 1,
        "configuration": {
          "url": "https://api.identity-service.com/verify",
          "method": "POST"
        },
        "retryPolicy": {
          "maxRetries": 3,
          "backoffStrategy": "exponential"
        },
        "timeout": 30000
      }
    ]
  },
  "isActive": true
}
```

**Response**: `201 Created`
```json
{
  "data": {
    "id": "def_550e8400-e29b-41d4-a716-446655440000",
    "name": "customer-onboarding",
    "version": 1,
    "description": "Complete customer onboarding workflow",
    "schema": { ... },
    "isActive": true,
    "createdAt": "2026-02-11T10:00:00Z",
    "updatedAt": "2026-02-11T10:00:00Z"
  }
}
```

**Errors**:
- `400`: Validation error (invalid schema)
- `409`: Definition with same name and version already exists

---

### 1.2 Get Process Definition

**Endpoint**: `GET /process-definitions/{id}`

**Description**: Retrieve a specific process definition by ID

**Path Parameters**:
- `id` (UUID): Process definition ID

**Response**: `200 OK`
```json
{
  "data": {
    "id": "def_550e8400-e29b-41d4-a716-446655440000",
    "name": "customer-onboarding",
    "version": 1,
    "description": "Complete customer onboarding workflow",
    "schema": { ... },
    "isActive": true,
    "createdAt": "2026-02-11T10:00:00Z",
    "updatedAt": "2026-02-11T10:00:00Z"
  }
}
```

**Errors**:
- `404`: Process definition not found

---

### 1.3 List Process Definitions

**Endpoint**: `GET /process-definitions`

**Description**: List all process definitions with optional filtering

**Query Parameters**:
- `name` (string, optional): Filter by name
- `isActive` (boolean, optional): Filter by active status
- `page` (integer, optional, default: 1): Page number
- `pageSize` (integer, optional, default: 20): Items per page

**Response**: `200 OK`
```json
{
  "data": [
    {
      "id": "def_550e8400-e29b-41d4-a716-446655440000",
      "name": "customer-onboarding",
      "version": 1,
      "description": "Complete customer onboarding workflow",
      "isActive": true,
      "createdAt": "2026-02-11T10:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 45,
    "totalPages": 3
  }
}
```

---

### 1.4 Update Process Definition Status

**Endpoint**: `PATCH /process-definitions/{id}`

**Description**: Update process definition (primarily for activating/deactivating)

**Path Parameters**:
- `id` (UUID): Process definition ID

**Request Body**:
```json
{
  "isActive": false
}
```

**Response**: `200 OK`
```json
{
  "data": {
    "id": "def_550e8400-e29b-41d4-a716-446655440000",
    "name": "customer-onboarding",
    "version": 1,
    "isActive": false,
    "updatedAt": "2026-02-11T10:05:00Z"
  }
}
```

**Errors**:
- `404`: Process definition not found

---

## 2. Process Instances

### 2.1 Create Process Instance

**Endpoint**: `POST /process-instances`

**Description**: Create and optionally start a new process instance

**Request Body**:
```json
{
  "processDefinitionId": "def_550e8400-e29b-41d4-a716-446655440000",
  "inputPayload": {
    "customerId": "cust_123",
    "email": "customer@example.com",
    "orderItems": [
      { "sku": "ITEM-001", "quantity": 2 }
    ]
  },
  "metadata": {
    "correlationId": "web_request_789",
    "source": "web-checkout",
    "priority": "high"
  },
  "autoStart": true
}
```

**Response**: `201 Created`
```json
{
  "data": {
    "id": "proc_660e8400-e29b-41d4-a716-446655440000",
    "processDefinitionId": "def_550e8400-e29b-41d4-a716-446655440000",
    "status": "IN_PROGRESS",
    "inputPayload": { ... },
    "metadata": { ... },
    "currentActivityInstanceId": "act_770e8400-e29b-41d4-a716-446655440000",
    "version": 1,
    "createdAt": "2026-02-11T10:00:00Z",
    "startedAt": "2026-02-11T10:00:00Z"
  }
}
```

**Errors**:
- `400`: Invalid input payload
- `404`: Process definition not found or inactive

---

### 2.2 Get Process Instance

**Endpoint**: `GET /process-instances/{id}`

**Description**: Retrieve a specific process instance with current state

**Path Parameters**:
- `id` (UUID): Process instance ID

**Response**: `200 OK`
```json
{
  "data": {
    "id": "proc_660e8400-e29b-41d4-a716-446655440000",
    "processDefinitionId": "def_550e8400-e29b-41d4-a716-446655440000",
    "processDefinitionName": "customer-onboarding",
    "status": "IN_PROGRESS",
    "inputPayload": { ... },
    "outputPayload": null,
    "metadata": { ... },
    "currentActivityInstanceId": "act_770e8400-e29b-41d4-a716-446655440000",
    "currentActivityName": "verify-identity",
    "version": 3,
    "createdAt": "2026-02-11T10:00:00Z",
    "startedAt": "2026-02-11T10:00:00Z",
    "updatedAt": "2026-02-11T10:01:30Z",
    "completedAt": null
  }
}
```

**Errors**:
- `404`: Process instance not found

---

### 2.3 List Process Instances

**Endpoint**: `GET /process-instances`

**Description**: List process instances with filtering and pagination

**Query Parameters**:
- `status` (string, optional): Filter by status (CREATED, IN_PROGRESS, WAITING, COMPLETED, FAILED, CANCELLED)
- `processDefinitionId` (UUID, optional): Filter by definition
- `createdAfter` (ISO 8601, optional): Filter by creation date
- `createdBefore` (ISO 8601, optional): Filter by creation date
- `page` (integer, optional, default: 1)
- `pageSize` (integer, optional, default: 20)
- `sortBy` (string, optional, default: "createdAt"): Sort field
- `sortOrder` (string, optional, default: "desc"): "asc" or "desc"

**Response**: `200 OK`
```json
{
  "data": [
    {
      "id": "proc_660e8400-e29b-41d4-a716-446655440000",
      "processDefinitionName": "customer-onboarding",
      "status": "IN_PROGRESS",
      "currentActivityName": "verify-identity",
      "createdAt": "2026-02-11T10:00:00Z",
      "startedAt": "2026-02-11T10:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 1523,
    "totalPages": 77
  }
}
```

---

### 2.4 Cancel Process Instance

**Endpoint**: `POST /process-instances/{id}/cancel`

**Description**: Cancel a running or waiting process

**Path Parameters**:
- `id` (UUID): Process instance ID

**Request Body**:
```json
{
  "reason": "Customer requested cancellation",
  "cancelledBy": "user:admin@example.com"
}
```

**Response**: `200 OK`
```json
{
  "data": {
    "id": "proc_660e8400-e29b-41d4-a716-446655440000",
    "status": "CANCELLED",
    "updatedAt": "2026-02-11T10:10:00Z"
  }
}
```

**Errors**:
- `404`: Process not found
- `400`: Cannot cancel process in terminal state (COMPLETED, FAILED, CANCELLED)

---

### 2.5 Get Process Instance History

**Endpoint**: `GET /process-instances/{id}/history`

**Description**: Retrieve complete state transition history for a process

**Path Parameters**:
- `id` (UUID): Process instance ID

**Response**: `200 OK`
```json
{
  "data": [
    {
      "id": "hist_001",
      "fromStatus": null,
      "toStatus": "CREATED",
      "reason": "Process created via API",
      "triggeredBy": "api:POST /process-instances",
      "metadata": {},
      "timestamp": "2026-02-11T10:00:00Z"
    },
    {
      "id": "hist_002",
      "fromStatus": "CREATED",
      "toStatus": "IN_PROGRESS",
      "reason": "Process execution started",
      "triggeredBy": "system",
      "metadata": {
        "firstActivityName": "verify-identity"
      },
      "timestamp": "2026-02-11T10:00:00Z"
    }
  ]
}
```

**Errors**:
- `404`: Process not found

---

## 3. Activity Instances

### 3.1 Get Activity Instance

**Endpoint**: `GET /activity-instances/{id}`

**Description**: Retrieve a specific activity instance

**Path Parameters**:
- `id` (UUID): Activity instance ID

**Response**: `200 OK`
```json
{
  "data": {
    "id": "act_770e8400-e29b-41d4-a716-446655440000",
    "processInstanceId": "proc_660e8400-e29b-41d4-a716-446655440000",
    "activityDefinitionId": "actdef_880e8400-e29b-41d4-a716-446655440000",
    "name": "verify-identity",
    "type": "API_CALL",
    "status": "RUNNING",
    "inputData": { ... },
    "outputData": null,
    "retryCount": 0,
    "maxRetries": 3,
    "createdAt": "2026-02-11T10:00:05Z",
    "startedAt": "2026-02-11T10:00:05Z",
    "completedAt": null
  }
}
```

**Errors**:
- `404`: Activity not found

---

### 3.2 List Activities for Process

**Endpoint**: `GET /process-instances/{processInstanceId}/activity-instances`

**Description**: List all activities for a specific process instance

**Path Parameters**:
- `processInstanceId` (UUID): Process instance ID

**Query Parameters**:
- `status` (string, optional): Filter by status

**Response**: `200 OK`
```json
{
  "data": [
    {
      "id": "act_770e8400-e29b-41d4-a716-446655440000",
      "name": "verify-identity",
      "type": "API_CALL",
      "status": "COMPLETED",
      "retryCount": 0,
      "createdAt": "2026-02-11T10:00:05Z",
      "completedAt": "2026-02-11T10:00:08Z"
    },
    {
      "id": "act_880e8400-e29b-41d4-a716-446655440000",
      "name": "create-account",
      "type": "API_CALL",
      "status": "RUNNING",
      "retryCount": 1,
      "createdAt": "2026-02-11T10:00:08Z",
      "startedAt": "2026-02-11T10:00:10Z"
    }
  ]
}
```

**Errors**:
- `404`: Process not found

---

### 3.3 Complete Manual Activity

**Endpoint**: `POST /activity-instances/{id}/complete`

**Description**: Manually complete a waiting activity (for MANUAL or WAIT activity types)

**Path Parameters**:
- `id` (UUID): Activity instance ID

**Request Body**:
```json
{
  "outputData": {
    "approved": true,
    "approver": "john.doe@example.com",
    "notes": "Application approved"
  },
  "completedBy": "user:john.doe@example.com"
}
```

**Response**: `200 OK`
```json
{
  "data": {
    "id": "act_770e8400-e29b-41d4-a716-446655440000",
    "status": "COMPLETED",
    "outputData": { ... },
    "completedAt": "2026-02-11T10:15:00Z"
  }
}
```

**Errors**:
- `404`: Activity not found
- `400`: Activity is not in WAITING state or not a manual activity

---

## 4. Process Events

### 4.1 Get Process Events

**Endpoint**: `GET /process-instances/{processInstanceId}/events`

**Description**: Retrieve all events for a specific process

**Path Parameters**:
- `processInstanceId` (UUID): Process instance ID

**Query Parameters**:
- `eventType` (string, optional): Filter by event type
- `limit` (integer, optional, default: 100): Max events to return

**Response**: `200 OK`
```json
{
  "data": [
    {
      "id": "evt_001",
      "eventType": "PROCESS_CREATED",
      "payload": {
        "processDefinitionName": "customer-onboarding",
        "inputPayload": { ... }
      },
      "timestamp": "2026-02-11T10:00:00Z"
    },
    {
      "id": "evt_002",
      "eventType": "ACTIVITY_STARTED",
      "payload": {
        "activityName": "verify-identity",
        "activityType": "API_CALL"
      },
      "timestamp": "2026-02-11T10:00:05Z"
    }
  ]
}
```

**Errors**:
- `404`: Process not found

---

### 4.2 Ingest External Event

**Endpoint**: `POST /events/external`

**Description**: Receive external events to resume waiting processes

**Request Body**:
```json
{
  "eventType": "payment.confirmed",
  "payload": {
    "orderId": "order_123",
    "transactionId": "txn_456",
    "amount": 99.99
  },
  "correlationId": "order_123"
}
```

**Response**: `202 Accepted`
```json
{
  "data": {
    "eventId": "ext_evt_001",
    "status": "ACCEPTED",
    "message": "Event queued for processing"
  }
}
```

**Note**: This is an asynchronous operation. The event is queued and processed by workers.

---

## 5. Health & Monitoring

### 5.1 Health Check

**Endpoint**: `GET /health`

**Description**: Check API service health

**Response**: `200 OK`
```json
{
  "status": "healthy",
  "timestamp": "2026-02-11T10:00:00Z",
  "version": "1.0.0",
  "services": {
    "database": "healthy",
    "messageQueue": "healthy",
    "redis": "healthy"
  }
}
```

---

### 5.2 Readiness Check

**Endpoint**: `GET /health/ready`

**Description**: Check if service is ready to accept traffic

**Response**: `200 OK` or `503 Service Unavailable`

---

### 5.3 Get Metrics

**Endpoint**: `GET /metrics`

**Description**: Retrieve platform metrics (Prometheus format)

**Response**: `200 OK`
```
# TYPE process_instances_total counter
process_instances_total{status="completed"} 1523
process_instances_total{status="failed"} 45
process_instances_total{status="in_progress"} 234

# TYPE process_duration_seconds histogram
process_duration_seconds_bucket{le="10"} 450
process_duration_seconds_bucket{le="60"} 1200
```

---

## Rate Limiting

**MVP**: 1000 requests per minute per API key

**Headers**:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1644576000
```

**Response** (when exceeded): `429 Too Many Requests`

---

## Versioning Strategy

API version is included in the URL path (`/v1/`).

**Breaking changes** require a new version (v2, v3, etc.)

**Non-breaking changes** can be added to existing version:
- Adding new optional fields
- Adding new endpoints
- Adding new query parameters

---

## Idempotency

### Idempotent Endpoints

The following endpoints support idempotency keys:

- `POST /process-instances`
- `POST /process-instances/{id}/cancel`
- `POST /activity-instances/{id}/complete`

**Header**: `Idempotency-Key: <unique-key>`

**Example**:
```bash
curl -X POST https://api.example.com/v1/process-instances \
  -H "X-API-Key: key_123" \
  -H "Idempotency-Key: req_unique_123" \
  -d '{ ... }'
```

If the same key is used within 24 hours, the cached response is returned.

---

## WebHooks (Phase 2)

**Future**: Support for registering webhook URLs to receive process events

**Example**:
```
POST /webhooks
{
  "url": "https://customer.com/webhook",
  "events": ["PROCESS_COMPLETED", "PROCESS_FAILED"],
  "secret": "whsec_123"
}
```

---

## Summary

### Endpoint Count by Category

| Category | Endpoints | Methods |
|----------|-----------|---------|
| Process Definitions | 4 | GET, POST, PATCH |
| Process Instances | 5 | GET, POST |
| Activity Instances | 3 | GET, POST |
| Events | 2 | GET, POST |
| Health | 3 | GET |
| **Total** | **17** | - |

### MVP Priority

**P0 (Critical)**:
- Create/Get Process Definition
- Create/Get Process Instance
- List Processes
- Get Activities for Process
- Complete Manual Activity
- Health Check

**P1 (Important)**:
- List Process Definitions
- Cancel Process
- Get Process History
- Get Process Events
- External Event Ingestion

**P2 (Nice to Have)**:
- Update Process Definition
- Get Activity Instance
- Metrics Endpoint

---

## OpenAPI Specification

A full OpenAPI 3.0 specification will be generated from this document in `openapi.yaml`.

---

## Related Documentation

- Entity definitions: `entity-*.md`
- Business requirements: `business-requirements.md`
- Technical requirements: `technical-requirements.md`
