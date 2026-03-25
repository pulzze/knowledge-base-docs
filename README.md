# Service Knowledge Base API

**Version:** 1.0.0
**Last Updated:** 2026-03-25

The Service Knowledge Base is Interactor's centralized repository of external service API information. It provides:

- **Service catalog** — browsable directory of integrated services with auth configs, capabilities, and API documentation
- **Semantic search** — AI-powered search across services, capabilities, and auth providers using vector embeddings
- **Auto-discovery** — automated service API discovery from URLs, documentation, or OpenAPI specs
- **Workflow matching** — context-aware guidance injection for workflow step selection
- **Feedback & metrics** — execution reliability tracking and usage analytics
- **Resource indexing** — per-account semantic search over synced external resources (repos, channels, etc.)

## Documentation

| Chapter | Description |
|---------|-------------|
| [01 — Services & Categories](01-services-and-categories.md) | Browse the service catalog, categories, and capability details |
| [02 — Search](02-search.md) | Semantic search for services, capabilities, and auth providers |
| [03 — Discovery](03-discovery.md) | Trigger and monitor automated service discovery |
| [04 — Feedback & Metrics](04-feedback-and-metrics.md) | Report execution outcomes, track reliability, and submit usage metrics |
| [05 — Resources](05-resources.md) | Sync and search external resources per account |
| [06 — Admin API](06-admin-api.md) | Service management, capability import, refinements, and discovery queue |

## Base URL

| Environment | URL |
|-------------|-----|
| Production | `https://kb.interactor.com` |
| Local Development | `http://localhost:4003` |

## Authentication

The Service Knowledge Base uses three authentication levels:

| Level | Endpoints | Auth Method |
|-------|-----------|-------------|
| **Public** | Service catalog, search, auth providers, feedback | None |
| **API** | Lookup, discovery status, metrics, resources | `Authorization: Bearer <token>` (JWT) |
| **Admin** | Service CRUD, capability import, discovery queue, refinements | `Authorization: Bearer <token>` (JWT with admin claims) |

### Getting a Token

Use OAuth client credentials to obtain a JWT:

```bash
curl -X POST https://auth.interactor.com/oauth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "<your_client_id>",
    "client_secret": "<your_client_secret>"
  }'
```

Response:
```json
{
  "access_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 3600
}
```

Include the token in subsequent requests:
```
Authorization: Bearer eyJ...
```

## Response Format

Most endpoints wrap responses in a `{"data": ...}` envelope:

```json
{
  "data": { ... }
}
```

Paginated endpoints include a `meta` object:
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

Search endpoints use a custom envelope:
```json
{
  "query": "...",
  "results": [...],
  "meta": { "count": 5, "min_confidence": 0.3 }
}
```

### Error Responses

Errors return appropriate HTTP status codes with a JSON body:

```json
{
  "error": "Description of the error"
}
```

Validation errors (422) include field-level details:

```json
{
  "errors": {
    "field_name": ["error message"]
  }
}
```

## Health Check

```
GET /health
```

Returns `200` with `{"status": "ok"}` when the service is healthy.
