# Knowledge Base Integration Guide

**Version:** 1.0.0
**Last Updated:** 2026-01-20

---

## Quick Reference

All endpoints use the base URL: `https://kb.interactor.com/api`

| Action | Method | Endpoint | Auth Required |
|--------|--------|----------|---------------|
| List Services | GET | `/services` | No |
| Get Service | GET | `/services/:id_or_slug` | No |
| List Capabilities | GET | `/services/:id/capabilities` | No |
| Get Capability | GET | `/services/:id/capabilities/:name` | No |
| Search Services | POST | `/services/search` | No |
| Search Capabilities | POST | `/capabilities/search` | No |
| Search Auth Providers | POST | `/auth-providers/search` | No |
| Service Lookup (with Discovery) | POST | `/services/lookup` | Yes (API Key) |
| Discovery Status | GET | `/discovery/:id/status` | Yes (API Key) |

---

## Overview

The Knowledge Base is a **service information repository** that provides structured data about external APIs, their authentication requirements, and available capabilities. It serves as the authoritative source for understanding how to integrate with third-party services.

### Who This Guide Is For

This guide is for developers who need to:
- Query service information for OAuth integration
- Search for services and capabilities programmatically
- Trigger service discovery for unknown services (Interactor only)

> **Building an application with Interactor?** Most developers will access Knowledge Base data through the [Interactor API](../../interactor/docs/integration-guide.md#knowledge-base), which proxies these endpoints with additional authentication and context.

### What Knowledge Base Provides

- **Service Specifications**: Detailed information about external service APIs
- **Authentication Configurations**: OAuth endpoints, scopes, API key formats
- **Service Capabilities**: What actions can be performed with each service
- **Semantic Search**: Find services and capabilities using natural language
- **Auto-Discovery**: Dynamically learn about new services when queried

### Base URLs

```
Production: https://kb.interactor.com/api
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         KNOWLEDGE BASE                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      PUBLIC ENDPOINTS                                  │  │
│  │                                                                        │  │
│  │  GET /services              - List & filter services                  │  │
│  │  GET /services/:id          - Get service details                     │  │
│  │  GET /services/:id/capabilities - Get service capabilities            │  │
│  │  POST /services/search      - Semantic service search                 │  │
│  │  POST /capabilities/search  - Find capabilities by intent             │  │
│  │  POST /auth-providers/search - Find OAuth providers                   │  │
│  │  GET /categories            - List service categories                 │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                   AUTHENTICATED ENDPOINTS (API Key)                    │  │
│  │                                                                        │  │
│  │  POST /services/lookup      - Query with auto-discovery               │  │
│  │  GET /discovery/:id/status  - Check discovery progress                │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      ADMIN ENDPOINTS                                   │  │
│  │                                                                        │  │
│  │  CRUD operations for services, capabilities, discovery queue          │  │
│  │  (Internal use only - not covered in this guide)                      │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Start

### 1. Browse Available Services

List services with optional filtering:

```bash
curl https://kb.interactor.com/api/services
```

Filter by category or authentication type:

```bash
curl "https://kb.interactor.com/api/services?category=productivity&auth_type=oauth2_authorization_code"
```

### 2. Get Service Details

Retrieve complete service information by ID or slug:

```bash
curl https://kb.interactor.com/api/services/google-calendar
```

**Response:**
```json
{
  "data": {
    "id": "svc_google_calendar",
    "name": "Google Calendar",
    "slug": "google-calendar",
    "description": "Google Calendar API for managing events and schedules",
    "website_url": "https://calendar.google.com",
    "developer_docs_url": "https://developers.google.com/calendar",
    "auth_configs": [
      {
        "auth_type": "oauth2_authorization_code",
        "is_primary": true,
        "auth_provider": {
          "name": "Google",
          "slug": "google",
          "authorization_url": "https://accounts.google.com/o/oauth2/v2/auth",
          "token_url": "https://oauth2.googleapis.com/token"
        },
        "oauth_config": {
          "scopes": {
            "https://www.googleapis.com/auth/calendar.readonly": "View calendar events",
            "https://www.googleapis.com/auth/calendar": "Full calendar access"
          },
          "pkce_required": true,
          "supports_refresh": true
        }
      }
    ],
    "status": "active",
    "verification_status": "verified"
  }
}
```

### 3. Search for Services

Use natural language to find services:

```bash
curl -X POST https://kb.interactor.com/api/services/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "calendar scheduling",
    "limit": 5
  }'
