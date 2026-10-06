---
description: How to run the conserver in production, covering images, Redis, scaling, the reverse proxy, secrets, metrics and shutdown, as the code behaves today.
---

# 🏭 Production Deployment

The conserver is two processes that share one Redis. The API accepts and serves vCons. The workers run the chains. Redis holds the queues and the working copy of every vCon. This page covers the deployment choices that follow from that. For the file format see [Configuring the Conserver](configuring-the-conserver.md). For symptoms and fixes see [Troubleshooting](troubleshooting.md).

## Images

vcon-server has three Dockerfiles under `docker/`.

| File | Contents | Use |
| ---- | -------- | --- |
| `Dockerfile.api` | FastAPI, uvicorn and the storage client libraries. No audio or ML packages. | The API tier. |
| `Dockerfile.conserver` | The above plus the link dependencies and `ffmpeg` and `sox`. | The worker tier. |
| `Dockerfile` | Everything, plus the test tools. | Development and CI. The checked-in `docker-compose.yml` uses it. |

All three install with `uv` into `/opt/venv`, so a bind mount over `/app` does not hide the packages. Each image starts through `docker/wait_for_redis.sh`, which waits for Redis at `REDIS_URL` and then runs the command. The default commands are `opentelemetry-instrument uvicorn api:app --host 0.0.0.0 --port 8000` for the API and `opentelemetry-instrument python /app/conserver/main.py` for workers. A worker command that says `python main.py` fails, because the working directory is `/app`.

The `pymilvus` package is in none of the production images. If you use the `milvus` storage, add the `storage-milvus` dependency group to your build. Build arguments `VCON_SERVER_VERSION`, `VCON_SERVER_GIT_COMMIT` and `VCON_SERVER_BUILD_TIME` set what `/api/version` and the `X-Vcon-Server-*` headers report.

## Architecture

```
  clients / partners
          |
   reverse proxy (TLS, rate limits)
          |
     API replicas  ---->  Redis  <----  worker containers
                          queues,         (each runs
                          vCon copies      CONSERVER_WORKERS processes)
                                              |
                                   storages: Postgres, S3, ...
```

Nothing but Redis is shared, so API replicas and workers scale independently.

## Redis

Run one Redis primary with persistence and `noeviction`. The conserver stores queued UUIDs and vCon bodies there, and an eviction policy can delete them.

```yaml
redis:
  image: redis:7.4-alpine
  env_file: .env
  command: >
    redis-server --appendonly yes --maxmemory 4gb --maxmemory-policy noeviction
    --requirepass ${REDIS_PASSWORD}
  volumes:
    - redis_data:/data
  healthcheck:
    test: ["CMD-SHELL", "redis-cli -a \"$$REDIS_PASSWORD\" ping | grep PONG"]
    interval: 10s
    timeout: 5s
    retries: 5
```

Set `REDIS_URL=redis://:<password>@redis:6379` on the API and the workers. Plain Redis is enough. The conserver stores a vCon as a JSON string, so it needs no RedisJSON or Redis Stack module.

Do not use Redis Cluster. The conserver connects to one node and uses commands that span keys, including `MGET` on several `vcon:` keys, a script over a vCon key and its DLQ key, and `KEYS`. Sentinel URLs are not supported either. A managed Redis in single-primary mode works.

When Redis reaches `maxmemory` under `noeviction`, writes fail and vCons start going to the dead letter queue. Alert on `conserver.redis.memory_used_bytes` and lower `VCON_REDIS_EXPIRY` if working copies pile up.

## Compose example

This runs the API, the workers and Redis from the production images. Build the two images in your CI from `Dockerfile.api` and `Dockerfile.conserver`, and push them to your registry.

```yaml
services:
  api:
    image: your-registry/vcon-server-api:2026.05.18
    env_file: .env
    environment:
      - OTEL_SERVICE_NAME=api
      - OTEL_EXPORTER_OTLP_ENDPOINT=${OTEL_EXPORTER_OTLP_ENDPOINT:-}
    volumes:
      - ./config.yml:/app/config.yml:ro
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/api/health', timeout=3)"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s
    depends_on:
      redis:
        condition: service_healthy
    networks: [internal]

  conserver:
    image: your-registry/vcon-server-conserver:2026.05.18
    env_file: .env
    environment:
      - OTEL_SERVICE_NAME=conserver
      - OTEL_EXPORTER_OTLP_ENDPOINT=${OTEL_EXPORTER_OTLP_ENDPOINT:-}
    volumes:
      - ./config.yml:/app/config.yml:ro
    deploy:
      replicas: 3
    stop_grace_period: 5m
    depends_on:
      redis:
        condition: service_healthy
    networks: [internal]

  redis:
    # as above
    networks: [internal]

networks:
  internal:

volumes:
  redis_data:
```

