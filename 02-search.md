# Search

**Version:** 1.0.0
**Last Updated:** 2026-03-25

The Service Knowledge Base provides semantic search powered by vector embeddings. Queries are matched against service descriptions, capability signatures, and auth provider metadata using cosine similarity.

---

## Search Services

Find services matching a natural language query.

```
POST /api/services/search
```

**Authentication:** None

**Request Body:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `query` | string | Yes | — | Natural language search query |
| `limit` | integer | No | 10 | Max results to return |
| `min_confidence` | float | No | 0.3 | Minimum similarity threshold (0.0–1.0) |
| `filters` | object | No | — | Additional filters (category, status) |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/services/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "send email notifications",
    "limit": 5,
    "min_confidence": 0.5
  }'
```

**Response:**
```json
{
  "query": "send email notifications",
  "results": [
    {
      "id": "uuid",
      "name": "SendGrid",
      "slug": "sendgrid",
      "description": "Email delivery and marketing platform",
      "confidence": 0.89,
      "logo_url": "https://example.com/sendgrid-logo.png",
      "website_url": "https://sendgrid.com",
      "status": "active",
      "verification_status": "verified"
    },
    {
      "id": "uuid",
      "name": "Mailgun",
      "slug": "mailgun",
      "description": "Transactional email API service",
      "confidence": 0.82,
      "logo_url": null,
      "website_url": "https://mailgun.com",
      "status": "active",
      "verification_status": "verified"
    }
  ],
  "meta": {
    "count": 2,
    "min_confidence": 0.5
  }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 400 | Missing `query` parameter |

---

## Search Capabilities

Find specific API capabilities matching a natural language query. Optionally scoped to a single service.

```
POST /api/capabilities/search
```

**Authentication:** None

**Request Body:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `query` | string | Yes | — | Natural language search query |
| `limit` | integer | No | 10 | Max results to return |
| `min_confidence` | float | No | 0.3 | Minimum similarity threshold (0.0–1.0) |
| `service_id` | string | No | — | Scope search to a specific service UUID |
| `filters` | object | No | — | Additional filters |
| `with_feedback` | boolean | No | false | Include feedback tracking (returns `search_event_id`) |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/capabilities/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "create a new Jira ticket",
    "limit": 5,
    "with_feedback": true
  }'
```

**Response:**
```json
{
  "search_event_id": "uuid",
  "query": "create a new Jira ticket",
  "results": [
    {
      "name": "create_issue",
      "signature": "create_issue(project_key: string, summary: string, issue_type?: string) -> issue",
      "description": "Create a new issue in a Jira project",
      "service_id": "uuid",
      "service_name": "Jira",
      "confidence": 0.94
    },
    {
      "name": "create_task",
      "signature": "create_task(project: string, name: string, description?: string) -> task",
      "description": "Create a task in Asana",
      "service_id": "uuid",
      "service_name": "Asana",
      "confidence": 0.72
    }
  ],
  "meta": {
    "count": 2,
    "min_confidence": 0.3,
    "feedback_enabled": true
  }
}
```

When `with_feedback` is `true`, the response includes a `search_event_id` that can be used with the [Search Feedback](04-feedback-and-metrics.md#search-feedback) endpoint to report which result the user selected.

**Errors:**

| Status | Description |
|--------|-------------|
| 400 | Missing `query` parameter |

---

## Search Auth Providers

Find auth providers matching a query. Useful for discovering which OAuth provider to use for a given service.

```
POST /api/auth-providers/search
```

**Authentication:** None

**Request Body:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `query` | string | Yes | — | Natural language search query |
| `limit` | integer | No | 10 | Max results to return |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/auth-providers/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "google workspace"
  }'
```

**Response:**
```json
{
  "query": "google workspace",
  "results": [
    {
      "id": "uuid",
      "name": "Google",
      "slug": "google",
      "description": "Google OAuth 2.0 for Workspace and Cloud APIs",
      "logo_url": "https://example.com/google-logo.png",
      "authorization_url": "https://accounts.google.com/o/oauth2/v2/auth",
      "token_url": "https://oauth2.googleapis.com/token",
      "pkce_required": true,
      "supports_refresh": true,
      "identity_fields": ["email"],
      "scope_url_prefixes": ["https://www.googleapis.com/auth/"],
      "token_prefix": null
    }
  ],
  "meta": {
    "count": 1
  }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 400 | Missing `query` parameter |

---

## Workflow Matching

Match a user intent to relevant workflow patterns. Returns matching workflows with context guidance for AI agents.

```
POST /api/workflows/match
```

**Authentication:** None

**Request Body:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `query` | string | Yes | — | User intent or task description |
| `threshold` | float | No | 0.5 | Minimum similarity threshold |
| `max_results` | integer | No | 5 | Maximum matches to return |
| `category` | string | No | — | Filter by workflow category |

**Example:**
```bash
curl -X POST https://kb.interactor.com/api/workflows/match \
  -H "Content-Type: application/json" \
  -d '{
    "query": "onboard a new employee",
    "threshold": 0.6
  }'
```

**Response:**
```json
{
  "query": "onboard a new employee",
  "matches": [
    {
      "name": "employee_onboarding",
      "display_name": "Employee Onboarding",
      "category": "hr",
      "similarity": 0.91,
      "primary_capabilities": [
        "create_user",
        "send_invitation",
        "provision_workspace"
      ]
    }
  ],
  "guidance": "## Employee Onboarding\n\nThis workflow automates the employee onboarding process...",
  "meta": {
    "count": 1,
    "threshold": 0.6
  }
}
```

The `guidance` field contains markdown-formatted context that AI agents can use to understand the workflow and guide users through it.

**Errors:**

| Status | Description |
|--------|-------------|
| 400 | Missing `query` parameter |
