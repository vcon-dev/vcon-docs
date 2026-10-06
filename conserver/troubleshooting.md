---
description: Symptoms, causes and fixes for the problems operators hit when running the conserver, checked against how vcon-server behaves.
---

# 😥 Troubleshooting

Commands below use the service names in the checked-in `docker-compose.yml`: `api`, `conserver` and `redis`. Rename them if your compose file differs. For how chains, links and queues behave, see [Concepts](concepts.md).

## Start with these checks

```bash
# Containers and the build that is running
docker compose ps
curl http://localhost:8000/api/version

# The API process
curl http://localhost:8000/api/health

# Redis
docker compose exec redis redis-cli ping

# Queues
docker compose exec redis redis-cli LLEN incoming_calls
docker compose exec redis redis-cli LLEN DLQ:incoming_calls
docker compose exec redis redis-cli LLEN DLQ:storage:postgres

# Logs
docker compose logs --tail 200 conserver
docker compose logs --tail 200 api
```

`/api/health` returns `healthy` whenever the API process runs, even with Redis down. Use `redis-cli ping` for Redis.

## Connection problems

### The containers cannot reach Redis

**Symptoms:** `Redis not ready yet. Retrying...` repeats in the log, or requests return `500`.

1. Check `REDIS_URL`. In the compose network it is `redis://redis`. With a password, it is `redis://:<password>@redis:6379`.
2. The compose file declares the network `conserver` as external. If `docker compose up` says the network is missing, run `docker network create conserver`.
3. Check Redis memory with `redis-cli INFO memory`. Under `maxmemory-policy noeviction`, a full Redis refuses writes with an `OOM command not allowed` error. Raise `maxmemory`, lower `VCON_REDIS_EXPIRY`, or drain a backed-up queue. Do not switch to an eviction policy, because it can delete queued UUIDs and vCons.

### The API returns 403

**Symptoms:** `{"detail": "Invalid API Key"}` or another `403`.

1. The header must be `x-conserver-api-token`, or the name in `CONSERVER_HEADER_NAME`.
2. Compare the token with `CONSERVER_API_TOKEN`, or with a line of the file named by `CONSERVER_API_TOKEN_FILE`. The API reads the file once at start, so restart it after a change.
3. A partner key from `ingress_auth` works only on `POST /vcon/external-ingress`. The main token does not work there, and a partner key does not work anywhere else. The `detail` for external ingress names the reason: `API Key required`, `No ingress authentication configured`, `Ingress list '<name>' not configured`, or `Invalid API Key for ingress list '<name>'`.

### The API returns 422

