# Querying

**Version:** 1.0.0
**Last Updated:** 2026-03-25

The query API converts natural language questions into SQL (or adapter-appropriate queries) and executes them against connected data sources. Results are returned with the generated SQL and an explanation.

---

## Execute Natural Language Query

```
POST /api/v1/data-sources/:id/query
```

**Request Body:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `query` | string | Yes | — | Natural language question |
| `context` | string | No | — | Additional context for query generation (e.g., date range, user info) |
| `dry_run` | boolean | No | false | If true, generate SQL but don't execute |

**Example:**
```bash
curl -X POST https://user-kb.interactor.com/api/v1/data-sources/:id/query \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How many orders were placed last month?",
    "context": "Current date is 2026-03-25"
  }'
```

**Response:**
```json
{
  "data": {
    "results": [
      {"count": 1247}
    ],
    "metadata": {
      "sql": "SELECT COUNT(*) AS count FROM orders WHERE created_at >= '2026-02-01' AND created_at < '2026-03-01'",
      "explanation": "Counting all orders where the creation date falls within February 2026",
      "dry_run": false,
      "row_count": 1,
      "truncated": false
    }
  }
}
```

### Dry Run

Set `dry_run: true` to preview the generated SQL without executing it:

```bash
curl -X POST https://user-kb.interactor.com/api/v1/data-sources/:id/query \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Show me the top 10 customers by revenue",
    "dry_run": true
  }'
```

**Response:**
```json
{
  "data": {
    "results": [],
    "metadata": {
      "sql": "SELECT u.name, SUM(o.total_amount) AS total_revenue FROM users u JOIN orders o ON o.user_id = u.id GROUP BY u.name ORDER BY total_revenue DESC LIMIT 10",
      "explanation": "Joining users with orders to calculate total revenue per customer, sorted descending, limited to top 10",
      "dry_run": true,
      "row_count": 0,
      "truncated": false
    }
  }
}
```

### Result Truncation

Results are limited to the data source's `max_rows` setting (default 1000). When truncated, `metadata.truncated` is `true`:

```json
{
  "data": {
    "results": [ ... ],
    "metadata": {
      "sql": "SELECT * FROM orders",
      "row_count": 1000,
      "truncated": true
    }
  }
}
```

### How Queries Work

1. The natural language query is combined with the data source's schema and semantic mappings
2. An LLM generates the appropriate SQL query
3. The SQL is validated against allowed/denied tables
4. The query is executed with the configured timeout and row limits
5. Results are returned with the SQL and an explanation

Semantic mappings (see [Data Sources](01-data-sources.md#update-semantic-mappings)) and learned mappings (see [Learned Mappings](03-learned-mappings.md)) help the LLM produce accurate SQL for domain-specific terminology.

---

## Error Handling

### Missing Query

```json
{
  "error": "query_required",
  "message": "The 'query' field is required"
}
```
Status: `400`

### Conversion Failed

When the LLM cannot convert the query to SQL:

```json
{
  "error": "conversion_failed",
  "message": "Could not convert query to SQL: ambiguous table reference"
}
```
Status: `422`

### Execution Failed

When the generated SQL fails to execute:

```json
{
  "error": "execution_failed",
  "message": "Query execution failed: relation \"nonexistent_table\" does not exist",
  "metadata": {
    "sql": "SELECT * FROM nonexistent_table",
    "explanation": "Attempting to query the nonexistent_table"
  }
}
```
Status: `422`

Note: When execution fails, the response includes the generated SQL in `metadata` so you can understand what went wrong.

### Unsupported Adapter

```json
{
  "error": "adapter_not_supported",
  "message": "Natural language queries are not supported for http adapters"
}
```
Status: `422`

### Data Source Not Found

```json
{
  "error": "not_found"
}
```
Status: `404`
