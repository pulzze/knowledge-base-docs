# Services & Categories

**Version:** 1.0.0
**Last Updated:** 2026-03-25

---

## Categories

### List Categories

Returns the full category tree with nested children.

```
GET /api/categories
```

**Authentication:** None

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "Communication",
      "slug": "communication",
      "description": "Messaging, email, and collaboration tools",
      "icon": "chat",
      "children": [
        {
          "id": "uuid",
          "name": "Email",
          "slug": "email",
          "description": "Email sending and management",
          "icon": "mail",
          "children": []
        }
      ]
    }
  ]
}
```

---

## Services

### List Services

Returns a paginated list of services. By default, only active services are returned.

```
GET /api/services
```

**Authentication:** None

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `category` | string | — | Filter by category slug |
| `status` | string | `active` | Filter by status (`active`, `draft`, `archived`) |
| `verification_status` | string | — | Filter by verification status (`verified`, `pending`, `rejected`) |
| `search` | string | — | Text search across name and description |
| `limit` | integer | 20 | Results per page (max 100) |
| `offset` | integer | 0 | Pagination offset |

**Response:**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Slack",
      "slug": "slack",
      "description": "Team messaging and collaboration platform",
      "logo_url": "https://example.com/slack-logo.png",
      "website_url": "https://slack.com",
      "status": "active",
      "verification_status": "verified",
      "category": {
        "id": "uuid",
        "name": "Communication",
        "slug": "communication"
      },
      "classification": "messaging",
      "recommended_sharing_scope": "organization",
      "identity_sensitive": false,
      "supports_resource_sync": true,
      "resource_type": "channel",
      "resource_sync_config": {
        "endpoint": "/api/conversations.list",
        "fields": ["id", "name", "topic"]
      },
      "read_only_explanation": null
    }
  ],
  "meta": {
    "total": 42,
    "limit": 20,
    "offset": 0
  }
}
```

**Example:**
```bash
# List all verified services in the "communication" category
curl "https://kb.interactor.com/api/services?category=communication&verification_status=verified"
```

---

### Get Service

Returns detailed information about a single service, including environments, auth configs, and discovery metadata.

```
GET /api/services/:id_or_slug
```

**Authentication:** None

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `id_or_slug` | string | Service UUID or slug (e.g., `slack`) |

**Response:**
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Slack",
    "slug": "slack",
    "description": "Team messaging and collaboration platform",
    "logo_url": "https://example.com/slack-logo.png",
    "website_url": "https://slack.com",
    "api_base_url": "https://slack.com/api",
    "developer_docs_url": "https://api.slack.com/docs",
    "status": "active",
    "verification_status": "verified",
    "category": {
      "id": "uuid",
      "name": "Communication",
      "slug": "communication"
    },
    "classification": "messaging",
    "recommended_sharing_scope": "organization",
    "identity_sensitive": false,
    "supports_resource_sync": true,
    "resource_type": "channel",
    "resource_sync_config": { ... },
    "read_only_explanation": null,
    "rate_limits": {
      "requests_per_minute": 60
    },
    "environments": [
      {
        "name": "production",
        "base_url": "https://slack.com/api"
      }
    ],
    "auth_configs": [
      {
        "auth_provider_slug": "slack",
        "scopes": {
          "channels:read": "View basic channel information",
          "chat:write": "Send messages"
        },
        "default_scopes": ["channels:read", "chat:write"]
      }
    ],
    "verified_at": "2026-01-15T10:30:00Z",
    "discovered_at": "2026-01-10T08:00:00Z",
    "discovery_source": "openapi_spec"
  }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Service not found |

---

## Capabilities

### List Capabilities

Returns capabilities for a specific service. Supports three format modes for different use cases.

```
GET /api/services/:id_or_slug/capabilities
```

**Authentication:** None

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `id_or_slug` | string | Service UUID or slug |

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `format` | string | `full` | Response format: `full`, `compact`, or `signatures_only` |
| `category` | string | — | Filter by capability category |
| `scopes` | string | — | Filter by required OAuth scopes (comma-separated) |