```

### 4. Find Capabilities

Search for what you can do across all services:

```bash
curl -X POST https://kb.interactor.com/api/capabilities/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "send notifications to team members",
    "limit": 10
  }'
```

**Response:**
```json
{
  "data": {
    "search_event_id": "550e8400-e29b-41d4-a716-446655440099",
    "query": "send notifications to team members",
    "results": [
      {
        "capability_id": "cap_slack_send_message",
        "service_id": "svc_slack",
        "service_name": "Slack",
        "name": "send_message",
        "signature": "send_message(channel, text, thread_id?) → {ts, channel}",
        "description": "Send a message to a Slack channel or direct message",
        "category": "messaging",
        "confidence": 0.94
      }
    ]
  }
}
```

---

## API Reference

### Public Endpoints (No Authentication)

#### GET /services

List all services with optional filtering.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `category` | string | Filter by category slug |
| `auth_type` | string | Filter by authentication type |
| `status` | string | Filter by status (default: `active`) |
| `verification` | string | Filter by verification status |
| `search` | string | Full-text search query |
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Items per page (default: 20, max: 100) |

```bash
curl "https://kb.interactor.com/api/services?category=productivity&limit=10"
```

**Response:**
```json
{
  "data": {
    "services": [
      {
        "id": "svc_google_calendar",
        "name": "Google Calendar",
        "slug": "google-calendar",
        "description": "Google Calendar API for managing events and schedules",
        "category": {
          "id": "cat_productivity",
          "name": "Productivity",
          "slug": "productivity"
        },
        "auth_types": ["oauth2_authorization_code"],
        "status": "active",
        "verification_status": "verified"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total": 25,
      "total_pages": 3
    }
  }
}
```

---

#### GET /services/:id_or_slug

Get detailed service information.

```bash
curl https://kb.interactor.com/api/services/slack
```

**Response:**
```json
{
  "data": {
    "id": "svc_slack",
    "name": "Slack",
    "slug": "slack",
    "description": "Team communication and collaboration platform",
    "website_url": "https://slack.com",
    "developer_docs_url": "https://api.slack.com",
    "environments": [
      {
        "environment_type": "production",
        "base_url": "https://slack.com/api",
        "is_default": true
      }
    ],
    "auth_configs": [
      {
        "auth_type": "oauth2_authorization_code",
        "is_primary": true,
        "auth_provider": {
          "id": "auth_slack",
          "name": "Slack",
          "slug": "slack",
          "authorization_url": "https://slack.com/oauth/v2/authorize",
          "token_url": "https://slack.com/api/oauth.v2.access",
          "revocation_url": "https://slack.com/api/auth.revoke"
        },
        "oauth_config": {
          "scopes": {
            "chat:write": "Post messages to channels",
            "channels:read": "View channel information",
            "users:read": "View user profiles"
          },
          "default_scopes": []
        }
      }
    ],
    "rate_limits": [
      {
        "tier": "default",
        "requests_per_minute": 50,
        "notes": "Tier 2 methods"
      }
    ],
    "status": "active",
    "verification_status": "verified"
  }
}
```

---

#### GET /services/:id_or_slug/capabilities

Get all capabilities for a service.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `format` | string | `full` (default), `compact`, or `signatures_only` |
| `category` | string | Filter by capability category |
| `scopes` | string | Comma-separated OAuth scopes to filter by |

```bash
curl "https://kb.interactor.com/api/services/slack/capabilities?format=compact"
```

**Response (format=compact):**
```json
{
  "data": {
    "service_id": "svc_slack",
    "service_name": "Slack",
    "capabilities": [
      {
        "name": "send_message",
        "signature": "send_message(channel, text, thread_id?) → {ts, channel}",
        "description": "Send a message to a channel or user",
        "category": "messaging",
        "required_scopes": ["chat:write"]
      },
      {
        "name": "list_channels",
        "signature": "list_channels(types?, limit?) → [{id, name, is_private}]",
        "description": "List available channels",
        "category": "data_read",
        "required_scopes": ["channels:read"]
      }
    ],
    "total_capabilities": 2
  }
}
```

**Response (format=signatures_only):**
```json
{
  "data": {
    "service_id": "svc_slack",
    "service_name": "Slack",
    "signatures": [
      "send_message(channel, text, thread_id?) → {ts, channel}: Send a message to a channel or user",
      "list_channels(types?, limit?) → [{id, name, is_private}]: List available channels"
    ]
  }
}
```

---

#### GET /services/:id/capabilities/:name

Get full details for a specific capability.

```bash
curl https://kb.interactor.com/api/services/slack/capabilities/send_message
```

**Response:**
```json
{
  "data": {
    "id": "cap_slack_send_message",
    "service_id": "svc_slack",
    "name": "send_message",
    "display_name": "Send Message",
    "signature": "send_message(channel, text, thread_id?) → {ts, channel}",
    "description": "Send a message to a Slack channel or direct message to a user",
    "category": "messaging",
    "required_scopes": ["chat:write"],
    "parameters": {
      "type": "object",
      "properties": {
        "channel": {
          "type": "string",
          "description": "Channel name (e.g., '#general') or user ID for DM",
          "required": true
        },
        "text": {
          "type": "string",
          "description": "Message content (supports Slack markdown)",
          "required": true,
          "max_length": 4000
        },
        "thread_id": {
          "type": "string",
          "description": "Thread timestamp to reply in thread",
          "required": false
        }
      },
      "required": ["channel", "text"]
    },
    "returns": {
      "type": "object",
      "properties": {
        "ts": { "type": "string", "description": "Message timestamp (unique ID)" },
        "channel": { "type": "string", "description": "Channel ID" }
      }
    },
    "is_destructive": false,
    "requires_confirmation": false
  }
}
```

---

#### POST /services/search

Search for services using natural language.

```bash
curl -X POST https://kb.interactor.com/api/services/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "project management tools",
    "limit": 5
  }'
