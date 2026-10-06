---
description: Every route of the conserver REST API, with authentication, parameters, status codes and the Redis behavior behind each.
---

# 🧩 API

The conserver API is a FastAPI application. It stores and fetches vCons, feeds and drains chains, reads and writes the configuration, and manages the dead letter queues.

## Base URL and headers

Every route, including the system routes, sits under `API_ROOT_PATH` (default `/api`). With the compose file, the API listens on port 8000, so the health route is `http://localhost:8000/api/health`.

Every response carries `X-Vcon-Server-Version` and `X-Vcon-Server-Commit`, taken from the image build arguments. Both read `dev` or `unknown` on an image built without them. CORS is open to all origins.

## Authentication

Two separate models apply, and a key from one is not accepted by the other.

| Routes | Credential |
| ------ | ---------- |
| All routes below except those named next | The main token in the `x-conserver-api-token` header, or the header named by `CONSERVER_HEADER_NAME`. |
| `GET /version`, `GET /health`, `GET /stats/queue` | None. |
| `POST /vcon/external-ingress` | A key from `ingress_auth` in `config.yml`, scoped to one ingress list. |

Set `CONSERVER_API_TOKEN`, or `CONSERVER_API_TOKEN_FILE` with one token per line. With neither set, the main API accepts every request. A wrong or missing token returns `403` with `{"detail": "Invalid API Key"}`.

```bash
curl -H "x-conserver-api-token: $TOKEN" "http://localhost:8000/api/vcon"
```

## Errors

| Code | Meaning |
| ---- | ------- |
| `400` | `GET /vcons/search` called with no search parameter. |
| `403` | Bad or missing key. External ingress adds the reason to `detail`. |
| `404` | Unknown vCon, or an unknown storage name on a storage DLQ replay. |
| `409` | A storage DLQ replay is already running, or a stale replay lock needs recovery. |
| `422` | The request body or a query parameter failed validation. FastAPI returns a `detail` list that names the failing field. |
| `500` | A Redis or storage error. `detail` is a short fixed message. The cause is in the server log. |

```json
{ "detail": "vCon not found" }
```

## vCon validation

`POST /vcon` and `POST /vcon/external-ingress` parse the body with a model that requires `vcon` (string), `uuid` and `created_at`, and accepts extra fields. It returns `422` when:

* `dialog[].url` is not empty and does not look like a URL (`scheme://...`).
* `dialog[].duration` is negative.
* `dialog[].mimetype` is present and not a valid media type, or `dialog[].alg` is not a known algorithm.
* `parties[].tel` is not empty and does not look like a phone number.
* a `dialog[].parties` index is outside the `parties` array.

The model fills `redacted`, `group` and `appended` with empty defaults, so the copy the API stores in Redis can carry them. The first link that stores the vCon drops an empty `group` and `redacted` and stamps `vcon` as `0.4.0`.