**Response (full format):**
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "send_message",
      "display_name": "Send Message",
      "description": "Send a message to a Slack channel",
      "category": "messaging",
      "signature": "send_message(channel: string, text: string, blocks?: array) -> message",
      "parameters": [
        {
          "name": "channel",
          "type": "string",
          "required": true,
          "description": "Channel ID or name"
        },
        {
          "name": "text",
          "type": "string",
          "required": true,
          "description": "Message text"
        }
      ],
      "returns": {
        "type": "object",
        "description": "The sent message object"
      },
      "returns_hint": "Returns the message ID and timestamp",
      "required_scopes": ["chat:write"],
      "api_operations": [
        {
          "method": "POST",
          "path": "/chat.postMessage"
        }
      ],
      "is_destructive": false,
      "requires_confirmation": false,
      "estimated_latency": "fast",
      "captures_identity": false,
      "default_params": {},
      "post_processing": null,
      "query_syntax_rules": null,
      "prerequisites": []
    }
  ],
  "meta": {
    "service_id": "uuid",
    "service_name": "Slack",
    "format": "full",
    "count": 15
  }
}
```

**Response (compact format):**
```json
{
  "data": [
    {
      "name": "send_message",
      "signature": "send_message(channel: string, text: string) -> message",
      "description": "Send a message to a Slack channel",
      "required_scopes": ["chat:write"]
    }
  ],
  "meta": {
    "service_id": "uuid",
    "service_name": "Slack",
    "format": "compact",
    "count": 15
  }
}
```

**Response (signatures_only format):**
```json
{
  "data": [
    "send_message(channel: string, text: string) -> message: Send a message to a Slack channel",
    "list_channels() -> channels[]: List all channels in the workspace"
  ],
  "meta": {
    "service_id": "uuid",
    "service_name": "Slack",
    "format": "signatures_only",
    "count": 15
  }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Service not found |

---

### Get Capability

Returns full details for a single capability.

```
GET /api/services/:id_or_slug/capabilities/:name
```

**Authentication:** None

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `id_or_slug` | string | Service UUID or slug |
| `name` | string | Capability name (e.g., `send_message`) |

**Response:**
```json
{
  "data": {
    "id": "uuid",
    "name": "send_message",
    "display_name": "Send Message",
    "description": "Send a message to a Slack channel",
    "category": "messaging",
    "signature": "send_message(channel: string, text: string, blocks?: array) -> message",
    "parameters": [...],
    "returns": {...},
    "returns_hint": "Returns the message ID and timestamp",
    "required_scopes": ["chat:write"],
    "api_operations": [...],
    "is_destructive": false,
    "requires_confirmation": false,
    "estimated_latency": "fast",
    "captures_identity": false,
    "default_params": {},
    "post_processing": null,
    "query_syntax_rules": null,
    "prerequisites": []
  }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Service or capability not found |

---

## Auth Providers

### List Auth Providers

Returns all auth providers with their associated services and scope definitions.

```
GET /api/auth-providers
```

**Authentication:** None

**Response:**
```json
{
  "data": [
    {
      "slug": "slack",
      "name": "Slack",
      "identity_fields": ["team_id", "user_id"],
      "scope_url_prefixes": [],
      "token_prefix": "xoxb-",
      "services": [
        {
          "slug": "slack",
          "name": "Slack",
          "logo_url": "https://example.com/slack-logo.png",
          "scopes": {
            "channels:read": "View basic channel information",
            "chat:write": "Send messages as the bot"
          },
          "default_scopes": ["channels:read", "chat:write"]
        }
      ]
    }
  ]
}
```

---

### Get Auth Provider

Returns full details for a single auth provider, including OAuth configuration and services that use it.

```
GET /api/auth-providers/:slug
```

**Authentication:** None

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `slug` | string | Auth provider slug (e.g., `slack`, `google`, `github`) |

**Response:**
```json
{
  "data": {
    "id": "uuid",
    "name": "Slack",
    "slug": "slack",
    "description": "Slack OAuth provider for workspace access",
    "logo_url": "https://example.com/slack-logo.png",
    "authorization_url": "https://slack.com/oauth/v2/authorize",
    "token_url": "https://slack.com/api/oauth.v2.access",
    "pkce_required": false,
    "supports_refresh": true,
    "token_endpoint_auth_methods": ["client_secret_post"],
    "identity_fields": ["team_id", "user_id"],
    "scope_url_prefixes": [],
    "token_prefix": "xoxb-",
    "services_using": [
      {
        "id": "uuid",
        "name": "Slack",
        "slug": "slack",
        "logo_url": "https://example.com/slack-logo.png"
      }
    ]
  }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Auth provider not found |