The body failed validation. The reply names the field. The usual causes are a missing `vcon`, `uuid` or `created_at`, a `dialog[].url` that is not a URL, a negative `duration`, a `parties[].tel` that is not a phone number, or a `dialog[].parties` index that does not exist. The rules are listed under [vCon validation](api.md#vcon-validation).

## Processing problems

### vCons stay on the ingress list

**Symptoms:** `LLEN` on the ingress list keeps growing and nothing happens.

1. Is a worker running? `docker compose ps` should show `conserver`. In the log, look for `Worker-1 started`.
2. Does a chain name this exact list in `ingress_lists`? A list that no chain names is never read. With no chains at all, the worker logs `No ingress lists configured, retrying in 15s`.
3. Does `config.yml` parse? The worker re-reads the file on every pass, and a file that does not parse stops the worker. Check the log for a YAML error, validate the file, and restart the worker. Test with `python -c "import yaml; yaml.safe_load(open('config.yml'))"`.
4. `enabled: 0` does not stop a chain. The conserver ignores `enabled`. If a chain should not run, remove it from the file.
5. You do not need to restart for a `config.yml` change. The next vCon uses it. Environment variable changes and new `imports` do need a restart.

### A vCon enters a chain and nothing comes out

A link that returns `None` ends the chain for that vCon without an error. Nothing reaches egress or storage, and the log says `Link <name> halted chain processing for vCon <uuid>`. The usual causes are:

* `sampler` or `jq_link` filtered it. A `jq_link` filter that errors also filters.
* `tag_router` with `forward_original: false`.
* A link could not load the vCon. `VconRedis.get_vcon` returns `None` when the vCon is in neither Redis nor a storage, which happens when the working copy expired before the vCon was processed.
* `wtf_transcribe` has no `vfun-server-url`.

### The dead letter queue grows

**Symptoms:** `conserver.dlq.count` rises, or `LLEN DLQ:<list>` is above zero.

There are two queues, and they replay different things.

| Queue | A UUID is there because | Read | Replay |
| ----- | ----------------------- | ---- | ------ |
| `DLQ:<ingress_list>` | A link raised, or a tracer with `dlq_vcon_on_error: true` raised | `GET /api/dlq?ingress_list=<list>` | `POST /api/dlq/reprocess?ingress_list=<list>` |
| `DLQ:storage:<name>` | One storage `save` raised | `GET /api/dlq/storage?storage_name=<name>` | `POST /api/dlq/storage/reprocess?storage_name=<name>` |

1. Find the cause. Search the log for `Critical error processing vCon` (ingress DLQ) or `Failed to save vCon ... Moving to storage DLQ` (storage DLQ). The line carries the link or storage name and the exception.
2. Fix it. The common causes are a provider key that is wrong or still reads `${...}`, because the conserver does not expand variables, a provider that is down or rate limiting, a storage that refuses connections, or a vCon whose body is missing.
3. Replay. Each call moves or retries at most `count` UUIDs, default 1000, so repeat it until it returns `0`:

```bash
curl -X POST -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/dlq/reprocess?ingress_list=incoming_calls&count=1000"
```

An ingress replay runs the whole chain again. Links skip work they have already done, such as a transcript that exists, so the repeat is cheap. A storage replay runs only the write, and stops at the first failure, so fix the backend first.

A dead-lettered vCon keeps its Redis copy for `VCON_DLQ_EXPIRY` seconds (default seven days). Replay before it expires. A storage replay that returns `409` found a stale `DLQ:storage:<name>:replay-lock` key. Delete that key if no replay is running.

### A link seems stuck

The conserver does not enforce a chain `timeout`, so a link that never returns holds its worker slot until you restart the worker. Put a timeout on the call that can hang. `vfun-timeout` and `url-timeout` do this for `wtf_transcribe`. The `webhook` link has no timeout at all, so prefer the `webhook` storage, which times out after 30 seconds. Lower `CONSERVER_VCON_CONCURRENCY` to 1 to see one chain at a time, and use `conserver.vcons.inflight` and the per-link `execution_time` histogram to find the slow link. `delay` is a built-in link you can use to test a slow chain.

### vCons are lost when a worker restarts

With more than one worker per container, shutdown gives each worker 30 seconds. A chain still running then is killed, and its UUID is already off the ingress list. Resubmit it with `POST /api/vcon/ingress?ingress_list=<list>`, which accepts the UUID if the vCon is still in Redis or a storage. See [Production Deployment](production-deployment.md#shutdown).

## Transcription problems

### A vCon has no transcript

1. The dialog must have `type: "recording"` and, for `openai_transcribe` and `deepgram_link`, a `url`. `groq_whisper` and `hugging_face_whisper` also accept an inline body, and read `duration` without a default. A recording with no `duration` raises a `KeyError`.
2. The recording must be at least `minimum_duration` seconds. The defaults are 3 for OpenAI, 30 for Groq and Hugging Face, and 60 for Deepgram.
3. A dialog that already has a `transcript` analysis is skipped. That makes reruns safe, and it also means a changed option does not re-transcribe. To redo one, remove the analysis from the stored vCon.
4. For Deepgram, a transcript below `minimum_confidence` (default 0.5) is discarded.
5. The key goes in the link's `options`. If the log shows a 401 from the provider, check that the value is the real key and not a `${VAR}` placeholder.
6. A `wtf_transcribe` failure on one dialog is logged and skipped. Look for `Error transcribing dialog` in the log.

### An analysis link does nothing

An analysis link reads the transcript at `source.text_location`. The default, `body.paragraphs.transcript`, matches Deepgram output. For OpenAI, Groq and Hugging Face transcripts set `text_location: body.text`. A mismatch logs `No source_text found at <path>` and skips the dialog. The link also skips when `sampling_rate` or `only_if` excludes the vCon, and when an analysis of its `analysis_type` already exists.

## Storage problems

### A storage write fails

1. Replay the queue after fixing the backend. The reason is in the log line `Failed to save vCon <uuid> to storage <name>`.
2. Postgres and S3 read most options by key, so `options` must include them all. A `KeyError` for `aws_access_key_id` or `host` means one is missing. S3 does not use an instance role.
3. `file` raises when a vCon is larger than `max_file_size` (10 MB by default). Files also vanish with the container unless `path` is on a volume.
4. `milvus` raises `ModuleNotFoundError` when `pymilvus` is not in the image, and raises when the collection does not exist and `create_collection_if_missing` is false.
5. `elasticsearch` skips a vCon with no dialog without an error, and logs, but does not raise, when a single document fails.
6. `sftp` and `chatgpt_files` have known faults at this release. See [Storage](storage.md).
7. `spaceandtime` logs a failed write and does not dead-letter it.

### The API returns 404 for a vCon

The working copy expires after `VCON_REDIS_EXPIRY` seconds. After that the API asks each storage with a `get`, in config order. If none has the vCon, you get `404`. Check that the chain lists a storage that supports `get`, such as `postgres`, `s3`, `mongo`, `file` or `vcon_mcp`, and that the write succeeded.

```bash
docker compose exec redis redis-cli EXISTS vcon:<uuid>
docker compose exec redis redis-cli TTL vcon:<uuid>
```

`DELETE /api/vcon/<uuid>` removes the Redis key and the storage copies but leaves the entry in the `vcons` sorted set, so `GET /api/vcon` can still list a UUID that returns `404`.

## Configuration problems

### A setting has no effect

* **Secrets show as literal text.** The conserver does not expand `${VAR}`. Write the value into `config.yml`.
* **A `_FILE` variable is ignored.** Only `CONSERVER_API_TOKEN_FILE` exists.
* **An environment variable is not read.** `LOG_LEVEL`, `HOSTNAME`, `TICK_INTERVAL` and `VCON_SORTED_FORCE_RESET` have no effect. Use `LOGGING_CONFIG_FILE` for logging.
* **A chain `timeout` or `enabled` does nothing.** Neither is enforced.
* **The `diet` link did nothing.** Its options are `remove_dialog_body`, `remove_analysis`, `remove_attachment_types`, `remove_system_prompts`, `post_media_to_url` and the `s3_*` options. Names such as `remove_dialog_bodies` are ignored.
* **The `tag` link adds the wrong tags.** `tags` is a list of `"name:value"` strings or a dict, not a list of objects.
* **`POST /api/config` returns 500.** `CONSERVER_CONFIG_FILE` is unset, or the file is on a read-only mount.

### An imported module is not found

**Symptoms:** `ModuleNotFoundError` in the log.

1. Check the `imports:` entry. `module` is the import name, and `pip_name` is the package to install.
2. A missing module is installed with `pip install` when a worker starts, or when the link first runs. The container needs network access to the package index, or the package built into the image.
3. A module you wrote must be on `PYTHONPATH` in the image.

## Performance problems

### Processing is slow or the queue backs up

1. Check what limits you. `conserver.link.execution_time` by link shows the slow step, and `conserver.vcons.inflight` shows whether workers are at their limit.
2. For chains that wait on the network, raise `CONSERVER_VCON_CONCURRENCY`. For CPU-bound work, raise `CONSERVER_WORKERS` or add containers.
3. Use `sampler` or `sampling_rate` to analyze a share of vCons.
4. Use a smaller model for `analyze` and the other analysis links.
5. Make sure Redis has memory headroom. See `conserver.redis.memory_used_bytes`.

### Redis memory climbs

1. Lower `VCON_REDIS_EXPIRY`, and add the `expire_vcon` link at the end of chains that do not need the working copy.
2. Add the `diet` link to drop dialog bodies, and store them in S3.
3. Check that the dead letter queues are not holding vCons for `VCON_DLQ_EXPIRY`.
4. Check the `redis_storage` storage. It writes into the same Redis.

## Report a problem

Open an issue on [vcon-server](https://github.com/vcon-dev/vcon-server/issues) with the output of `GET /api/version`, the relevant log lines, and your chain configuration with every key and password removed. Do not paste names of real customers or callers into a public issue.
