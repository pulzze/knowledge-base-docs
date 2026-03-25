# Feedback & Metrics

**Version:** 1.0.0
**Last Updated:** 2026-03-25

The Service Knowledge Base tracks execution outcomes and search quality to continuously improve capability recommendations and reliability scoring.

---

## Search Feedback

Report which capability a user selected from search results. This feedback improves future search ranking.

```
POST /api/capabilities/search/feedback
```

**Authentication:** None

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `search_event_id` | string | Yes | The `search_event_id` from a capability search response (requires `with_feedback: true`) |
| `selected_capability_id` | string | Yes | UUID of the capability the user selected |
| `was_clarified` | boolean | No | Whether the user needed clarification before selecting |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/capabilities/search/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "search_event_id": "uuid-from-search",
    "selected_capability_id": "uuid-of-selected-capability"
  }'
```

**Response:**
```json
{
  "status": "recorded"
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 400 | Missing required parameters |
| 404 | Search event or capability not found |

---

## Execution Outcome Tracking

Report the success or failure of a capability execution. This updates the capability's reliability score.

```
POST /api/feedback/executions
```

**Authentication:** None

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `capability_id` | string | Yes | UUID of the executed capability |
| `service_id` | string | Yes | UUID of the service |
| `success` | boolean | Yes | Whether the execution succeeded |
| `account_id` | string | No | Account UUID for per-account tracking |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/feedback/executions \
  -H "Content-Type: application/json" \
  -d '{
    "capability_id": "uuid",
    "service_id": "uuid",
    "success": true,
    "account_id": "uuid"
  }'
```

**Response:**
```json
{
  "data": {
    "id": "uuid",
    "capability_id": "uuid",
    "success_count": 45,
    "failure_count": 3,
    "reliability_score": 0.94,
    "last_execution_at": "2026-03-25T10:30:00Z"
  }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 400 | Missing required parameters |
| 422 | Invalid capability_id or service_id |

---

## Get Reliability Score

Check the reliability score for a specific capability.

```
GET /api/feedback/reliability/:capability_id
```

**Authentication:** None

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `capability_id` | string | Capability UUID |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `account_id` | string | No | Filter to a specific account's reliability data |

**Response:**
```json
{
  "data": {
    "capability_id": "uuid",
    "reliability_score": 0.94,
    "success_count": 45,
    "failure_count": 3,
    "last_execution_at": "2026-03-25T10:30:00Z"
  }
}
```

---

## Execution Error Reporting

Report execution errors in bulk. Used by the Interactor service to report capability failures for analysis.

```
POST /api/capabilities/execution-errors
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `errors` | array | Yes | Array of error records |

Each error record:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `service_slug` | string | Yes | Service slug |
| `capability_name` | string | Yes | Capability name |
| `total_calls` | integer | Yes | Total calls in the period |
| `failure_count` | integer | Yes | Number of failures |
| `failure_rate` | float | Yes | Failure rate (0.0–1.0) |
| `account_id` | string | No | Account UUID |
| `common_error` | string | No | Most common error message |
| `error_breakdown` | object | No | Error counts by type |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/capabilities/execution-errors \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "errors": [
      {
        "service_slug": "slack",
        "capability_name": "send_message",
        "total_calls": 100,
        "failure_count": 5,
        "failure_rate": 0.05,
        "common_error": "channel_not_found",
        "error_breakdown": {
          "channel_not_found": 3,
          "rate_limited": 2
        }
      }
    ]
  }'
```

**Response:**
```json
{
  "status": "recorded",
  "count": 1,
  "received": 1,
  "skipped": 0
}
```

---

## Metrics Reporting

### Report Credential Stats

Submit credential usage statistics in bulk.

```
POST /api/metrics/credentials
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `stats` | array | Yes | Array of credential stat records |
| `reported_at` | string | No | ISO 8601 timestamp of the reporting period |

Each stat record:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `account_id` | string | Yes | Account UUID |
| `auth_provider` | string | Yes | Auth provider slug |
| `status` | string | Yes | Credential status (`active`, `expired`, `revoked`) |
| `count` | integer | Yes | Number of credentials in this state |

**Response:**
```json
{
  "status": "recorded",
  "count": 3,
  "received": 3
}
```

---

### Report Usage Stats

Submit capability usage statistics in bulk.

```
POST /api/metrics/usage
```

**Authentication:** Bearer token required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `usage` | array | Yes | Array of usage records |
| `period_start` | string | Yes | ISO 8601 start of reporting period |
| `period_end` | string | Yes | ISO 8601 end of reporting period |
| `reported_at` | string | No | ISO 8601 timestamp of submission |

Each usage record:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `account_id` | string | Yes | Account UUID |
| `service_slug` | string | Yes | Service slug |
| `capability_name` | string | Yes | Capability name |
| `total_calls` | integer | Yes | Total calls in the period |
| `success_count` | integer | Yes | Successful calls |
| `failure_count` | integer | Yes | Failed calls |
| `avg_latency_ms` | float | No | Average response time |
| `p95_latency_ms` | float | No | 95th percentile response time |

**Response:**
```json
{
  "status": "recorded",
  "count": 5,
  "received": 5
}
```

---

### Get Credential Summary

Retrieve aggregated credential statistics.

```
GET /api/metrics/credentials
```

**Authentication:** Bearer token required

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `account_id` | string | No | Filter to a specific account |

**Response:**
```json
{
  "status": "ok",
  "stats": [
    {
      "auth_provider": "slack",
      "status": "active",
      "count": 15
    },
    {
      "auth_provider": "google",
      "status": "active",
      "count": 8
    }
  ]
}
```

---

### Get Usage Summary

Retrieve aggregated usage statistics.

```
GET /api/metrics/usage
```

**Authentication:** Bearer token required

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `account_id` | string | No | Filter to a specific account |
| `since` | string | No | ISO 8601 start date |
| `until` | string | No | ISO 8601 end date |

**Response:**
```json
{
  "status": "ok",
  "stats": [
    {
      "service_slug": "slack",
      "capability_name": "send_message",
      "total_calls": 1250,
      "success_count": 1200,
      "failure_count": 50,
      "avg_latency_ms": 120.5
    }
  ]
}
```

---

### Get Platform Summary

Retrieve platform-wide credential and usage statistics.

```
GET /api/metrics/platform
```

**Authentication:** Bearer token required

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `since` | string | No | ISO 8601 start date |
| `until` | string | No | ISO 8601 end date |

**Response:**
```json
{
  "status": "ok",
  "credentials": {
    "total": 150,
    "active": 120,
    "expired": 25,
    "revoked": 5,
    "by_provider": {
      "slack": 45,
      "google": 60,
      "github": 45
    }
  },
  "usage": {
    "total_calls": 50000,
    "success_rate": 0.96,
    "top_services": [...]
  }
}
```
