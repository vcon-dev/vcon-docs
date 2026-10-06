---
description: Every environment variable and every section of config.yml that the conserver reads, with the defaults it applies.
---

# 🔧 Configuring the Conserver

Two things configure the conserver. Environment variables set server behavior: Redis, authentication, workers, telemetry. A YAML file, `config.yml`, defines the processing: links, storages, chains, tracers and followers. For what a chain does with a vCon, see [Concepts](concepts.md).

## Environment variables

Set these in `.env` or the container environment. The `api` and `conserver` services read the same names. Settings are read once at process start, so changing one needs a restart.

### Core

| Variable | Description | Default |
| -------- | ----------- | ------- |
| `REDIS_URL` | Redis connection URL. A `rediss://` URL gives TLS. | `redis://localhost` |
| `CONSERVER_CONFIG_FILE` | Path to the YAML file. Set it explicitly: `POST /config` writes to this variable and fails if it is unset. | `./example_config.yml` |
| `LOGGING_CONFIG_FILE` | Python `fileConfig` file for logging. `common/logging.conf` writes JSON. `common/logging_dev.conf` writes plain text. | `common/logging.conf` |
| `SENTRY_DSN` | Turns on Sentry error reporting when set. `ENV` must also be set, because Sentry uses it as the environment name. | (unset) |
| `ENV` | Environment name sent to Sentry. Nothing else reads it. | (unset) |
| `UUID8_DOMAIN_NAME` | Read into a constant in `common/vcon.py`. The `Vcon.build_new()` helper does not use it and hardcodes its own domain, so setting it changes nothing today. | `strolid.com` |

### API

| Variable | Description | Default |
| -------- | ----------- | ------- |
| `CONSERVER_API_TOKEN` | Token that authorizes the main API. With no token and no token file, authentication is off. | (unset) |
| `CONSERVER_API_TOKEN_FILE` | File of accepted tokens, one per line. Read once when the API starts. This is the only variable with a `_FILE` form. | (unset) |
| `CONSERVER_HEADER_NAME` | Header that carries the token. | `x-conserver-api-token` |
| `API_ROOT_PATH` | URL prefix for every route, including `/health`. | `/api` |

### Redis and caching

| Variable | Description | Default |
| -------- | ----------- | ------- |
| `VCON_REDIS_EXPIRY` | TTL in seconds for a vCon that the API creates or reloads from storage into Redis. | `3600` |
| `VCON_STORAGE_FALLBACK_ENABLED` | On a Redis miss, `VconRedis.get_vcon` tries each configured storage in turn and caches the first hit. `false` returns `None` on a miss, which halts the chain. | `true` |
| `VCON_INDEX_EXPIRY` | TTL in seconds for the party search index keys (`tel:`, `mailto:`, `name:`). | `86400` |
| `VCON_CONTEXT_EXPIRY` | TTL in seconds for the trace context stored with each queued UUID. | `86400` |
| `VCON_DLQ_EXPIRY` | TTL in seconds applied to a vCon when it is dead-lettered. `0` leaves the TTL alone. | `604800` |
| `VCON_SORTED_SET_NAME` | Redis sorted set that backs `GET /vcon`. | `vcons` |

### Workers

The most vCons in flight is `CONSERVER_WORKERS` times `CONSERVER_VCON_CONCURRENCY`, per container.

| Variable | Description | Default |
| -------- | ----------- | ------- |
| `CONSERVER_WORKERS` | Worker processes per container. Each blocks on the ingress lists with a 15 second timeout. A worker that dies is restarted. | `1` |
| `CONSERVER_VCON_CONCURRENCY` | vCons one worker runs at once, in threads. Above 1 it suits chains that wait on the network (LLM, transcription, webhooks, storage). | `1` |
| `CONSERVER_PARALLEL_STORAGE` | Write a vCon to its chain's storages in parallel threads. | `true` |
| `CONSERVER_START_METHOD` | `fork`, `spawn` or `forkserver`, for more than one worker. Blank uses the platform default. Any other value stops startup. | (unset) |

### Format and telemetry

