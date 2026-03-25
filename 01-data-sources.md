# Data Sources

**Version:** 1.0.0
**Last Updated:** 2026-03-25

---

## List Data Sources

Returns all data sources for the authenticated account. By default, only active data sources are returned.

```
GET /api/v1/data-sources
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `active_only` | string | `"true"` | Set to `"false"` to include inactive/archived data sources |

**Response:**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Production Database",
      "description": "Main PostgreSQL database",
      "adapter": "postgresql",
      "capabilities": ["read"],
      "max_rows": 1000,
      "timeout_ms": 30000,
      "active": true,
      "schema_refreshed_at": "2026-03-25T10:00:00Z",
      "created_at": "2026-03-01T08:00:00Z"
    }
  ]
}
```

---

## Get Data Source

Returns full details for a data source, including schema information and semantic mappings.

```
GET /api/v1/data-sources/:id
```

**Response:**
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Production Database",
    "description": "Main PostgreSQL database",
    "adapter": "postgresql",
    "capabilities": ["read"],
    "max_rows": 1000,
    "timeout_ms": 30000,
    "active": true,
    "schema_refreshed_at": "2026-03-25T10:00:00Z",
    "created_at": "2026-03-01T08:00:00Z",
    "schema_info": {
      "users": {
        "columns": [
          {"name": "id", "type": "integer", "nullable": false},
          {"name": "email", "type": "varchar", "nullable": false},
          {"name": "created_at", "type": "timestamp", "nullable": false}
        ]
      },
      "orders": {
        "columns": [
          {"name": "id", "type": "integer", "nullable": false},
          {"name": "user_id", "type": "integer", "nullable": false},
          {"name": "total_amount", "type": "numeric", "nullable": false}
        ]
      }
    },
    "semantic_mappings": {
      "terms": {
        "revenue": "SUM(orders.total_amount)",
        "active user": "users WHERE last_login_at > NOW() - INTERVAL '30 days'"
      }
    },
    "allowed_tables": [],
    "denied_tables": ["internal_logs"]
  }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Data source not found or not owned by account |

---

## Create Data Source

```
POST /api/v1/data-sources
```

**Request Body:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | — | Display name |
| `adapter` | string | Yes | — | Database adapter (`postgresql`, `mysql`, `http`, `graphql`) |
| `connection_config` | object | Yes | — | Connection details (see below) |
| `description` | string | No | null | Description |
| `capabilities` | array | No | `["read"]` | Capabilities (`read`, `write`) |
| `max_rows` | integer | No | 1000 | Maximum rows per query |
| `timeout_ms` | integer | No | 30000 | Query timeout in milliseconds |
| `allowed_tables` | array | No | `[]` | Whitelist of queryable tables (empty = all) |
| `denied_tables` | array | No | `[]` | Blacklist of tables to exclude |
| `semantic_mappings` | object | No | `{}` | Initial semantic mapping definitions |
| `metadata` | object | No | `{}` | Custom metadata |

### Connection Config

For PostgreSQL/MySQL:
```json
{
  "connection_config": {
    "host": "db.example.com",
    "port": 5432,
    "database": "myapp_production",
    "username": "readonly_user",
    "password": "secret",
    "ssl": true
  }
}
```

For HTTP APIs:
```json
{
  "connection_config": {
    "base_url": "https://api.example.com",
    "headers": {
      "Authorization": "Bearer token"
    }
  }
}
```

**Example:**
```bash
curl -X POST https://user-kb.interactor.com/api/v1/data-sources \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Production Database",
    "adapter": "postgresql",
    "connection_config": {
      "host": "db.example.com",
      "port": 5432,
      "database": "myapp",
      "username": "readonly",
      "password": "secret",
      "ssl": true
    },
    "denied_tables": ["internal_logs", "migrations"]
  }'
```

**Response (201):**
```json
{
  "data": {
    "id": "uuid",
    "name": "Production Database",
    "adapter": "postgresql",
    "capabilities": ["read"],
    "max_rows": 1000,
    "timeout_ms": 30000,
    "active": true,
    "schema_refreshed_at": null,
    "created_at": "2026-03-25T10:00:00Z"
  }
}
```

**Errors:**

| Status | Description |
|--------|-------------|
| 422 | Validation error (missing name, invalid adapter, etc.) |

---

## Update Data Source

```
PUT /api/v1/data-sources/:id
```

**Request Body:** Same fields as create (all optional).

**Response:** Same format as create (200).

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Data source not found |
| 422 | Validation error |

---

## Delete Data Source

Soft-deletes the data source (sets `active: false`). The data source can be recovered by updating `active` back to `true`.

```
DELETE /api/v1/data-sources/:id
```

**Response:** `204 No Content`

**Errors:**

| Status | Description |
|--------|-------------|
| 404 | Data source not found |

---

## Refresh Schema

Introspect the data source connection to discover tables, columns, and types. Includes AI-powered analysis that generates column descriptions and identifies relationships.

```
POST /api/v1/data-sources/:id/refresh-schema
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `force` | boolean | false | Override 5-minute cooldown |
| `include_analysis` | boolean | true | Include AI-powered schema analysis |

**Example:**
```bash
curl -X POST "https://user-kb.interactor.com/api/v1/data-sources/:id/refresh-schema?force=true" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "status": "success",
    "tables_count": 12,
    "tables": [
      {
        "name": "users",
        "columns": [
          {
            "name": "id",
            "type": "integer",
            "nullable": false,
            "description": "Primary key"
          },
          {
            "name": "email",
            "type": "varchar",
            "nullable": false,
            "description": "User's email address"
          }
        ]
      }
    ],
    "analysis": {
      "relationships": [
        {"from": "orders.user_id", "to": "users.id", "type": "belongs_to"}
      ],
      "suggestions": [
        "Consider adding a semantic mapping for 'revenue' → SUM(orders.total_amount)"
      ]
    },
    "refreshed_at": "2026-03-25T10:00:00Z"
  }
}
```

**Errors:**

| Status | Code | Description |
|--------|------|-------------|
| 404 | `not_found` | Data source not found |
| 422 | `unsupported_adapter` | Adapter doesn't support schema refresh |
| 422 | `connection_failed` | Cannot connect to data source |
| 429 | `cooldown_active` | Schema refreshed recently (use `force=true`) |

---

## Update Semantic Mappings

Define business terminology and semantic hints that help the NL→SQL engine produce accurate queries.

```
PATCH /api/v1/data-sources/:id/semantic-mappings
```

**Request Body:**

```json
{
  "terms": {
    "revenue": "SUM(orders.total_amount)",
    "active user": "users WHERE last_login_at > NOW() - INTERVAL '30 days'",
    "churn rate": "COUNT(CASE WHEN status = 'churned' THEN 1 END) / COUNT(*)"
  },
  "relationships": {
    "orders.user_id": "users.id"
  },
  "users": {
    "description": "Application users table",
    "columns": [
      {"name": "status", "description": "Account status: active, suspended, churned"}
    ]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `terms` | object | Business terms mapped to SQL expressions |
| `relationships` | object | Table relationships for JOINs |
| `[table_name]` | object | Per-table semantic hints (description, column descriptions) |

**Response:**
```json
{
  "data": {
    "id": "uuid",
    "semantic_mappings": { ... }
  },
  "message": "Semantic mappings updated"
}
```