The `.env` file holds `REDIS_URL`, `CONSERVER_CONFIG_FILE=/app/config.yml`, `CONSERVER_API_TOKEN_FILE` or `CONSERVER_API_TOKEN`, `CONSERVER_WORKERS`, `CONSERVER_VCON_CONCURRENCY` and `ENV`. The API image does not include `curl`, so the health check uses the Python interpreter already in the image.

The config file holds your provider keys and storage passwords as literal values. The conserver does not read `${VAR}` and does not read `*_FILE` variables, apart from `CONSERVER_API_TOKEN_FILE`. Render `config.yml` at deploy time from your secret store, give it a restrictive mode, and mount it read-only. A read-only mount means `POST /config` fails by design, which is usually what you want. Change the file through your deploy tooling instead.

## Reverse proxy

Terminate TLS in front of the API. The application speaks plain HTTP. All routes are under `/api`.

```nginx
limit_req_zone $binary_remote_addr zone=vcon_api:10m rate=100r/s;

upstream conserver_api {
    least_conn;
    server api:8000;
}

server {
    listen 443 ssl;
    server_name conserver.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    client_max_body_size 50m;

    location /api/ {
        limit_req zone=vcon_api burst=200 nodelay;
        proxy_pass http://conserver_api;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
    }
}
```

Put this file in `conf.d`, where it is included at the `http` level, so the `limit_req_zone` line is valid. Nginx limits request bodies to 1 MB by default, and a vCon with inline audio is larger, so set `client_max_body_size` for your payloads.

If you accept vCons from partners but do not want the main API on the internet, expose only the partner route, and keep the rest on the internal network:

```nginx
location = /api/vcon/external-ingress {
    proxy_pass http://conserver_api;
}
```

`GET /api/config` returns the whole configuration, secrets included, to anyone with the main token. `GET /api/stats/queue` and `GET /api/health` need no token.

## Scaling

The most vCons in flight is `CONSERVER_WORKERS` times `CONSERVER_VCON_CONCURRENCY` times the number of worker containers.

* Raise `CONSERVER_WORKERS` to use more CPU cores. Each worker is its own process.
* Raise `CONSERVER_VCON_CONCURRENCY` when chains wait on the network, for transcription, LLM calls, webhooks or storage. It runs several vCons in threads inside one worker.
* Add worker containers to spread load over hosts. Workers pop from the same Redis lists, so no coordination is needed.
* Add API replicas for request volume. The API keeps no state.

Workers wait on the ingress lists with a 15 second timeout, so an idle worker polls Redis about four times a minute. Watch `conserver.ingress_list.length`, or call `GET /api/stats/queue?list_name=<list>`, and scale on it.

Each ingress list is served by one chain. A worker re-reads `config.yml` on every pass, so chains and links change without a restart. Environment variables do not.

## Security

* **API token.** `CONSERVER_API_TOKEN_FILE` takes one token per line and is read when the API starts. To rotate, add the new token, restart the API, move clients over, then remove the old one and restart again.
* **Partner keys.** Give each partner its own key in `ingress_auth`. A key opens one ingress list and nothing else.
* **Networks.** Keep Redis and the workers on a network the proxy cannot reach, and set a Redis password.
* **Logs.** The `mongo` storage logs its options, including a password inside the connection URL. Keep credentials out of the URL, or restrict log access.

## Monitoring

### Health

```bash
curl http://localhost:8000/api/health
curl http://localhost:8000/api/version
curl "http://localhost:8000/api/stats/queue?list_name=incoming_calls"
redis-cli -a "$REDIS_PASSWORD" ping
```

`/api/health` reports that the API process is up. It does not test Redis.

### Metrics

The containers run under `opentelemetry-instrument`. Set `OTEL_EXPORTER_OTLP_ENDPOINT` to an OTLP collector, plus `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_EXPORTER_OTLP_HEADERS` and `OTEL_EXPORTER_OTLP_INSECURE` as your backend needs. The `.env.example` file shows settings for Langfuse and for an OTLP collector. With no endpoint, nothing is exported.

The conserver's own metrics, built in `common/lib/metrics.py`, go through a gRPC exporter at that endpoint whatever `OTEL_EXPORTER_OTLP_PROTOCOL` says. Point a collector with a gRPC receiver at them.

| Metric | Type | Attributes |
| ------ | ---- | ---------- |
| `conserver.main_loop.count_vcons_received` | counter | `ingress_list` |
| `conserver.main_loop.count_vcons_processed` | counter | `chain.name` |
| `conserver.main_loop.vcon_processing_time` | histogram, seconds | `chain.name` |
| `conserver.vcons.inflight` | up-down counter | `chain.name` |
| `conserver.link.count` | counter | `link_name`, `outcome` (`success`, `halt` or `error`) |
| `conserver.link.execution_time` | histogram, seconds | `link.name`, `chain.name` |
| `conserver.storage.count` | counter | `backend`, `outcome` (`success` or `error`) |
| `conserver.storage.duration_ms` | histogram, milliseconds | `backend`, `outcome` |
| `conserver.dlq.count` | counter | `queue_name` (`DLQ:<list>` or `DLQ:storage:<name>`) |
| `conserver.ingress_list.length` | gauge | `ingress_list`, `kind` (`ingress` or `dlq`) |
| `conserver.redis.memory_used_bytes` | gauge | none |
| `conserver.api.count_vcons_enqueued` | counter | `ingress_list`, `source` (`new`, `external`, `reingress`, `dlq_reprocess`) |
| `conserver.lib.vcon_redis.get_vcon_redis_miss`, `...get_vcon_storage_hit`, `...get_vcon_not_found` | counters | |
| `conserver.webhook.duration` | histogram | `status_code` |