The conserver does not require or check a lawful basis. A vCon you send should carry an attachment with `purpose: "lawful_basis"`, as in [Quick Start](conserver-quick-start.md#send-a-vcon). See the [Lawful Basis extension](../extensions/lawful-basis.md).

## System routes

### Version

```
GET /version
```

```json
{ "version": "2026.05.18", "git_commit": "5bc6b6e", "build_time": "2026-05-18T10:00:00Z" }
```

### Health

```
GET /health
```

Returns `200` with `{"status": "healthy", "version": {...}}`. It does not touch Redis, so it shows the API process is up, not that Redis is reachable.

### Queue depth

```
GET /stats/queue?list_name=<redis-list>
```

Returns `{"list_name": "incoming_calls", "depth": 127}` for any Redis list name. It needs no token, so use it for autoscalers and dashboards, and keep the route off the public internet.

## vCons

### Create a vCon

```
POST /vcon?ingress_lists=<list>&ingress_lists=<list>
```

Stores the body in Redis with a TTL of `VCON_REDIS_EXPIRY` seconds, adds it to the `vcons` sorted set and indexes its parties. If you pass `ingress_lists`, it also pushes the UUID onto each list, with the caller's trace context stored first so the worker can link its spans. It returns `201` and the stored vCon. The body is not written to any storage until a chain does it.

```bash
curl -X POST "http://localhost:8000/api/vcon?ingress_lists=main_chain" \
  -H "x-conserver-api-token: $TOKEN" -H "Content-Type: application/json" \
  -d @vcon.json
```

### Get a vCon

```
GET /vcon/{vcon_uuid}
```

Reads `vcon:{uuid}` from Redis. On a miss it asks each configured storage in turn, restores the first hit into Redis with `VCON_REDIS_EXPIRY`, and adds it to the sorted set. `404` if no storage has it. If `EGRESS_FORMAT_VERSION` is set, the response uses that legacy shape.

### Get several vCons

```
GET /vcons?vcon_uuids=<uuid>&vcon_uuids=<uuid>
```

Returns a JSON array in request order. A UUID that is in neither Redis nor storage comes back as `null` in its position.

### List vCon UUIDs

```
GET /vcon?page=1&size=50&since=<datetime>&until=<datetime>
```

Returns UUIDs from the sorted set, newest first by `created_at`. `since` and `until` filter on that timestamp.

### Search by party

```
GET /vcons/search?tel=<number>&mailto=<address>&name=<name>
```

Returns matching UUIDs. At least one parameter is required, or the route returns `400`. Matches are exact. The index is written when a vCon arrives through `POST /vcon` or `POST /vcon/external-ingress`, and its keys expire after `VCON_INDEX_EXPIRY`. Rebuild it with `GET /index_vcons`. When you pass several parameters the route intersects the sets that matched, and a parameter with no match is ignored instead of emptying the result.

### Delete a vCon

```
DELETE /vcon/{vcon_uuid}
```

Deletes `vcon:{uuid}` from Redis, then calls `delete` on every storage in `config.yml`. It always returns `204`, including when a delete failed, and logs each failure. The storages that implement delete are `s3`, `postgres`, `file`, `elasticsearch`, `vcon_mcp` and `utopia`. The others are skipped with a warning. The call leaves the sorted set entry and the search index keys in place.

## Chains

### Add UUIDs to an ingress list

```
POST /vcon/ingress?ingress_list=<list>
```

Body: a JSON array of UUIDs. Each vCon must exist in Redis or in a storage. The route restores a stored one into Redis, skips a missing one with a warning, and pushes the rest. It returns `204` with no body, whether or not it skipped any. This route takes only the main token. A partner key does not work here.

```bash
curl -X POST "http://localhost:8000/api/vcon/ingress?ingress_list=main_chain" \
  -H "x-conserver-api-token: $TOKEN" -H "Content-Type: application/json" \
  -d '["550e8400-e29b-41d4-a716-446655440000"]'
```

### External ingress

```
POST /vcon/external-ingress?ingress_list=<list>
```

Body: one full vCon. This is the route for a partner system, and it is the only route that does not take the main token. The partner sends a key from `ingress_auth` in the same header:

```yaml
ingress_auth:
  partner_data:
    - "partner-key-1"
    - "partner-key-2"
  customer_data: "single-key"
```

The route stores, indexes and enqueues the vCon exactly as `POST /vcon` does, onto the one list named in the query, and returns `204`. The key opens only that list. The conserver reads `ingress_auth` from the file on each call. A `403` carries one of these reasons in `detail`: `API Key required`, `No ingress authentication configured`, `Ingress list '<name>' not configured`, `Invalid API Key for ingress list '<name>'`.

```bash
curl -X POST "http://localhost:8000/api/vcon/external-ingress?ingress_list=partner_data" \
  -H "x-conserver-api-token: partner-key-1" -H "Content-Type: application/json" \
  -d @vcon.json
```

### Take UUIDs from an egress list

```
GET /vcon/egress?egress_list=<list>&limit=1
```

Removes up to `limit` UUIDs from the list and returns them as a JSON array with status `200`. Despite being a `GET`, it changes state: a UUID you read is gone from the list. It pops from the tail, which is where the conserver appends, so with a small `limit` you get the newest UUIDs first. Fetch each vCon with `GET /vcon/{uuid}`.

### Count an egress list

```
GET /vcon/count?egress_list=<list>
```

Returns the list length as a bare number.

## Dead letter queues

Two kinds exist. The ingress DLQ `DLQ:<ingress_list>` holds a UUID whose chain raised, and replaying it runs the whole chain again. The storage DLQ `DLQ:storage:<storage_name>` holds a UUID whose write to one storage raised, and replaying it retries only that write. [Concepts](concepts.md#dead-letter-queues) has the full rules.

### Read a DLQ

```
GET /dlq?ingress_list=<list>
GET /dlq/storage?storage_name=<name>
```

Each returns every UUID in the queue, oldest first.

### Replay an ingress DLQ

```
POST /dlq/reprocess?ingress_list=<list>&count=1000
```

Moves up to `count` UUIDs, oldest first, from `DLQ:<list>` back onto the ingress list. `count` is 1 to 100000 and defaults to 1000, so a single call stays short on a large queue. It returns the number moved. Call it again until it returns `0`.

### Replay a storage DLQ

```
POST /dlq/storage/reprocess?storage_name=<name>&count=1000
```

Calls `save` on the named storage for up to `count` UUIDs from `DLQ:storage:<name>`, and returns how many succeeded. A UUID leaves the queue only after its write succeeds. The route stops at the first failure, so a storage that is still down does not drain the queue. A second call while one runs returns `409`. An unknown storage name returns `404`.

Replay is at least once: a crash between the write and the removal repeats the write, so the storage has to accept a second save of the same UUID. The replay lock, `DLQ:storage:<name>:replay-lock`, has no expiry. If the API crashes mid-replay, delete that key in Redis to recover.

A vCon in either queue has its Redis TTL raised to `VCON_DLQ_EXPIRY` so the body survives until replay.

## Configuration

### Read the configuration

```
GET /config
```

Returns the parsed `config.yml` as JSON. It includes every key and password in the file.

### Replace the configuration

```
POST /config
```

Body: the whole configuration as JSON. The route writes it to the path in the `CONSERVER_CONFIG_FILE` environment variable with `yaml.dump`, which drops comments and formatting, and returns `204`. It returns `500` if the variable is unset or the file is read-only. Workers pick the new file up on their next vCon. The conserver does not validate the content.

### Rebuild the search index

```
GET /index_vcons
```

Scans every `vcon:*` key, rebuilds the party index for each, and returns the count. The scan uses `KEYS`, which blocks Redis while it runs, so avoid it on a large instance.
