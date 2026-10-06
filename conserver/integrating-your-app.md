---
description: Shows how an application sends vCons to the conserver, gets processed results back out, and which credentials each side should hold.
---

# 🔌 Integrating Your App

An application meets the conserver at three points. It sends vCons in through the REST API. It gets processed vCons out from a storage it can read, a webhook, or an egress list. It operates the conserver through the queue, DLQ and config endpoints. The full endpoint list is in [API](api.md). How a chain processes each vCon is in [Concepts](concepts.md#chain).

<figure><img src="../.gitbook/assets/Untitled (12).jpg" alt="External vCons enter the conserver, which uses Redis, sends webhooks to a web application and writes to MongoDB and S3"><figcaption><p>Application integration with the conserver</p></figcaption></figure>

## Ways to submit a vCon

All paths below are relative to the API root, `/api` by default.

| Caller | Call | What it does |
| ------ | ---- | ------------ |
| Your own service, holding a main API token | `POST /vcon?ingress_lists=<list>` | Stores the full vCon in Redis and pushes its UUID onto each named ingress list. One call. Repeat `ingress_lists` for more than one list. |
| An external partner, holding a key scoped in `ingress_auth` | `POST /vcon/external-ingress?ingress_list=<list>` | Same, with the full vCon as the body, but the key works only for its assigned list and only on this endpoint. Returns 204. |
| Your own service, re-running vCons already known to the conserver | `POST /vcon/ingress?ingress_list=<list>` with a JSON array of UUIDs | Pushes existing UUIDs onto a list, reloading any vCon missing from Redis from the configured storages first. |

`POST /vcon` without `ingress_lists` stores the vCon but does not process it.

## Ways to read processed vCons

The vCon in Redis is a working copy. It expires after `VCON_REDIS_EXPIRY` seconds (default 3600), and `GET /vcon/{uuid}` reloads it from storage when it is missing. For anything durable, read from a storage the chain writes to.

**Read through the API.** `GET /vcon/{uuid}` returns one vCon. Use this when your application has no database of its own for vCons or calls from many languages.

**Read the storage directly.** Configure the chain to write to Postgres, MongoDB, S3 or another [storage](storage.md) your application can query, and read it with native tools: SQL, Mongo aggregation, S3 events, Postgres logical replication or Mongo change streams. Writes still go through the API so the conserver can queue and process them.

**Get pushed.** Put the [`webhook` link](standard-links.md) or `webhook` storage at the end of a chain to POST each finished vCon to an endpoint you control, or `post_analysis_to_slack` to notify a channel.

**Pop an egress list.** `GET /vcon/egress?egress_list=<list>&limit=<n>` removes and returns up to `n` UUIDs. It pops from the tail of the list, so the most recently finished UUIDs come back first. Remember that the UUID is pushed to egress before storage writes complete.

**Ask in plain language.** Point the `vcon_mcp` storage at a [vCon MCP server](../mcp-server/README.md) and AI assistants can search and read the processed vCons.

## A minimal Python client

```python
import requests

CONSERVER = "http://conserver.internal:8000/api"
HEADERS = {"x-conserver-api-token": "your-api-token"}

def submit(vcon_json: dict) -> str:
    """Store a vCon and queue it on the main chain in one call."""
    r = requests.post(
        f"{CONSERVER}/vcon",
        params={"ingress_lists": "main_chain"},
        json=vcon_json,
        headers=HEADERS,
    )
    r.raise_for_status()
    return r.json()["uuid"]

def fetch(uuid: str) -> dict:
    r = requests.get(f"{CONSERVER}/vcon/{uuid}", headers=HEADERS)
    r.raise_for_status()
    return r.json()
```

The vCon you submit should carry its [lawful basis](../extensions/lawful-basis.md) attachment (`purpose: "lawful_basis"`) from the adapter that created it. The conserver does not add one.

## Security

- If neither `CONSERVER_API_TOKEN` nor `CONSERVER_API_TOKEN_FILE` is set, the main API runs with no authentication. Always set one in production.
- `CONSERVER_API_TOKEN_FILE` holds one token per line. Give each integrating system its own token so you can revoke it alone. Every main token has full access, including `DELETE /vcon/{uuid}` and `POST /config`.
- Give external partners an `ingress_auth` key instead. It can call only `POST /vcon/external-ingress`, and only for its list. See [Configuring the Conserver](configuring-the-conserver.md).
- Terminate TLS in a reverse proxy in front of the API. See [Production Deployment](production-deployment.md).
