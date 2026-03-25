# Admin API

**Version:** 1.0.0
**Last Updated:** 2026-03-25

The Admin API provides service management, capability import, discovery queue management, AI-powered refinements, and change tracking. All admin endpoints require JWT authentication.

---

## Service Management

### List Services (Admin)

List all services with full admin details, including draft and archived services.

```
GET /api/admin/services
```

**Authentication:** Bearer token required

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `status` | string | — | Filter by status (`active`, `draft`, `archived`) |
| `verification_status` | string | — | Filter by verification (`verified`, `pending`, `rejected`) |
| `search` | string | — | Text search |
| `limit` | integer | 20 | Results per page |
| `offset` | integer | 0 | Pagination offset |

**Response:**
```json
{
  "data": [...],
  "meta": {
    "total": 42,
    "limit": 20,
    "offset": 0
  }
}
```

---

### Get Service (Admin)

```
GET /api/admin/services/:id
```

**Authentication:** Bearer token required

**Response:**
```json
{
  "data": {
    "id": "uuid",
    "name": "Slack",
    "slug": "slack",
    ...full service detail...
  }
}
```

---

### Create Service

```
POST /api/admin/services
```

**Authentication:** Bearer token required

**Request Body:**
```json
{
  "service": {
    "name": "Slack",
    "slug": "slack",
    "description": "Team messaging and collaboration platform",
    "website_url": "https://slack.com",
    "api_base_url": "https://slack.com/api",
    "developer_docs_url": "https://api.slack.com/docs",
    "category_id": "uuid",
    "status": "draft"
  }
}
```

