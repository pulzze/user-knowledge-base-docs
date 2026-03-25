# Learned Mappings

**Version:** 1.0.0
**Last Updated:** 2026-03-25

Learned mappings capture successful natural language → SQL patterns that the system discovers during query execution. Over time, these improve query accuracy by teaching the system your domain-specific terminology.

---

## How Learned Mappings Work

1. When a user executes a query, the system records the natural language term and the SQL expression that worked
2. Mappings accumulate confidence based on success/failure counts and user selections
3. High-confidence mappings can be **promoted** to broader scopes (user → account → data source)
4. Low-quality mappings can be **dismissed** to exclude them from future queries

### Mapping Scopes

| Scope | Visibility | Description |
|-------|-----------|-------------|
| `user` | Single user | Learned from one user's queries |
| `account` | All users in account | Promoted from user scope or admin-defined |
| `data_source` | All accounts using this data source | Promoted from account scope |

Mappings at broader scopes override narrower ones. A data source-level mapping for "revenue" takes precedence over a user-level one.

---

## List Learned Mappings

Returns learned mappings for a data source, including all applicable scopes.

```
GET /api/v1/data-sources/:id/learned-mappings
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `external_user_id` | string | — | Include user-scoped mappings for this user |
| `min_confidence` | float | 0.0 | Minimum confidence threshold |
| `include_dismissed` | string | `"false"` | Set to `"true"` to include dismissed mappings |

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "term": "revenue",
      "expression": "SUM(orders.total_amount)",
      "scope_type": "data_source",
      "source": "promoted",
      "confidence": 0.95,
      "effective_score": 0.92,
      "success_count": 45,
      "failure_count": 2,
      "selection_count": 12,
      "last_used_at": "2026-03-25T10:00:00Z",
      "last_selected_at": "2026-03-24T15:30:00Z",
      "promoted_from": "account",
      "promoted_at": "2026-03-20T09:00:00Z",
      "dismissed": false,
      "dismissed_at": null
    }
  ],
  "meta": {
    "count": 5,
    "scopes_included": ["data_source", "account"]
  }
}
```

---

## List Promotion Candidates

Returns mappings that are good candidates for promotion to a broader scope, based on confidence and user adoption.

```
GET /api/v1/data-sources/:id/learned-mappings/candidates
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `min_confidence` | float | 0.7 | Minimum confidence for candidacy |
| `min_users` | integer | 2 | Minimum unique users who've used the mapping |

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "term": "monthly active users",
      "expression": "COUNT(DISTINCT user_id) FROM events WHERE created_at >= DATE_TRUNC('month', NOW())",
      "confidence": 0.88,
      "user_count": 4,
      "success_rate": 0.93
    }
  ],
  "meta": {
    "count": 1
  }
}
```

---

## Create Mapping

Manually create a learned mapping (e.g., from an admin defining a standard term).

```
POST /api/v1/data-sources/:id/learned-mappings
```

**Request Body:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `term` | string | Yes | — | Natural language term |
| `expression` | string | Yes | — | SQL expression |
| `scope_type` | string | No | `"data_source"` | Scope: `data_source`, `account`, or `user` |
| `source` | string | No | `"admin_defined"` | Origin of the mapping |
| `external_user_id` | string | No | — | Required when `scope_type` is `user` |

**Example:**
```bash
curl -X POST https://user-kb.interactor.com/api/v1/data-sources/:id/learned-mappings \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "term": "revenue",
    "expression": "SUM(orders.total_amount)",
    "scope_type": "data_source",
    "source": "admin_defined"
  }'
```

**Response (201):**
```json
{
  "data": {
    "id": "uuid",
    "term": "revenue",
    "expression": "SUM(orders.total_amount)",
    "scope_type": "data_source",
    "source": "admin_defined",
    "confidence": 1.0,
    "success_count": 0,
    "failure_count": 0
  },
  "message": "Mapping created"
}
```

**Errors:**

| Status | Code | Description |
|--------|------|-------------|
| 400 | `missing_fields` | `term` and `expression` are required |
| 422 | `validation_error` | Invalid scope_type or other validation failure |

---

## Promote Mapping

Promote a mapping to a broader scope (e.g., from user → account, or account → data_source).

```
POST /api/v1/data-sources/:id/learned-mappings/:mapping_id/promote
```

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to_scope` | string | Yes | Target scope: `account` or `data_source` |

**Example:**
```bash
curl -X POST https://user-kb.interactor.com/api/v1/data-sources/:id/learned-mappings/:mapping_id/promote \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"to_scope": "data_source"}'
```

**Response:**
```json
{
  "data": { ...mapping with updated scope... },
  "message": "Mapping promoted to data_source scope"
}
```

**Errors:**

| Status | Code | Description |
|--------|------|-------------|
| 400 | `invalid_scope` | `to_scope` must be `account` or `data_source` |
| 422 | `invalid_promotion` | Cannot promote to that scope (e.g., already at that level) |

---

## Dismiss Mapping

Mark a mapping as dismissed. Dismissed mappings are excluded from future query context.

```
POST /api/v1/data-sources/:id/learned-mappings/:mapping_id/dismiss
```

**Response:**
```json
{
  "data": { ...mapping with dismissed: true... },
  "message": "Mapping dismissed"
}
```

---

## Select Mapping

Record that a user selected this mapping during a query session. This improves the mapping's confidence score.

```
POST /api/v1/data-sources/:id/learned-mappings/:mapping_id/select
```

**Response:**
```json
{
  "data": { ...mapping with updated selection_count... },
  "message": "Selection recorded"
}
```

---

## Mapping Lifecycle

```
Query executed successfully
  → Mapping learned (scope: user, confidence: low)
    → Used successfully multiple times
      → Confidence increases
        → Appears in promotion candidates
          → Admin promotes to account scope
            → All account users benefit
              → Admin promotes to data_source scope
                → All accounts using this data source benefit

Or:

Query produced bad results
  → Mapping confidence decreases
    → Admin dismisses mapping
      → Excluded from future queries
```
