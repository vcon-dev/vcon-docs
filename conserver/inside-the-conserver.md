---
description: Shows how the conserver runs internally, its processes, Redis keys and processing loop, so you can size, debug and extend a deployment.
---

# ❤️ Inside the Conserver

This page describes the runtime as of [vcon-server main](https://github.com/vcon-dev/vcon-server). The rules a chain follows (link return values, egress, storage, dead letter queues, tracer timing) are defined once in [Concepts](concepts.md). This page covers the machinery underneath them.

<figure><img src="../.gitbook/assets/Conserver Internals (4).jpg" alt="A chain: UUIDs from an ingress list pass through three links, each reading the vCon from Redis, then go to an egress list and storage"><figcaption><p>A chain. Only the UUID moves between links. Each link fetches the full vCon from Redis.</p></figcaption></figure>

## Two kinds of process

A deployment runs two programs against one Redis.

The **API** ([api/api.py](https://github.com/vcon-dev/vcon-server/blob/main/api/api.py)) is a FastAPI application served under `/api` by default (`API_ROOT_PATH`). It writes vCons into Redis, pushes UUIDs onto ingress lists, pops egress lists, replays dead letter queues and reads and writes `config.yml`. It runs no links. See [API](api.md).

The **conserver** ([conserver/main.py](https://github.com/vcon-dev/vcon-server/blob/main/conserver/main.py)) runs the chains. It starts `CONSERVER_WORKERS` worker processes (default 1). With one worker the loop runs in the main process; with more, each worker is a separate `multiprocessing.Process`, started with the method in `CONSERVER_START_METHOD` (`fork`, `spawn` or `forkserver`, platform default when unset). Inside each worker, `CONSERVER_VCON_CONCURRENCY` (default 1) sets how many vCons that worker processes at once on a thread pool. The main process also starts any configured [followers](concepts.md#follower) on timer threads.

Workers share nothing but Redis, so scaling out means adding workers or containers. See [Production Deployment](production-deployment.md) for sizing.

## The worker loop

Each worker repeats the same loop ([`worker_loop`](https://github.com/vcon-dev/vcon-server/blob/main/conserver/main.py)):

1. Re-read `config.yml` from disk. Configuration is refreshed on every iteration, so a change written by `POST /config` or by hand reaches each worker on its next pass without a restart. The API and the workers must see the same file.
2. Build a map from every chain's ingress lists to that chain.
3. Call Redis `BLPOP` on all ingress lists at once with a 15 second timeout. If nothing arrives, go back to step 1. This is also why a config change can take up to 15 seconds to be noticed on an idle conserver.
4. Run the pre-processing hook. The default [`hook.py`](https://github.com/vcon-dev/vcon-server/blob/main/conserver/hook.py) passes every vCon through. A replacement can return a falsy value to skip a vCon without dead-lettering it.
5. Run the chain for that UUID, as described in [Concepts](concepts.md#chain). With `CONSERVER_VCON_CONCURRENCY` above 1, the chain runs on the thread pool and the worker goes back to `BLPOP` once a slot is free.
6. If the chain raised, push the UUID onto `DLQ:<ingress_list>` and extend the vCon's TTL to `VCON_DLQ_EXPIRY`.
7. Run the post-processing hook, which is a no-op by default.

`BLPOP` hands each UUID to exactly one waiting worker, so two workers never process the same pop. There is no pub/sub channel and no timer-driven scheduling. Work starts when a UUID lands on an ingress list.

Inside a chain run, egress lists are written before storages ([`_wrap_up`](https://github.com/vcon-dev/vcon-server/blob/main/conserver/main.py)). Storage writes run in parallel threads when the chain has more than one storage and `CONSERVER_PARALLEL_STORAGE` is true (the default). A failed storage write goes to `DLQ:storage:<name>` and does not affect the other storages.

## A link at work

<figure><img src="../.gitbook/assets/Conserver Internals (1).jpg" alt="A link receives a vCon UUID and settings from config.yml, fetches the vCon from Redis, calls an external service, and returns the UUID or None"><figcaption><p>A link takes a UUID and its options, reads the vCon from Redis, and returns a UUID or None.</p></figcaption></figure>

For each link in the chain the worker looks up the link's entry under `links:`, imports its module once per process (installing it with pip first if a `pip_name` is configured and the import fails), and calls `run(vcon_uuid, link_name, options)`. Tracers run before the first link and after each one that returns. A no-op [`after_link_hook.py`](https://github.com/vcon-dev/vcon-server/blob/main/conserver/after_link_hook.py) is also called after every link, on success and on error, for deployments that replace it at build time.

The legacy transcription modules `links.deepgram_link`, `links.openai_transcribe`, `links.groq_whisper` and `links.hugging_face_whisper` are redirected to `links.transcribe` with the matching `vendor` option, and a deprecation warning is logged once per module.

The [analyze link](https://github.com/vcon-dev/vcon-server/blob/main/conserver/links/analyze/__init__.py) is a typical example. It merges its `default_options` with the options from `config.yml`, loads the vCon from Redis, skips dialogs that already carry an analysis of the configured type, calls the model with the configured prompt for the rest, adds the result to `analysis[]`, writes the vCon back to Redis and returns the UUID. Writing your own is covered in [Creating Custom Links](creating-custom-links.md).

## What lives in Redis

| Key | Type | Written by | Purpose |
| --- | ---- | ---------- | ------- |
| `vcon:{uuid}` | String (JSON) | API, links, followers | The working copy of each vCon |
| Each ingress and egress list name | List | API, chains, followers | UUIDs waiting to be processed or collected |
| `DLQ:<ingress_list>` | List | Workers | UUIDs whose chain raised |
| `DLQ:storage:<storage_name>` | List | Workers | UUIDs whose write to one storage failed |

vCons are stored as plain JSON strings through ordinary `SET` and `GET`. RedisJSON is not required. Deployments upgrading from a release that used RedisJSON must migrate their data first; see [the Redis migration guide](https://github.com/vcon-dev/vcon-server/blob/main/docs/installation/redis-migration.md). The API also keeps a sorted set of vCons by creation time and short-lived party indexes for search.

Links, chains and storages are not stored in Redis. They exist only in `config.yml`, which every worker re-reads on each loop.

Redis must run with `maxmemory-policy noeviction`. Under any eviction policy Redis can silently drop queued UUIDs or the vCons they point to.

## Tech stack

Python 3.12, FastAPI for the API, ordinary Redis for state and queues (the Compose file pins 7.4), and the [vCon Python library](../vcon-library/README.md) for reading and writing vCons. The repository ships a Docker Compose file that runs the API, the conserver and Redis, along with Postgres, Elasticsearch and Langfuse services. See [Quick Start](conserver-quick-start.md).
