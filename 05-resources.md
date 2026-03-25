# Resources

**Version:** 1.0.0
**Last Updated:** 2026-03-25

The Resource API allows syncing and searching external resources (repositories, channels, documents, etc.) on a per-account basis. Resources are indexed with vector embeddings for semantic search.

---

## Sync Resources

Bulk upsert resources for an account. Resources are matched by `resource_id` within the scope of `account_id + service_id + resource_type`. Existing resources are updated; new ones are created.

```
POST /api/resources/sync
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `account_id` | string | Yes | Account UUID |
| `credential_id` | string | Yes | Credential UUID used to fetch resources |
| `service_id` | string | Yes | Service UUID |
| `resource_type` | string | Yes | Resource type (e.g., `repository`, `channel`, `document`) |
| `resources` | array | Yes | Array of resource objects |

Each resource object:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `resource_id` | string | Yes | External resource identifier |
| `name` | string | Yes | Display name |
| `description` | string | No | Resource description |
| `topics` | array | No | Array of topic/tag strings |
| `metadata` | object | No | Additional metadata |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/resources/sync \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "uuid",
    "credential_id": "uuid",
    "service_id": "uuid",
    "resource_type": "repository",
    "resources": [
      {
        "resource_id": "octocat/hello-world",
        "name": "Hello World",
        "description": "My first repository",
        "topics": ["demo", "hello-world"],
        "metadata": {
          "language": "JavaScript",
          "stars": 42
        }
      }
    ]
  }'
```

**Response:**
```json
{
  "success": true,
  "synced_count": 1,
  "message": "Resources synced successfully"
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 422 | Validation error (missing required fields) |

---

## Search Resources

Semantic search across synced resources for a specific account.

```
POST /api/resources/search
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `query` | string | Yes | Natural language search query |
| `account_id` | string | Yes | Account UUID to search within |
| `service_id` | string | No | Filter to a specific service |
| `resource_type` | string | No | Filter by resource type |
| `limit` | integer | No | Max results (default 10) |
| `min_confidence` | float | No | Minimum similarity threshold |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/resources/search \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "frontend React components",
    "account_id": "uuid",
    "resource_type": "repository",
    "limit": 5
  }'
```

**Response:**
```json
{
  "query": "frontend React components",
  "results": [
    {
      "id": "uuid",
      "service_id": "uuid",
      "resource_type": "repository",
      "resource_id": "myorg/ui-components",
      "name": "UI Components",
      "description": "Shared React component library",
      "topics": ["react", "components", "ui"],
      "metadata": {
        "language": "TypeScript",
        "stars": 15
      },
      "similarity": 0.89
    }
  ]
}
```

---

## List Resources

List synced resources for an account, optionally filtered by service or type.

```
GET /api/resources
```

**Authentication:** Bearer token required

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `account_id` | string | Yes | Account UUID |
| `service_id` | string | No | Filter by service UUID |
| `resource_type` | string | No | Filter by resource type |
| `limit` | integer | No | Max results (default 100) |

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "service_id": "uuid",
      "resource_type": "repository",
      "resource_id": "myorg/ui-components",
      "name": "UI Components",
      "description": "Shared React component library",
      "topics": ["react", "components", "ui"],
      "metadata": {"language": "TypeScript"},
      "synced_at": "2026-03-25T10:00:00Z",
      "has_embedding": true
    }
  ]
}
```

---

## Resource Stats

Get resource counts grouped by service, with embedding generation status.

```
GET /api/resources/stats
```

**Authentication:** Bearer token required

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `account_id` | string | Yes | Account UUID |

**Response:**
```json
{
  "by_service": {
    "github": 150,
    "slack": 45,
    "notion": 30
  },
  "total": 225,
  "pending_embeddings": 12
}
```

---

## Generate Embeddings

Trigger embedding generation for resources that don't have embeddings yet. This is normally handled automatically during sync, but can be triggered manually if needed.

```
POST /api/resources/generate-embeddings
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `account_id` | string | Yes | Account UUID |
| `batch_size` | integer | No | Number of resources to process (default varies) |

**Response:**
```json
{
  "success": true,
  "generated_count": 12
}
```
