# API

The Conserver provides a REST API built on FastAPI for managing vCons, chains, configuration, and more. The API supports both internal operations and external partner integrations with scoped authentication.

## Base URL

The API is served at the path configured by the `API_ROOT_PATH` environment variable (default: `/api`).

## Authorization

The Conserver API supports two authentication models:

### Internal API Authentication

For full system access, use the main API token via the `x-conserver-api-token` header (or custom header name via `CONSERVER_HEADER_NAME`).

```bash
curl -H "x-conserver-api-token: your-api-token" \
  "http://localhost:8000/api/vcon"
```

Configure the API token via environment variables:

| Variable | Description |
|----------|-------------|
| `CONSERVER_API_TOKEN` | Single API token for authentication |
| `CONSERVER_API_TOKEN_FILE` | Path to file containing API tokens (one per line) |
| `CONSERVER_HEADER_NAME` | Custom header name (default: `x-conserver-api-token`) |

When neither `CONSERVER_API_TOKEN` nor `CONSERVER_API_TOKEN_FILE` is set, authentication is disabled.

### External Ingress Authentication

For external partners, use ingress-specific API keys that only allow access to designated ingress lists. Configure in `config.yml`:

```yaml
ingress_auth:
  partner_ingress:
    - "partner-key-1"
    - "partner-key-2"
  customer_data: "single-customer-key"
```

External partners can only use the `/vcon/external-ingress` endpoint with their scoped keys.

---

## vCon Management

### List vCon UUIDs

```
GET /vcon
```

Retrieves a paginated list of vCon UUIDs, sorted by timestamp (newest first).

**Query Parameters:**

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `page` | int | Page number (1-indexed) | `1` |
| `size` | int | Items per page | `50` |
| `since` | datetime | Filter vCons created after this date | (none) |
| `until` | datetime | Filter vCons created before this date | (none) |

**Response:** `200 OK`
```json
["uuid-1", "uuid-2", "uuid-3"]
```

**Example:**
```bash
curl -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/vcon?page=1&size=10&since=2024-01-01"
```

---

### Get vCon by UUID

```
GET /vcon/{vcon_uuid}
```

Retrieves a single vCon by its UUID. First checks Redis, then falls back to configured storage backends.

**Response:** `200 OK` - Returns the full vCon JSON

**Response:** `404 Not Found` - vCon not found

**Example:**
```bash
curl -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/vcon/550e8400-e29b-41d4-a716-446655440000"
```

---

### Get Multiple vCons

```
GET /vcons
```

Retrieves multiple vCons by their UUIDs in a single request.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `vcon_uuids` | List[UUID] | List of vCon UUIDs to retrieve |

**Response:** `200 OK`
```json
[
  { "uuid": "...", "vcon": "0.0.1", ... },
  { "uuid": "...", "vcon": "0.0.1", ... }
]
```

**Example:**
```bash
curl -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/vcons?vcon_uuids=uuid1&vcon_uuids=uuid2"
```

---

### Create vCon

```
POST /vcon
```

Stores a new vCon in Redis and indexes it for searching.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ingress_lists` | List[str] | Optional ingress queues to add the vCon to |

**Request Body:** Full vCon JSON object

**Response:** `201 Created` - Returns the stored vCon

**Example:**
```bash
curl -X POST "http://localhost:8000/api/vcon?ingress_lists=main_chain" \
  -H "x-conserver-api-token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "vcon": "0.0.1",
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "created_at": "2024-01-15T10:30:00Z",
    "parties": [],
    "dialog": []
  }'
```

---

### Delete vCon

```
DELETE /vcon/{vcon_uuid}
```

Removes a vCon from Redis and all configured storage backends.

**Response:** `204 No Content`

**Example:**
```bash
curl -X DELETE -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/vcon/550e8400-e29b-41d4-a716-446655440000"
```

---

### Search vCons

```
GET /vcons/search
```

Search for vCons by party information (phone, email, name). At least one parameter is required. Multiple parameters use AND logic.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tel` | string | Phone number to search for |
| `mailto` | string | Email address to search for |
| `name` | string | Party name to search for |

**Response:** `200 OK`
```json
["uuid-1", "uuid-2"]
```

**Example:**
```bash
curl -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/vcons/search?tel=%2B1234567890&name=John"
```

---

## External Partner API

### Submit External vCon

```
POST /vcon/external-ingress
```

Endpoint for external partners to submit vCons with scoped API access. Each API key is restricted to specific ingress lists.

