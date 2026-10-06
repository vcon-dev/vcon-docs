---
description: The conserver's primitives and the exact rules a chain follows, so you can predict what happens to every vCon you send it.
---

# 🏫 Concepts

Everything you configure in `config.yml` is one of the primitives on this page. This is the one place the chain rules are written down in full. Other pages link here rather than repeat them.

## vCon

The unit of work is one vCon, identified by its UUID. The conserver keeps a working copy in Redis under the key `vcon:{uuid}`, stored as a plain JSON string. Links, storages and tracers all receive the UUID and read the vCon from Redis themselves. The vCon itself is never passed between processes.

The working copy expires after `VCON_REDIS_EXPIRY` seconds (default 3600) when it arrives through the API. `GET /vcon/{uuid}` and `POST /vcon/ingress` reload a missing vCon from the configured storages.

## Link

A link is a Python module that does one thing to a vCon: transcribe, summarize, tag, redact, route, notify. The conserver ships 23 links (see [Standard Links](standard-links.md)) and you can write your own (see [Creating Custom Links](creating-custom-links.md)).

Every link exposes the same function:

```python
def run(vcon_uuid: str, link_name: str, opts: dict) -> str | None
```

What a link returns decides what happens next:

| Return value | Effect |
| ------------ | ------ |
| The same UUID | The chain continues to the next link. |
| A different UUID | The chain continues with the new vCon. Later links, egress and storage all use the new UUID. This is how a link hands on a derived vCon, such as a redacted copy. |
| `None` or any falsy value | The chain stops for this vCon. No later link runs, nothing goes to egress and nothing is stored. This is how `sampler`, `jq_link` and `tag_router` (with `forward_original: false`) filter. |
| An exception | The chain stops and the UUID goes to the dead letter queue for its ingress list. |

A link instance is a named entry under `links:` that points at a module and carries options. The same module can appear many times under different names with different options, and one link instance can be used in many chains.

## Chain

A chain is an ordered list of links plus the lists and storages around it:

```yaml
chains:
  main:
    ingress_lists: [default_ingress]
    links: [deepgram, summarize]
    storages: [postgres, s3]
    egress_lists: [main_done]
```

For each vCon, a chain runs in this order:

1. A worker pops the UUID from one of the chain's ingress lists.
2. Every configured tracer runs once with link index `-1`.
3. Each link runs in order. After each link returns, every tracer runs again with that link's index. A link that raises gets no tracer call.
4. If no link stopped the chain, the UUID is pushed onto every egress list.
5. The vCon is written to every storage. With more than one storage and `CONSERVER_PARALLEL_STORAGE=true` (the default) the writes run in parallel threads; otherwise they run in order.

Egress happens before storage. A consumer reading an egress list can see the UUID before the storage writes have finished, and a storage failure does not take the UUID back off the egress list.

Each ingress list belongs to one chain. If two chains name the same ingress list, the one defined last in `config.yml` wins.

The chain keys `timeout` and `enabled` are parsed but not enforced. A long link runs to completion, and a chain with `enabled: 0` still processes vCons. To stop a chain, remove it from `config.yml`.

## Ingress and egress lists

Ingress and egress lists are ordinary Redis lists of UUIDs. Ingress lists feed chains. Egress lists hold UUIDs a chain has finished with. Point one chain's egress list at another chain's ingress list and you have a pipeline of chains. External consumers pop egress lists with `GET /vcon/egress`.

Lists are durable buffering. A burst of arrivals waits in the list until a worker is free. Redis must run with `maxmemory-policy noeviction` so queued UUIDs and vCons are never evicted under memory pressure.

## Storage

A storage is a durable destination for finished vCons. The conserver ships 15 storage modules: `chatgpt_files`, `dataverse`, `elasticsearch`, `file`, `milvus`, `mongo`, `postgres`, `redis_storage`, `s3`, `scitt`, `sftp`, `spaceandtime`, `utopia`, `vcon_mcp` and `webhook`. Each chain names its own storages, so different chains can write to different places. See [Storage](storage.md).

A storage write that fails does not fail the chain. The UUID goes onto that storage's own dead letter queue so the write alone can be retried.

## Dead letter queues

There are two kinds, and they replay different things.

| Queue | Filled when | Replay | What a replay does |
| ----- | ----------- | ------ | ------------------ |
| `DLQ:<ingress_list>` | A link, or a tracer with `dlq_vcon_on_error: true`, raises | `POST /dlq/reprocess?ingress_list=<list>` | Moves the UUIDs back onto the ingress list, so the whole chain runs again |
| `DLQ:storage:<storage_name>` | One storage write raises | `POST /dlq/storage/reprocess` | Retries that storage write only |

When a vCon is dead-lettered its Redis TTL is extended to `VCON_DLQ_EXPIRY` (default 604800 seconds, seven days) so the body is still there to replay. Inspect the queues with `GET /dlq` and `GET /dlq/storage`. See [API](api.md).

## Tracer

A tracer records what happened to a vCon without changing it. It runs at the two points described under [Chain](#chain): once before the first link, and after each link that returns. There is no separate call when the chain finishes. A tracer that raises is logged and ignored, unless its options set `dlq_vcon_on_error: true`, in which case the vCon is dead-lettered.

One tracer ships: `jlinc`. DataTrails and SCITT are available as links, and SCITT also as a storage, not as tracers. See [Conserver Tracers](conserver-tracers.md).

## Follower

A follower lets one conserver pull work from another. On a timer it calls the remote conserver's `GET /vcon/egress`, fetches each vCon by UUID, writes it into local Redis and pushes it onto a local ingress list. Use it to move vCons between conservers across a network or trust boundary. See the `followers:` section of [Configuring the Conserver](configuring-the-conserver.md).