```

**Request Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Natural language search query |
| `limit` | integer | No | Maximum results (default: 10) |
| `min_confidence` | float | No | Minimum similarity score 0.0-1.0 (default: 0.5) |

**Response:**
```json
{
  "data": {
    "query": "project management tools",
    "results": [
      {
        "id": "svc_asana",
        "name": "Asana",
        "slug": "asana",
        "description": "Work management platform for teams",
        "confidence": 0.92
      },
      {
        "id": "svc_jira",
        "name": "Jira",
        "slug": "jira",
        "description": "Project and issue tracking software",
        "confidence": 0.89
      }
    ],
    "meta": {
      "count": 2,
      "min_confidence": 0.5
    }
  }
}
```

---

#### POST /capabilities/search

Search for capabilities across all services using natural language. Supports feedback learning to improve results over time.

```bash
curl -X POST https://kb.interactor.com/api/capabilities/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "send email with attachments",
    "limit": 5,
    "min_confidence": 0.7
  }'
```

**Request Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Natural language search query |
| `limit` | integer | No | Maximum results (default: 10) |
| `min_confidence` | float | No | Minimum similarity score 0.0-1.0 (default: 0.7) |
| `with_feedback` | boolean | No | Enable feedback learning (default: true) |
| `filters.categories` | array | No | Filter by capability categories |
| `filters.service_ids` | array | No | Filter by service IDs or slugs |

**Response:**
```json
{
  "data": {
    "search_event_id": "550e8400-e29b-41d4-a716-446655440099",
    "query": "send email with attachments",
    "results": [
      {
        "capability_id": "cap_gmail_send_email",
        "service_id": "svc_gmail",
        "service_name": "Gmail",
        "name": "send_email",
        "signature": "send_email(to, subject, body, attachments?) → {id, thread_id}",
        "description": "Send an email message with optional attachments",
        "category": "messaging",
        "required_scopes": ["gmail.send"],
        "confidence": 0.96,
        "from_cache": false,
        "boosted": false
      }
    ],
    "meta": {
      "count": 1,
      "min_confidence": 0.7,
      "feedback_enabled": true
    }
  }
}
```

---

#### POST /capabilities/search/feedback

Record user feedback when a capability is selected. This helps improve future search results.

```bash
curl -X POST https://kb.interactor.com/api/capabilities/search/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "search_event_id": "550e8400-e29b-41d4-a716-446655440099",
    "selected_capability_id": "cap_gmail_send_email",
    "was_clarified": false
  }'
