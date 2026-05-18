---
description: How an application talks to the Conserver — via the REST API for control, and via storage backends for read paths.
---

# 🔌 Integrating Your App

Most applications integrate with the Conserver in three places:

1. **Send vCons in** through the REST API (`POST /vcon` and `POST /vcon/ingress`).
2. **Get processed vCons out** by configuring a storage backend that your application can read (Postgres, MongoDB, S3, the [vCon MCP server](../mcp-server/README.md), etc.) or by configuring a [webhook](standard-links.md#webhook) at the end of the chain.
3. **Operate** — health checks, queue depth, DLQ inspection, configuration changes — through the system endpoints and `/config` API.

<figure><img src="../.gitbook/assets/Untitled (12).jpg" alt=""><figcaption><p>Application Integration with the Conserver</p></figcaption></figure>

## Two integration patterns

### Pattern A: API-only

Your application talks to the Conserver exclusively via the REST API:

```
your-app ──HTTP──▶ Conserver API ──▶ Redis ──▶ Workers ──▶ Storages
   ▲                                                          │
   └──────────── REST GET /vcon/{uuid} ◀──────────────────────┘
```

Use this when:

- Your application doesn't already have a database you'd want vCons in.
- You want a single source of truth for vCon access (the API enforces auth, logs reads, and orchestrates expiry).
- You're calling from many languages or platforms — REST is the lowest common denominator.

### Pattern B: Direct storage read, API-only writes

Your application reads vCons from a shared database that the Conserver also writes to. Writes still go through the API so the Conserver can index, queue, and track them.

```
your-app ──HTTP write──▶ Conserver API ──▶ Redis ──▶ Workers ──▶ Postgres / Mongo / S3
   ▲                                                                  │
   └────────────── direct read (SQL / Mongo / S3 GET) ◀───────────────┘
```

Use this when:

- You already have a Postgres / Mongo / S3 cluster and want vCons there too.
- You want to query vCons with native database tools (SQL joins, Mongo aggregation pipelines, etc.).
- You want change-data-capture: tap Mongo's oplog, Postgres logical replication, or S3 events instead of webhooks.

## Step-by-step lifecycle

1. **Create.** `POST /vcon` with the full vCon JSON. The vCon is written to Redis under the key `vcon:{uuid}` (Redis JSON). At this point it's stored but unprocessed.
2. **Queue.** `POST /vcon/ingress?ingress_list=<chain_name>` with the UUID. The chain's workers pick it up in FIFO order.
3. **Process.** The chain runs every configured link in sequence. Each link reads the vCon from Redis, modifies it, writes it back. If any link returns `None`, the chain stops for that vCon.
4. **Store.** After the last link, every configured storage runs in parallel (controlled by `CONSERVER_PARALLEL_STORAGE`). The vCon lands in your durable store.
5. **Egress (optional).** If the chain has `egress_lists` configured, the UUID is also pushed onto those lists. Downstream chains or external consumers can pop them via `GET /vcon/egress`.
6. **Expire.** The Redis copy expires per `VCON_REDIS_EXPIRY` (default 1 hour). Subsequent reads via `GET /vcon/{uuid}` fall back to storage and re-populate Redis.
7. **Audit (optional).** If [tracers](conserver-tracers.md) are configured, every link execution is recorded — either via JLINC, DataTrails, SCITT, or your own custom tracer.

## A minimal Python client

```python
import requests

CONSERVER = "http://conserver.internal:8000/api"
TOKEN = "your-api-token"
HEADERS = {"x-conserver-api-token": TOKEN, "Content-Type": "application/json"}

def create_and_process(vcon_json):
    # 1. Create
    r = requests.post(f"{CONSERVER}/vcon", json=vcon_json, headers=HEADERS)
    r.raise_for_status()
    uuid = vcon_json["uuid"]

    # 2. Queue for processing
    requests.post(
        f"{CONSERVER}/vcon/ingress",
        json=[uuid],
        params={"ingress_list": "main_chain"},
        headers=HEADERS,
    ).raise_for_status()

    return uuid

def fetch(uuid):
    r = requests.get(f"{CONSERVER}/vcon/{uuid}", headers=HEADERS)
    r.raise_for_status()
    return r.json()
```

For the full endpoint list and parameters, see [API](api.md).

## When to use webhooks vs storage-side change capture

| Need | Use |
|------|-----|
| Notify a Slack channel when a vCon hits a chain | [`post_analysis_to_slack` link](standard-links.md#post_analysis_to_slack) |
| Push every finished vCon to an HTTP endpoint your app controls | [`webhook` link or storage](standard-links.md#webhook) |
| React to vCons landing in Postgres | Postgres logical replication or `LISTEN/NOTIFY` |
| React to vCons landing in Mongo | Mongo change streams (`db.collection.watch()`) |
| React to vCons landing in S3 | S3 event notifications → SQS / Lambda / EventBridge |
| Subscribe to MCP-server changes | Use the [vCon MCP Server](../mcp-server/README.md) directly |

## Security

- Use a separate API token per integrating system (`CONSERVER_API_TOKEN_FILE` accepts one per line). Rotate them on a schedule.
- For external partners, give them an `ingress_auth` scoped key. Those keys can only write to their assigned ingress list — they can't read, delete, or hit `/config`.
- Run the API tier behind a reverse proxy with TLS terminated outside the conserver container. See [Production Deployment](production-deployment.md) for the nginx pattern.
