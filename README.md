# User Knowledge Base API

**Version:** 1.0.0
**Last Updated:** 2026-03-25

The User Knowledge Base manages per-account data source connections. It enables natural language querying of databases and APIs, automatic schema discovery, and learned semantic mappings that improve query accuracy over time.

## Core Features

- **Data source CRUD** — connect databases (PostgreSQL, MySQL, etc.) and APIs as queryable data sources
- **Natural language queries** — convert plain English questions into SQL and execute them
- **Schema discovery** — automatically introspect database schemas with AI-powered analysis
- **Semantic mappings** — define business terminology (e.g., "revenue" → `SUM(orders.total_amount)`)
- **Learned mappings** — system learns from successful queries and user selections to improve accuracy

## Documentation

| Chapter | Description |
|---------|-------------|
| [01 — Data Sources](01-data-sources.md) | CRUD operations, schema refresh, and semantic mappings |
| [02 — Querying](02-querying.md) | Natural language queries, dry runs, and result format |
| [03 — Learned Mappings](03-learned-mappings.md) | Mapping lifecycle: create, learn, promote, dismiss |

## Base URL

| Environment | URL |
|-------------|-----|
| Production | `https://user-kb.interactor.com` |
| Local Development | `http://localhost:4005` |

## Authentication

All API endpoints require JWT authentication via the `Authorization` header:

```
Authorization: Bearer <token>
```

Obtain a token using OAuth client credentials from the Account Server:

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

All data is automatically scoped to the authenticated account. You can only access data sources owned by your account.

## Response Format

All endpoints use a `{"data": ...}` envelope:

```json
{
  "data": { ... }
}
```

### Error Responses

Errors return appropriate HTTP status codes:

```json
{
  "error": "error_code",
  "message": "Human-readable description"
}
```

Validation errors (422) include field-level details:

```json
{
  "error": "validation_error",
  "details": {
    "name": ["can't be blank"],
    "adapter": ["is invalid"]
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `missing_token` | 401 | No Authorization header |
| `invalid_token` | 401 | Expired or malformed token |
| `not_found` | 404 | Data source not found or not owned by account |
| `validation_error` | 422 | Invalid request parameters |
| `cooldown_active` | 429 | Schema refresh cooldown (use `force=true`) |

## Health Check

```
GET /health
```

Returns `200` with `{"status": "ok", "service": "user-knowledge-base"}`.