```

**Request Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `search_event_id` | UUID | Yes | The search event ID from the search response |
| `selected_capability_id` | UUID | Yes | The capability ID the user selected |
| `was_clarified` | boolean | No | Whether user went through clarification flow |

**Response:**
```json
{
  "data": {
    "status": "recorded"
  }
}
```

---

#### POST /auth-providers/search

Search for OAuth authentication providers (e.g., "google", "microsoft", "slack").

```bash
curl -X POST https://kb.interactor.com/api/auth-providers/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "google",
    "limit": 5
  }'
```

**Response:**
```json
{
  "data": {
    "query": "google",
    "results": [
      {
        "id": "auth_google",
        "name": "Google",
        "slug": "google",
        "description": "Google OAuth 2.0 authentication provider",
        "authorization_url": "https://accounts.google.com/o/oauth2/v2/auth",
        "token_url": "https://oauth2.googleapis.com/token",
        "pkce_required": true,
        "supports_refresh": true
      }
    ],
    "meta": {
      "count": 1
    }
  }
}
```

---

#### GET /categories

List all service categories.

```bash
curl https://kb.interactor.com/api/categories
```

**Response:**
```json
{
  "data": {
    "categories": [
      {
        "id": "cat_productivity",
        "name": "Productivity",
        "slug": "productivity",
        "description": "Tools for productivity and organization",
        "service_count": 25,
        "subcategories": [
          {
            "id": "cat_calendar",
            "name": "Calendar",
            "slug": "calendar",
            "service_count": 8
          }
        ]
      },
      {
        "id": "cat_communication",
        "name": "Communication",
        "slug": "communication",
        "description": "Messaging and collaboration tools",
        "service_count": 18
      }
    ]
  }
}
```

---

### Authenticated Endpoints (API Key Required)

These endpoints require an API key for authentication. The API key is used by Interactor to trigger service discovery for unknown services.

#### POST /services/lookup

Query for a service, triggering discovery if not found.

**Headers:**
```
X-API-Key: <api_key>
```

```bash
curl -X POST https://kb.interactor.com/api/services/lookup \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "identifier": "notion",
    "wait_for_discovery": true,
    "discovery_timeout_seconds": 30
  }'
```

**Request Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `identifier` | string | Yes | Service name, slug, or URL |
| `context.requested_auth_type` | string | No | Preferred auth type |
| `context.required_scopes` | array | No | Required OAuth scopes |
| `wait_for_discovery` | boolean | No | Wait for discovery to complete (default: false) |
| `discovery_timeout_seconds` | integer | No | Discovery timeout (default: 30) |

**Response (Service Found):**
```json
{
  "data": {
    "status": "found",
    "service": {
      "id": "svc_notion",
      "name": "Notion",
      "auth_configs": [...],
      "verification_status": "verified"
    }
  }
}
```

**Response (Discovery Initiated):**
```json
{
  "data": {
    "status": "discovery_initiated",
    "discovery_id": "disc_abc123",
    "estimated_time_seconds": 30,
    "poll_url": "/discovery/disc_abc123/status"
  }
}
```

**Response (Discovery Completed):**
```json
{
  "data": {
    "status": "discovered",
    "service": {
      "id": "svc_new_service",
      "name": "New Service",
      "verification_status": "unverified",
      "auth_configs": [...]
    },
    "discovery_confidence": 0.85,
    "needs_verification": true
  }
}
```

---

#### GET /discovery/:id/status

Check discovery status.

**Headers:**
```
X-API-Key: <api_key>
```

```bash
curl https://kb.interactor.com/api/discovery/disc_abc123/status \
  -H "X-API-Key: your_api_key"