Most links add their own counters and histograms under `conserver.link.<family>.*`, such as `conserver.link.openai.analysis_time` and `conserver.link.deepgram.transcription_failures`. [Standard Links](standard-links.md) names the behavior behind each.

The core metrics above carry no vCon identifier, because a per-vCon attribute would create one time series for every vCon. Many of the per-link metrics do add `vcon.uuid` as an attribute. Drop it in your collector before it reaches a metrics backend. Per-vCon detail belongs in traces: each vCon gets a `vcon_processing.<chain>` span, with a `link.<name>` child for each link and a `storage.<name>` child for each write. The root span carries the vCon UUID and, when the producer sent trace context, a span link to the producer's trace.

`conserver.ingress_list.length` covers ingress lists and their ingress DLQs. It does not cover storage DLQs. Read those with `GET /api/dlq/storage?storage_name=<name>`, or `LLEN DLQ:storage:<name>` in Redis.

### Logs

Logs go to standard output as JSON from `common/logging.conf`. To change the format or levels, point `LOGGING_CONFIG_FILE` at your own `fileConfig` file, and mount it into the container. `LOG_FORMAT` and `LOG_LEVEL` are not read. Set `SENTRY_DSN` and `ENV` to send errors to Sentry.

### Alerts

| Signal | Condition | Action |
| ------ | --------- | ------ |
| `conserver.dlq.count` | any increase | Read the DLQ, fix the cause, replay |
| `conserver.ingress_list.length` | rising for several minutes | Add workers or raise `CONSERVER_VCON_CONCURRENCY` |
| `conserver.link.count{outcome="error"}` | above your baseline | Read the logs for the failing link |
| `conserver.storage.count{outcome="error"}` | any | Check the backend, then replay its DLQ |
| `conserver.redis.memory_used_bytes` | near `maxmemory` | Raise `maxmemory` or shorten expiries |
| `conserver.vcons.inflight` | flat at the maximum | Chains are slow or stuck, check the slowest link |

## Shutdown

On `SIGTERM` or `SIGINT` a worker stops taking new work. A vCon it popped but has not started goes back to the head of its list. A vCon already running finishes its chain first. A worker may be waiting on Redis for up to 15 seconds, so shutdown can take that long even when idle.

With one worker (`CONSERVER_WORKERS=1`) the worker runs in the main process and the shutdown waits for the chain to end. With more than one, the main process gives each worker 30 seconds, then terminates it. A vCon that is mid-chain at that point has already left the ingress list and is not put back. Its working copy stays in Redis until it expires, so resubmit it with `POST /api/vcon/ingress`. Set `stop_grace_period` above the longest chain you run, and run chains that outlast 30 seconds with a single worker per container.

## Backup and recovery

* **Redis.** Use AOF (`--appendonly yes`) and take RDB snapshots with `BGSAVE`. Queued UUIDs and vCons that are not yet stored exist only here.
* **Storages.** Back up each one the usual way: `pg_dump` for Postgres, versioning on S3, snapshots for Elasticsearch.
* **Configuration.** Keep a template of `config.yml`, without secrets, in version control, and keep the secrets in your secret store. `GET /api/config` returns JSON without comments and includes every secret, so it is a poor backup.
* **Replay.** After an outage, replay `DLQ:<list>` with `POST /api/dlq/reprocess` and each `DLQ:storage:<name>` with `POST /api/dlq/storage/reprocess`. See [API](api.md#dead-letter-queues).

## Checklist

Before you deploy:

* [ ] Redis runs with `--appendonly yes` and `--maxmemory-policy noeviction`, and a password.
* [ ] `config.yml` is rendered from your secret store, mounted read-only, and not in version control.
* [ ] `CONSERVER_API_TOKEN` or `CONSERVER_API_TOKEN_FILE` is set.
* [ ] The proxy terminates TLS and sets `client_max_body_size`.
* [ ] Each storage's credentials work, and the milvus build includes `pymilvus` if you use it.
* [ ] A collector receives the metrics, and alerts exist for DLQ growth and Redis memory.

After you deploy:

* [ ] `GET /api/health` and `GET /api/version` return the expected build.
* [ ] A test vCon with a lawful basis attachment moves through each chain, reaches each storage, and appears on the egress list.
* [ ] `docker compose stop conserver` returns inside `stop_grace_period`.