| Variable | Description | Default |
| -------- | ----------- | ------- |
| `EGRESS_FORMAT_VERSION` | Set to `0.0.1` to emit that legacy shape from every egress point: the `webhook` link, the `s3`, `postgres` and `elasticsearch` storages, and the API read endpoints. It maps `amended` to `appended`, `critical` to `must_support`, `mediatype` to `mimetype`, `schema` to `schema_version`, attachment `purpose` to `type`, and turns native JSON bodies back into strings. The copy in Redis stays on the current spec. | (unset) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP collector. Blank turns metrics and traces export off. | (unset) |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` or `http/protobuf`, as the OpenTelemetry SDK defines them. | set by compose |
| `OTEL_EXPORTER_OTLP_HEADERS` | Headers for the collector, for example a vendor auth header. | (unset) |
| `OTEL_EXPORTER_OTLP_INSECURE` | Plain-text gRPC. | `false` |
| `OTEL_SERVICE_NAME` | Service name on every metric and span. Compose sets `conserver` and `api`. | `vcon-server` |
| `OTEL_METRIC_EXPORT_INTERVAL` | Metric export interval in milliseconds. | `5000` |

The containers start under `opentelemetry-instrument`, which reads the `OTEL_EXPORTER_OTLP_*` variables for traces. The conserver's own `conserver.*` metrics are created in `common/lib/metrics.py`, which builds a gRPC exporter against `OTEL_EXPORTER_OTLP_ENDPOINT` and does not consult the protocol variable. [Production Deployment](production-deployment.md) lists the metrics.

`LOG_LEVEL`, `HOSTNAME`, `TICK_INTERVAL` and `VCON_SORTED_FORCE_RESET` appear in `common/settings.py`, but nothing reads them. Remove them from older `.env` files.

### Provider keys

The conserver does not read `OPENAI_API_KEY`, `DEEPGRAM_KEY` or `GROQ_API_KEY` from the environment for its links. Put each key in the link's `options`, as described under [Secrets](#secrets). Two links read a variable as a default for an option: `detect_engagement` for `OPENAI_API_KEY`, and `groq_whisper` for `API_KEY` from `GROQ_API_KEY`. Both read it when the module is imported.

## config.yml

The conserver opens the file named by `CONSERVER_CONFIG_FILE` and parses it with `yaml.safe_load`. It re-reads the file on every pass of the worker loop, so edits apply to the next vCon without a restart. Three things are read once: `imports` (when a worker starts), `followers` (when the process starts) and environment variables.

```yaml
ingress_auth: {}
imports: {}
links: {}
storages: {}
tracers: {}
chains: {}
followers: {}
```

### Secrets

Values in `config.yml` are literal strings. The conserver does not expand `${VAR}`: a link configured with `OPENAI_API_KEY: ${OPENAI_API_KEY}` sends the text `${OPENAI_API_KEY}` to the provider. Write the key into the file, and treat the file as a secret. Keep it out of version control, give it restrictive permissions, and mount it read-only where you can.

Keys go in the link or storage `options`, under the names that module reads, such as `OPENAI_API_KEY`, `DEEPGRAM_KEY`, `API_KEY` or `api_key`. Each page of [Standard Links](standard-links.md) and [Storage](storage.md) names them. Links that call OpenAI also accept `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY` and `AZURE_OPENAI_API_VERSION`, or `LITELLM_PROXY_URL` with `LITELLM_MASTER_KEY`. The `_FILE` pattern is not supported for provider keys.

`GET /config` returns the whole file, secrets included, to anyone who holds the API token.

### ingress_auth

Maps an ingress list name to the key or keys that may submit to it through `POST /vcon/external-ingress`. Each key opens one list and nothing else. See [API](api.md#external-ingress).

```yaml
ingress_auth:
  partner_data:
    - "partner-key-1"
    - "partner-key-2"
  customer_data: "single-key"
```

### imports

Names modules to import when a worker starts. A module that is not installed is installed with `pip install` at that moment, so the container needs network access to the package index, and a package named in `imports` runs code in your worker.

```yaml
imports:
  custom_analysis:
    module: my_analysis_module
    pip_name: my-analysis-package>=1.0.0
  github_link:
    module: github_link
    pip_name: git+https://github.com/example/github-link.git@v2.0.0