```

**Response:**
```json
{
  "data": {
    "discovery_id": "disc_abc123",
    "status": "in_progress",
    "started_at": "2026-01-20T10:00:00Z",
    "progress": {
      "openapi_search": "completed",
      "documentation_scrape": "in_progress",
      "ai_extraction": "pending"
    }
  }
}
```

**Status Values:**
| Status | Description |
|--------|-------------|
| `pending` | Discovery queued but not started |
| `in_progress` | Discovery is running |
| `completed` | Discovery finished successfully |
| `failed` | Discovery failed |

---

## Data Models

### Service

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique service identifier (e.g., `svc_slack`) |
| `name` | string | Display name |
| `slug` | string | URL-friendly identifier |
| `description` | string | Service description |
| `website_url` | string | Main website URL |
| `developer_docs_url` | string | Developer documentation URL |
| `environments` | array | API environments (production, sandbox) |
| `auth_configs` | array | Authentication configurations |
| `rate_limits` | array | Rate limit information |
| `status` | string | `active`, `deprecated`, `disabled` |
| `verification_status` | string | `verified`, `unverified`, `rejected` |

### Auth Config

| Field | Type | Description |
|-------|------|-------------|
| `auth_type` | string | `oauth2_authorization_code`, `oauth2_pkce`, `api_key`, etc. |
| `is_primary` | boolean | Whether this is the primary auth method |
| `auth_provider` | object | OAuth provider details (for OAuth types) |
| `oauth_config` | object | OAuth-specific configuration (scopes, etc.) |
| `api_key_config` | object | API key configuration (placement, header name) |

### Capability

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique capability identifier |
| `name` | string | Capability name (snake_case) |
| `display_name` | string | Human-readable name |
| `signature` | string | Function signature with parameters and return type |
| `description` | string | What this capability does |
| `category` | string | `messaging`, `data_read`, `data_write`, `file_ops`, etc. |
| `required_scopes` | array | OAuth scopes required |
| `parameters` | object | JSON Schema for parameters |
| `returns` | object | JSON Schema for return value |

---

## Error Handling

All errors follow a consistent format:

```json
{
  "error": {
    "code": "error_code",
    "message": "Human-readable message"
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `not_found` | 404 | Service or capability not found |
| `validation_error` | 422 | Invalid request parameters |
| `unauthorized` | 401 | Missing or invalid API key |
| `rate_limited` | 429 | Too many requests |
| `discovery_failed` | 500 | Service discovery failed |
| `discovery_timeout` | 504 | Discovery timed out |

---

## Code Examples

### Node.js / TypeScript

```typescript
class KnowledgeBaseClient {
  private baseUrl = 'https://kb.interactor.com/api';
  private apiKey?: string;

  constructor(apiKey?: string) {
    this.apiKey = apiKey;
  }

  private async request(method: string, path: string, body?: any) {
    const headers: Record<string, string> = {
      'Content-Type': 'application/json'
    };

    if (this.apiKey) {
      headers['X-API-Key'] = this.apiKey;
    }

    const response = await fetch(`${this.baseUrl}${path}`, {
      method,
      headers,
      body: body ? JSON.stringify(body) : undefined
    });

    const data = await response.json();

    if (!response.ok) {
      throw new Error(data.error?.message || 'Request failed');
    }

    return data;
  }

  // Public endpoints
  async listServices(params?: { category?: string; auth_type?: string }) {
    const query = new URLSearchParams(params as any).toString();
    return this.request('GET', `/services${query ? `?${query}` : ''}`);
  }

  async getService(idOrSlug: string) {
    return this.request('GET', `/services/${idOrSlug}`);
  }

  async getCapabilities(serviceId: string, format: 'full' | 'compact' | 'signatures_only' = 'compact') {
    return this.request('GET', `/services/${serviceId}/capabilities?format=${format}`);
  }

  async searchServices(query: string, limit = 10) {
    return this.request('POST', '/services/search', { query, limit });
  }

  async searchCapabilities(query: string, options?: { limit?: number; min_confidence?: number }) {
    return this.request('POST', '/capabilities/search', { query, ...options });
  }

  async searchAuthProviders(query: string) {
    return this.request('POST', '/auth-providers/search', { query });
  }

  // Authenticated endpoints (requires API key)
  async lookupService(identifier: string, waitForDiscovery = true) {
    return this.request('POST', '/services/lookup', {
      identifier,
      wait_for_discovery: waitForDiscovery,
      discovery_timeout_seconds: 30
    });
  }

  async getDiscoveryStatus(discoveryId: string) {
    return this.request('GET', `/discovery/${discoveryId}/status`);
  }
}

// Usage
const kb = new KnowledgeBaseClient();

// Search for services
const services = await kb.searchServices('calendar scheduling');
console.log('Found services:', services.data.results);

// Get service capabilities
const capabilities = await kb.getCapabilities('slack', 'compact');
console.log('Slack capabilities:', capabilities.data.capabilities);

// Search for capabilities
const results = await kb.searchCapabilities('send message to team');
console.log('Matching capabilities:', results.data.results);
```

### Python

```python
import requests
from typing import Optional, Dict, Any, List

class KnowledgeBaseClient:
    def __init__(self, api_key: Optional[str] = None):
        self.base_url = 'https://kb.interactor.com/api'
        self.api_key = api_key

    def _request(self, method: str, path: str, data: Optional[Dict] = None) -> Dict[str, Any]:
        headers = {'Content-Type': 'application/json'}

        if self.api_key:
            headers['X-API-Key'] = self.api_key

        response = requests.request(
            method,
            f'{self.base_url}{path}',
            headers=headers,
            json=data
        )
        response.raise_for_status()
        return response.json()

    # Public endpoints
    def list_services(self, category: Optional[str] = None, auth_type: Optional[str] = None) -> Dict:
        params = []
        if category:
            params.append(f'category={category}')
        if auth_type:
            params.append(f'auth_type={auth_type}')
        query = '?' + '&'.join(params) if params else ''
        return self._request('GET', f'/services{query}')

    def get_service(self, id_or_slug: str) -> Dict:
        return self._request('GET', f'/services/{id_or_slug}')

    def get_capabilities(self, service_id: str, format: str = 'compact') -> Dict:
        return self._request('GET', f'/services/{service_id}/capabilities?format={format}')

    def search_services(self, query: str, limit: int = 10) -> Dict:
        return self._request('POST', '/services/search', {'query': query, 'limit': limit})

    def search_capabilities(self, query: str, limit: int = 10, min_confidence: float = 0.7) -> Dict:
        return self._request('POST', '/capabilities/search', {
            'query': query,
            'limit': limit,
            'min_confidence': min_confidence
        })

    def search_auth_providers(self, query: str) -> Dict:
        return self._request('POST', '/auth-providers/search', {'query': query})

    # Authenticated endpoints
    def lookup_service(self, identifier: str, wait_for_discovery: bool = True) -> Dict:
        return self._request('POST', '/services/lookup', {
            'identifier': identifier,
            'wait_for_discovery': wait_for_discovery,
            'discovery_timeout_seconds': 30
        })

    def get_discovery_status(self, discovery_id: str) -> Dict:
        return self._request('GET', f'/discovery/{discovery_id}/status')


# Usage
kb = KnowledgeBaseClient()

# Search for services
services = kb.search_services('calendar scheduling')
print('Found services:', services['data']['results'])

# Get service capabilities
capabilities = kb.get_capabilities('slack', 'compact')
print('Slack capabilities:', capabilities['data']['capabilities'])

# Search for capabilities
results = kb.search_capabilities('send message to team')
print('Matching capabilities:', results['data']['results'])
```

---

## Troubleshooting

### Service Not Found

If a service isn't in the Knowledge Base:
1. Check the spelling and try alternative names
2. Use the search endpoint with natural language
3. If using the authenticated API, set `wait_for_discovery: true` to trigger auto-discovery

### Low Confidence Search Results

If search results have low confidence scores:
1. Be more specific in your query
2. Use filters to narrow down results
3. Try different phrasing or synonyms

### Discovery Timeout

If discovery times out:
1. Increase `discovery_timeout_seconds` (max: 60)
2. Check the discovery status endpoint for progress
3. The service may have limited or inaccessible documentation

### Rate Limiting

If you receive 429 errors:
1. Implement exponential backoff
2. Cache responses where appropriate
3. Contact support for higher rate limits if needed