**Response (201):**
```json
{
  "data": { ...created service... }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 422 | Validation error (missing required fields, duplicate slug) |

---

### Update Service

```
PATCH /api/admin/services/:id
```

**Authentication:** Bearer token required

**Request Body:**
```json
{
  "service": {
    "description": "Updated description",
    "status": "active"
  }
}
```

**Response:**
```json
{
  "data": { ...updated service... }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Service not found |
| 422 | Validation error |

---

### Delete Service

```
DELETE /api/admin/services/:id
```

**Authentication:** Bearer token required

**Response:** `204 No Content`

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Service not found |

---

### Verify Service

Mark a service as verified after review.

```
POST /api/admin/services/:id/verify
```

**Authentication:** Bearer token required

**Response:**
```json
{
  "data": { ...service with verification_status: "verified"... },
  "message": "Service verified"
}
```

---

### Reject Service

Mark a service as rejected with an optional reason.

```
POST /api/admin/services/:id/reject
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | No | Rejection reason |

**Response:**
```json
{
  "data": { ...service with verification_status: "rejected"... },
  "message": "Service rejected"
}
```

---

## Capability Management

### Update Capability

Update a capability's metadata.

```
PATCH /api/admin/capabilities/:id
```

**Authentication:** Bearer token required

**Request Body:**
```json
{
  "capability": {
    "display_name": "Send Slack Message",
    "description": "Updated description",
    "category": "messaging",
    "is_destructive": false,
    "requires_confirmation": false
  }
}
```

**Response:**
```json
{
  "data": { ...updated capability... },
  "message": "Capability updated successfully"
}
```

---

### Import Capabilities from OpenAPI Spec

Import capabilities from an OpenAPI/Swagger specification. Creates or updates capabilities and auto-detects auth providers.

```
POST /api/admin/services/:service_id/capabilities/import
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec` | object/string | Yes | OpenAPI spec (JSON object or YAML string) |
| `mode` | string | No | Import mode: `merge` (default) or `replace` |
| `base_path` | string | No | Override API base path |
| `include_auth` | boolean | No | Auto-detect and import auth providers (default true) |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/admin/services/:service_id/capabilities/import \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "spec": { ...openapi spec... },
    "mode": "merge",
    "include_auth": true
  }'
```

**Response (201):**
```json
{
  "data": [ ...imported capabilities... ],
  "message": "Imported 15 capabilities (12 created, 3 updated)",
  "meta": {
    "created": 12,
    "updated": 3,
    "total": 15
  },
  "auth": {
    "providers": 1,
    "configs": 1
  }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 400 | Invalid or unparseable spec |
| 404 | Service not found |
| 422 | Spec validation failed |

---

## Discovery Queue

### List Queue

```
GET /api/admin/discovery/queue
```

**Authentication:** Bearer token required

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "service_identifier": "https://api.notion.com",
      "status": "pending",
      "priority": 5,
      "attempts": 0,
      "max_attempts": 3,
      "confidence_score": null,
      "result_service_id": null,
      "error_message": null,
      "created_at": "2026-03-25T10:00:00Z",
      "updated_at": "2026-03-25T10:00:00Z"
    }
  ]
}
```

---

### Queue Stats

```
GET /api/admin/discovery/stats
```

**Authentication:** Bearer token required

**Response:**
```json
{
  "pending": 5,
  "in_progress": 2,
  "completed": 150,
  "failed": 8
}
```

---

### Queue New Discovery

```
POST /api/admin/discovery
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `service_identifier` | string | Yes | URL or name to discover |
| `priority` | integer | No | Queue priority (higher = sooner) |

**Response (201):**
```json
{
  "data": { ...queue item... }
}
```

---

### Rediscover Service

Re-run discovery for an existing service to update its capabilities.

```
POST /api/admin/discovery/rediscover/:service_id
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | No | LLM model to use for discovery |
| `merge` | boolean | No | Merge with existing capabilities (default true) |

**Response:**
```json
{
  "success": true,
  "service": { ...updated service... },
  "confidence": 0.87,
  "merged": true,
  "capabilities_count": 25,
  "auth_configs_count": 2
}
```

---

## Refinements

AI-powered targeted updates to service information. Refinements are generated by an AI agent and require admin approval before being applied.

### Create Refinement

Request an AI-powered refinement for a service.

```
POST /api/admin/services/:service_id/refine
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `task` | string | Yes | Refinement task type (e.g., `update_capabilities`, `fix_auth`, `improve_descriptions`) |
| `target` | string | No | Specific target within the service (e.g., capability name) |
| `hints` | string | No | Additional context for the AI agent |

**Response (201):**
```json
{
  "id": "uuid",
  "service_id": "uuid",
  "task": "update_capabilities",
  "status": "pending",
  "proposed_changes": { ... },
  "created_at": "2026-03-25T10:00:00Z"
}
```

---

### List Refinements

```
GET /api/admin/services/:service_id/refinements
```

**Authentication:** Bearer token required

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status (`pending`, `applied`, `rejected`) |
| `task` | string | Filter by task type |
| `limit` | integer | Results per page |
| `offset` | integer | Pagination offset |

---

### Get Refinement

```
GET /api/admin/refinements/:id
```

**Authentication:** Bearer token required

---

### Apply Refinement

Apply a pending refinement to the service.

```
POST /api/admin/refinements/:id/apply
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `applied_data` | object | No | Override proposed changes before applying |
| `notes` | string | No | Admin notes on the application |

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Refinement not found |
| 409 | Refinement already applied or rejected |

---

### Reject Refinement

Reject a pending refinement.

```
POST /api/admin/refinements/:id/reject
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `notes` | string | No | Reason for rejection |

---

## Change History

View the change history for a service.

```
GET /api/admin/services/:service_id/history
```

**Authentication:** Bearer token required

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `change_type` | string | Filter by change type |
| `limit` | integer | Max results (default 50) |

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "change_type": "capability_updated",
      "changed_by": "admin@example.com",
      "previous_value": { ... },
      "new_value": { ... },
      "change_reason": "Updated parameter descriptions",
      "created_at": "2026-03-25T10:30:00Z"
    }
  ]
}
```

---

## Dashboard

### Dashboard Stats

Overview statistics for the admin dashboard.

```
GET /api/admin/dashboard/stats
```

**Authentication:** Bearer token required

**Response:**
```json
{
  "services": {
    "total": 42,
    "verified": 35,
    "pending_verification": 5,
    "active": 38
  },
  "discovery_queue": {
    "pending": 3,
    "in_progress": 1,
    "completed_today": 12,
    "failed_today": 0
  }
}
```
