# Discovery

**Version:** 1.0.0
**Last Updated:** 2026-03-25

The Service Knowledge Base can automatically discover and catalog external APIs. When a service is not found in the catalog, the discovery system analyzes the service's documentation, OpenAPI specs, and website to extract capabilities, auth requirements, and API schemas.

---

## Service Lookup with Auto-Discovery

Look up a service by identifier (URL, name, or slug). If the service is not found and the identifier looks like a URL, a discovery job is automatically queued.

```
POST /api/services/lookup
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `identifier` | string | Yes | — | Service URL, name, or slug |
| `wait_for_discovery` | boolean | No | false | If true, wait for discovery to complete (up to timeout) |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/services/lookup \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "identifier": "https://api.notion.com",
    "wait_for_discovery": false
  }'
```

### Response: Service Found (200)

```json
{
  "status": "found",
  "service": {
    "id": "uuid",
    "name": "Notion",
    "slug": "notion",
    "description": "All-in-one workspace for notes, docs, and project management",
    ...
  }
}
```

### Response: Discovery Initiated (202)

When the service is not found and a discovery job is queued:

```json
{
  "status": "discovery_initiated",
  "discovery_id": "uuid",
  "message": "Service not found. Discovery has been queued."
}
```

Use the `discovery_id` to poll for status (see below).

**Errors:**

| Status | Description |
|--------|-------------|
| 400 | Missing `identifier` parameter |

---

## Check Discovery Status

Poll the status of a discovery job.

```
GET /api/discovery/:id/status
```

**Authentication:** Bearer token required

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | Discovery job UUID (from lookup response) |

**Response:**
```json
{
  "id": "uuid",
  "service_identifier": "https://api.notion.com",
  "status": "completed",
  "attempts": 1,
  "max_attempts": 3,
  "confidence_score": 0.85,
  "result_service_id": "uuid",
  "error_message": null,
  "created_at": "2026-03-25T10:00:00Z",
  "updated_at": "2026-03-25T10:02:30Z"
}
```

**Discovery Statuses:**

| Status | Description |
|--------|-------------|
| `pending` | Queued, not yet started |
| `in_progress` | Currently being processed |
| `completed` | Successfully discovered and cataloged |
| `failed` | Discovery failed after all attempts |

When `status` is `completed`, the `result_service_id` contains the UUID of the newly cataloged service.

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Discovery job not found |

---

## Discovery Flow

The typical integration pattern for service lookup with discovery:

```
1. POST /api/services/lookup
   ├── 200 {"status": "found"} → Use the service
   └── 202 {"status": "discovery_initiated", "discovery_id": "..."}
       └── Poll: GET /api/discovery/:id/status
           ├── "pending" or "in_progress" → Wait and retry
           ├── "completed" → GET /api/services/:result_service_id
           └── "failed" → Handle error
```

### Example: Lookup with Polling

```bash
# Step 1: Lookup
RESPONSE=$(curl -s -X POST https://kb.interactor.com/api/services/lookup \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"identifier": "https://api.notion.com"}')

STATUS=$(echo $RESPONSE | jq -r '.status')

if [ "$STATUS" = "found" ]; then
  echo "Service found!"
  echo $RESPONSE | jq '.service'
elif [ "$STATUS" = "discovery_initiated" ]; then
  DISCOVERY_ID=$(echo $RESPONSE | jq -r '.discovery_id')
  echo "Discovery started: $DISCOVERY_ID"

  # Step 2: Poll for completion
  while true; do
    DISCOVERY=$(curl -s "https://kb.interactor.com/api/discovery/$DISCOVERY_ID/status" \
      -H "Authorization: Bearer <token>")
    DISC_STATUS=$(echo $DISCOVERY | jq -r '.status')

    case $DISC_STATUS in
      "completed")
        SERVICE_ID=$(echo $DISCOVERY | jq -r '.result_service_id')
        echo "Discovery complete! Service ID: $SERVICE_ID"
        break
        ;;
      "failed")
        echo "Discovery failed: $(echo $DISCOVERY | jq -r '.error_message')"
        break
        ;;
      *)
        echo "Status: $DISC_STATUS (attempt $(echo $DISCOVERY | jq '.attempts')/$(echo $DISCOVERY | jq '.max_attempts'))"
        sleep 5
        ;;
    esac
  done
fi
```