**Authentication:** Uses ingress-specific API keys configured in `ingress_auth`

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ingress_list` | string | Target ingress queue (must match API key permissions) |

**Request Body:** Full vCon JSON object

**Response:** `204 No Content`

**Response:** `403 Forbidden` - Invalid API key or unauthorized ingress list

**Example:**
```bash
curl -X POST "http://localhost:8000/api/vcon/external-ingress?ingress_list=partner_data" \
  -H "x-conserver-api-token: partner-specific-key" \
  -H "Content-Type: application/json" \
  -d @vcon.json
```

**Security Model:**
- Each API key grants access only to predefined ingress list(s)
- No access to other API endpoints or system resources
- Multiple API keys can be configured per ingress list
- Submitted vCons are automatically stored, indexed, and queued

---

## Chain Management

### Add to Ingress

```
POST /vcon/ingress
```

Adds vCon UUIDs to a processing chain's ingress list.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ingress_list` | string | Name of the ingress list |

**Request Body:**
```json
["uuid-1", "uuid-2", "uuid-3"]
```

**Response:** `204 No Content`

**Example:**
```bash
curl -X POST "http://localhost:8000/api/vcon/ingress?ingress_list=main_chain" \
  -H "x-conserver-api-token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '["uuid-1", "uuid-2"]'
```

---

### Get from Egress

```
GET /vcon/egress
```

Removes and returns vCon UUIDs from a chain's egress list.

**Query Parameters:**

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `egress_list` | string | Name of the egress list | (required) |
| `limit` | int | Maximum UUIDs to remove | `1` |

**Response:** `204 No Content` with body:
```json
["uuid-1", "uuid-2"]
```

**Example:**
```bash
curl -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/vcon/egress?egress_list=processed&limit=10"
```

---

### Count Egress Queue

```
GET /vcon/count
```

Returns the number of vCons in an egress list.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `egress_list` | string | Name of the egress list |

**Response:** `200 OK`
```json
42
```

---

## Configuration

### Get Configuration

```
GET /config
```

Returns the current system configuration from the YAML file.

**Response:** `200 OK` - Returns full configuration as JSON

---

### Update Configuration

```
POST /config
```

Updates the system configuration file.

**Request Body:** Full configuration as JSON

**Response:** `204 No Content`

**Note:** Changes take effect immediately for new chain processing.

---

## Dead Letter Queue

When vCon processing fails, the vCon UUID is moved to a Dead Letter Queue (DLQ). Each ingress list has an associated DLQ named `{ingress_list}:dlq`.

### Get DLQ Contents

```
GET /dlq
```

Returns all vCon UUIDs in a dead letter queue.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ingress_list` | string | Name of the ingress list |

**Response:** `200 OK`
```json
["failed-uuid-1", "failed-uuid-2"]
```

**Example:**
```bash
curl -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/dlq?ingress_list=main_chain"
```

---

### Reprocess DLQ

```
POST /dlq/reprocess
```

Moves all items from a DLQ back to the original ingress list for reprocessing.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ingress_list` | string | Name of the ingress list |

**Response:** `200 OK`
```json
5  // Number of items moved
```

**Example:**
```bash
curl -X POST -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/dlq/reprocess?ingress_list=main_chain"
```

---

## Lifecycle

### Rebuild Search Index

```
GET /index_vcons
```

Rebuilds the search index for all vCons in Redis. Useful after bulk imports or to refresh expired indices.

**Response:** `200 OK`
```json
150  // Number of vCons indexed
```

---

## Redis Caching Behavior

When a vCon is requested but not found in Redis:

1. The API checks each configured storage backend
2. If found, the vCon is stored back in Redis with TTL (`VCON_REDIS_EXPIRY`, default 1 hour)
3. The vCon is added to the sorted set for timestamp-based retrieval
4. Subsequent requests are served from Redis

Configure caching with environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `VCON_REDIS_EXPIRY` | Redis cache TTL in seconds | `3600` (1 hour) |
| `VCON_INDEX_EXPIRY` | Search index TTL in seconds | `86400` (24 hours) |

---

## Error Responses

All endpoints return standard HTTP error codes:

| Code | Description |
|------|-------------|
| `400` | Bad Request - Invalid parameters |
| `403` | Forbidden - Invalid or missing API key |
| `404` | Not Found - Resource doesn't exist |
| `500` | Internal Server Error - Processing failure |

Error response format:
```json
{
  "detail": "Error message describing the issue"
}
```