```

`pip_name` can be left out when it equals `module`. A bare string value, `legacy: some.module`, also works. A link or tracer entry may carry its own `pip_name` and is installed the first time it runs.

### links

A link entry names a module and its options. The same module can appear under several names.

```yaml
links:
  summarize:
    module: links.analyze
    options:
      OPENAI_API_KEY: "sk-..."
      prompt: "Summarize this conversation in three sentences."
      analysis_type: summary
  summarize_short:
    module: links.analyze
    options:
      OPENAI_API_KEY: "sk-..."
      prompt: "Summarize this conversation in one sentence."
      analysis_type: short_summary
```

`module` is a dotted Python path. `links.<name>` loads a shipped link. Any importable module with a `run` function works, which is how [custom links](creating-custom-links.md) load. The keys `ingress-lists` and `egress-lists` on a link entry, which appear in `example_config.yml`, are not read.

A link can also carry an `after_link` block inside `options`. After each link the conserver calls the hook in `conserver/after_link_hook.py` with that block as `link_hook_config`, along with the status, the error if any, and the telephone numbers and email addresses of the vCon's parties. The shipped hook does nothing. Replace the file when you build the image to add audit logging or notifications. `conserver/hook.py` works the same way for `before_processing` and `after_processing`, once per vCon. A `before_processing` that returns a falsy value skips the vCon without dead-lettering it.

The vendor transcription links `links.openai_transcribe`, `links.groq_whisper`, `links.hugging_face_whisper` and `links.deepgram_link` are deprecated. The conserver reroutes each to `links.transcribe` with the matching `vendor` and logs one warning per module. See [Standard Links](standard-links.md#transcribe).

### storages

```yaml
storages:
  postgres:
    module: storage.postgres
    options:
      host: postgres
      port: 5432
      user: postgres
      password: "change-me"
      database: postgres
```

Each storage module has its own options. If a storage entry has no `options` key, the module's defaults apply as a whole. If it has one, the module merges it over its defaults only where the module says so, so give every required option. [Storage](storage.md) lists them.

### tracers

Tracers are global. They run for every chain and every link, and a chain cannot select them. A `tracers:` list inside a chain, as `example_config.yml` shows in a comment, is ignored. See [Conserver Tracers](conserver-tracers.md).

```yaml
tracers:
  jlinc:
    module: tracers.jlinc
    options:
      data_store_api_url: http://jlinc-server:9090
      data_store_api_key: "your-key"
      archive_api_url: http://jlinc-server:9090
      archive_api_key: "your-key"
```

### chains

```yaml
chains:
  main:
    ingress_lists: [incoming_calls]
    links: [transcribe_audio, summarize]
    storages: [postgres, s3]
    egress_lists: [processed]
    enabled: 1
    timeout: 600
```

`ingress_lists` and `links` are required. `storages` and `egress_lists` are optional. Two settings are accepted and ignored: `enabled` does not stop a chain, and `timeout` does not stop a link. Remove a chain from the file to stop it. The order of work, the return values of a link, egress before storage and the dead letter queues are all on [Concepts](concepts.md).

### followers

A follower pulls finished vCons from another conserver. Every key is required, and `url` must include the remote server's `API_ROOT_PATH`.

```yaml
followers:
  upstream:
    url: "https://remote.example.com/api"
    auth_token: "token-for-the-remote-api"
    egress_list: remote_output
    follower_ingress_list: local_input
    pulling_interval: 60
    fetch_vcon_limit: 10
```

Every `pulling_interval` seconds the follower calls `GET /vcon/egress` on the remote server, fetches each returned UUID, writes the vCon into local Redis and pushes the UUID onto `follower_ingress_list`.

## Changing the file at run time

Edit `config.yml` in place and the next vCon uses it. `POST /config` replaces the file from a JSON body, through the API container's own copy. That fails on a read-only mount, and it does not reach other hosts unless they share the file. A file that does not parse stops the worker, so validate the YAML before you save it.
