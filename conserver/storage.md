---
description: Reference for the 15 storage modules in vcon-server, with their options, defaults, and which of them can read, delete and dead-letter a vCon.
---

# 🗄️ Storage

A storage is where a chain writes a vCon when it finishes. The conserver ships 15 modules. The option tables below come from each module's `default_options` at vcon-server `b603b15`. For where storage sits in a chain, and how failed writes are queued, see [Concepts](concepts.md).

## How storage behaves

Each chain lists the storages it writes to. After the last link, the conserver loads the vCon from Redis and calls each module's `save`. With more than one storage and `CONSERVER_PARALLEL_STORAGE=true` (the default), the writes run in parallel threads.

If a `save` raises, the chain does not fail. The UUID goes onto `DLQ:storage:<name>` and the vCon's Redis TTL is raised to `VCON_DLQ_EXPIRY`. Replay the queue with `POST /dlq/storage/reprocess`. See [API](api.md#dead-letter-queues). Replay calls `save` again, so a module has to accept the same UUID twice. A module that logs its errors and returns, such as `spaceandtime`, never reaches the queue.

Some modules also implement `get` and `delete`. The API uses `get` to reload a vCon that expired from Redis, trying each storage in `config.yml` order, and uses `delete` for `DELETE /vcon/{uuid}`. A module without `get` or `delete` is skipped.

| Storage | `get` | `delete` | Notes |
| ------- | ----- | -------- | ----- |
| `chatgpt_files` | no | no | See the known issue below |
| `dataverse` | yes | no | |
| `elasticsearch` | no | yes | Write-only index |
| `file` | yes | yes | |
| `milvus` | yes | no | Needs `pymilvus` in the image |
| `mongo` | yes | no | |
| `postgres` | yes | yes | |
| `redis_storage` | yes | no | Uses the main Redis |
| `s3` | yes | yes | |
| `scitt` | no | no | Registers a statement, stores no copy |
| `sftp` | yes | no | See the known issue below |
| `spaceandtime` | yes | no | `save` swallows errors |
| `utopia` | no | yes | Stores a rendered text copy |
| `vcon_mcp` | yes | yes | |
| `webhook` | no | no | |

Options go under `options:` in the storage entry. If you omit `options` the module's defaults apply as a whole. If you include it, most modules read each key directly, so supply every option that has no usable default. Credentials go in the file as literal strings. The conserver does not expand `${VAR}`. See [Configuring the Conserver](configuring-the-conserver.md#secrets).

When `EGRESS_FORMAT_VERSION` is set, `s3`, `postgres` and `elasticsearch` store the legacy shape it names. The `webhook` link does too. The copy in Redis does not change.

***

### MongoDB

Stores the vCon as a document, keyed by its UUID in `_id`, with an upsert.

```yaml
storages:
  mongo:
    module: storage.mongo
    options:
      url: "mongodb://mongo:27017/"
      database: conserver
      collection: vcons
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `url` | Connection URL. The module raises if the key is missing | `mongodb://localhost:27017/` |
| `database` | Database name | `conserver` |
| `collection` | Collection name | `vcons` |

The module logs the full options at INFO, including a password in `url`. Restrict who can read your logs, or keep credentials out of the URL.

***

### PostgreSQL

Stores each vCon as a row, with the whole vCon in a JSONB column. The module creates the table if it is missing.

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
      table_name: vcons
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `host`, `port`, `user`, `password`, `database` | Connection. Give all five when you set `options` | `localhost`, `5432`, `postgres`, empty, `vcon_db` |
| `table_name` | Table name | `vcons` |

Columns: `id` (UUID, primary key), `vcon` (text), `uuid`, `created_at`, `updated_at`, `subject`, and `vcon_json` (JSONB). Query `vcon_json` for the content.

***

### Elasticsearch

Indexes the parts of a vCon into separate indices for search. It writes only. There is no `get`, so Elasticsearch cannot restore a vCon.

```yaml
storages:
  elasticsearch:
    module: storage.elasticsearch
    options:
      url: "https://elasticsearch:9200"
      username: elastic
      password: "change-me"
      ca_certs: "/certs/ca.crt"
      index_prefix: "myapp_"
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `cloud_id`, `api_key` | Elastic Cloud connection. If either is set, it is used and the self-hosted options are ignored | empty |
| `url`, `username`, `password` | Self-hosted connection | `None` |
| `ca_certs` | CA bundle path. If the file does not exist, TLS verification is switched off | `None` |
| `index_prefix` | Prefix for every index name | empty |

The `index` option is in the defaults and is not read. Documents go to `<prefix>vcon_parties_<role>` (`vcon_parties` when a party has no role), `<prefix>vcon_attachments_<purpose>`, `<prefix>vcon_analysis_<type>` and `<prefix>vcon_dialog`. The document id is `<uuid>_<position>`. Each document gets `vcon_id`, `started_at` from the first dialog, and `tenant_id` when the vCon has an attachment with `purpose: tenant`. A vCon with no dialog is skipped without an error. An error on one document is logged and the rest continue. `delete` removes every document for the vCon.

***

### Milvus

Embeds the text of a vCon and stores the vector for semantic search.

```yaml
storages:
  milvus:
    module: storage.milvus
    options:
      host: milvus
      port: "19530"
      collection_name: vcons
      api_key: "sk-..."
      embedding_model: text-embedding-3-small
      embedding_dim: 1536
      create_collection_if_missing: true
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `host`, `port` | Milvus server | `localhost`, `19530` |
| `collection_name` | Collection | `vcons` |
| `api_key`, `organization` | Embedding provider credentials. LiteLLM and Azure options also work, as for the [OpenAI-backed links](standard-links.md#conventions) | `None` |
| `embedding_model`, `embedding_dim` | Model and vector size. A reply of the wrong size raises | `text-embedding-3-small`, `1536` |
| `create_collection_if_missing` | Create the collection if absent. If false and it is absent, the write raises | `false` |
| `skip_if_exists` | Do nothing if the vCon is already in the collection | `true` |
| `index_type` | `IVF_FLAT`, `IVF_SQ8`, `IVF_PQ`, `HNSW`, `ANNOY` or `FLAT` | `IVF_FLAT` |
| `metric_type` | `L2`, `IP` or `COSINE` | `L2` |
| `nlist` | IVF clusters | `128` |
| `m`, `ef_construction` | HNSW settings | `16`, `200` |
| `pq_m`, `pq_nbits`, `n_trees` | `IVF_PQ` and `ANNOY` settings, read when the collection is created | `8`, `8`, `50` |

It builds the text from transcripts, summaries, party details and dialog. A vCon with no text is skipped. The `pymilvus` package is not in the `Dockerfile.conserver` or `Dockerfile.api` images, only in the development image. Add the `storage-milvus` dependency group to your build.

***

### Amazon S3

Writes the vCon to a bucket, or to any S3-compatible service.

```yaml
storages:
  s3:
    module: storage.s3
    options:
      aws_access_key_id: "AKIAEXAMPLE"
      aws_secret_access_key: "example-secret"
      aws_bucket: my-vcon-bucket
      aws_region: us-east-1
      s3_path: vcons
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `aws_access_key_id`, `aws_secret_access_key` | Credentials. Required, and the module does not fall back to an instance role | (required) |
| `aws_bucket` | Bucket | (required) |
| `aws_region` | Region | the SDK default |
| `endpoint_url` | Endpoint for S3-compatible storage such as MinIO | `None` |
| `s3_path` | Key prefix | none |

The object key is `[<s3_path>/]YYYY/MM/DD/<uuid>.vcon`, using the vCon's `created_at`. A small pointer object, `[<s3_path>/]lookup/<uuid>.txt`, holds the date path so `get` and `delete` can find the object from the UUID alone.

***

### SFTP

Uploads the vCon to an SFTP server.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `url` | Server host name. The name says URL, but the value goes straight to the SSH library, so give a bare host such as `sftp.example.com`. The default `sftp://localhost` does not resolve | `sftp://localhost` |
| `port` | Port | `22` |
| `username`, `password` | Login | `username`, `password` |
| `path` | Remote directory | `.` |
| `filename`, `extension` | File name parts | `vcon`, `json` |
| `add_timestamp_to_filename` | Append the time to the file name | `true` |

Known issues at `b603b15`: the file name does not contain the UUID, so `get` returns the newest file that matches the name pattern, whichever vCon that is, and the module's upload method calls itself, so `save` appears to fail with a recursion error. Treat the `sftp` storage as not working until both are fixed. If it is configured, every vCon lands in `DLQ:storage:sftp`.

***

### File system

Writes JSON files on the container's disk. Mount a volume at `path`, or the files go away with the container.

```yaml
storages:
  file:
    module: storage.file
    options:
      path: /data/vcons
      organize_by_date: true
      compression: false
      max_file_size: 10485760
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `path` | Base directory | `/data/vcons` |
| `organize_by_date` | Put files under `YYYY/MM/DD` | `true` |
| `compression` | Write gzip files | `false` |
| `max_file_size` | Largest vCon to accept, in bytes. A larger one raises | `10485760` |
| `file_permissions` | Mode for files | `0o644` |
| `dir_permissions` | Mode for directories | `0o755` |

Files are named `<uuid>.json`, or `<uuid>.json.gz` with compression. There are no `filename`, `extension` or timestamp options.

***

### Redis storage

Copies the vCon to a second key in Redis, with a prefix and an expiry.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `prefix` | Key prefix. The key is `<prefix>:<uuid>` | `vcon_storage` |
| `expires` | TTL in seconds | `604800` |
| `redis_url` | In the defaults, and not read | `redis://:localhost:6379` |

The module writes through the conserver's own Redis connection, so the copy lives in the same Redis as the working data, not in a separate one. It adds to that instance's memory use.

***

### ChatGPT Files

Uploads the vCon to OpenAI file storage and attaches it to a vector store.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `api_key`, `organization_key`, `project_key` | OpenAI credentials | placeholders |
| `vector_store_id` | Vector store to add the file to | placeholder |
| `purpose` | File purpose | `assistants` |

Known issue at `b603b15`: the module reads the vCon from Redis with the bare UUID instead of `vcon:<uuid>`, so it appears to upload the text `null`. Check an uploaded file before you depend on this storage.

***

### Microsoft Dataverse

Writes the vCon into a custom Dataverse entity.

```yaml
storages:
  dataverse:
    module: storage.dataverse
    options:
      url: "https://example.crm.dynamics.com"
      tenant_id: "your-tenant-id"
      client_id: "your-client-id"
      client_secret: "your-client-secret"
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `url` | Dataverse environment URL | `https://org.crm.dynamics.com` |
| `api_version` | API version | `9.2` |
| `tenant_id`, `client_id`, `client_secret` | Azure AD application credentials | empty |
| `entity_name` | Custom entity | `vcon_storage` |
| `uuid_field`, `data_field`, `subject_field`, `created_at_field` | Entity field names | `vcon_uuid`, `vcon_data`, `vcon_subject`, `vcon_created_at` |

Create the entity and its fields in Dataverse first.

***

### Space and Time

Inserts the vCon as a row in a Space and Time table. The module reads these variables from the environment, not from `options`:

| Variable | Description |
| -------- | ----------- |
| `SXT_API_KEY` | API key. The module authenticates when it is imported |
| `SXT_VCON_TABLENAME` | Table name |
| `SXT_VCON_TABLE_WRITE_BISCUI` | Write biscuit. The code spells the name without the final `T`, so `SXT_VCON_TABLE_WRITE_BISCUIT` is not read |

The table needs an upper-case column for every top-level vCon key. `save` logs a failure and returns, so a failed write is not dead-lettered.

***

### SCITT Transparency

Registers a signed statement about the vCon on a [SCRAPI](https://datatracker.ietf.org/doc/draft-ietf-scitt-scrapi/) transparency service. It does not write anything back to the vCon, and it does not verify receipts. The service is the record. To put the receipt on the vCon, use the [`scitt` link](standard-links.md#scitt). Both fit the [Lifecycle extension](../extensions/lifecycle.md).

```yaml
storages:
  scitt_transparency:
    module: storage.scitt
    options:
      scrapi_url: "http://scittles:8000"
      signing_key_pem: "<base64 of the PEM text>"
      issuer: conserver
      key_id: conserver-key-1
      operations: [vcon_enhanced]
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `scrapi_url` | Service base URL | `http://scittles:8000` |
| `signing_key_pem` | Base64 of the PEM private key text | `None` |
| `signing_key_path` | PEM file path, used when `signing_key_pem` is empty | `/etc/scitt/signing-key.pem` |
| `issuer`, `key_id` | COSE issuer and key id | `conserver`, `conserver-key-1` |
| `operations` | Events to register. One statement is sent for each operation and each party `tel`, or one per operation if no party has a `tel` | `[vcon_enhanced]` |

***

### Utopia

Sends the vCon to a [Utopia](https://github.com/deeplethe/utopia) knowledge base. Utopia reads prose, so the module renders the vCon as dated sentences and uploads `<uuid>.md`. What Utopia holds is that rendering, not the vCon, which is why there is no `get`. A changed vCon is uploaded again and the earlier rendering is removed. An unchanged one is a no-op.

```yaml
storages:
  utopia:
    module: storage.utopia
    options:
      base_url: "http://utopia:1516/api/v1"
      kb_id: "your-knowledge-base-uuid"
      email: "service-account@example.com"
      password: "change-me"
      require_analysis_grant: true
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `base_url` | Utopia REST root | `http://127.0.0.1:1516/api/v1` |
| `kb_id` | Knowledge base UUID | (required) |
| `email`, `password` | A service account with Editor on the knowledge base | (required) |
| `timeout` | Request timeout in seconds | `30` |
| `transient_retries` | Retries for 429, 502, 503, 504 and connection errors | `3` |
| `transient_backoff_base_s` | Backoff factor in seconds | `0.5` |
| `require_analysis_grant` | Skip a vCon unless its lawful basis attachment grants the `analysis` purpose. If false the module writes it and logs the gap | `false` |

`delete` removes the rendering from Utopia, and the removal can be reversed there.

***

### vCon MCP

Sends the vCon to a running [vCon MCP server](../mcp-server/README.md) through its REST API, so agents can query it through the MCP tools.

```yaml
storages:
  mcp_store:
    module: storage.vcon_mcp
    options:
      base_url: "http://vcon-mcp:3000/api/v1"
      api_key: "your-mcp-key"
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `base_url` | REST root | `http://127.0.0.1:3000/api/v1` |
| `api_key` | Sent as a bearer token | empty |
| `timeout` | Request timeout in seconds | `30` |
| `transient_retries` | Retries for 429, 502, 503, 504 and connection errors, including `POST` | `3` |
| `transient_backoff_base_s` | Backoff factor in seconds | `0.5` |

`save` posts to `/vcons`, `get` reads `/vcons/<uuid>`, and `delete` calls `DELETE /vcons/<uuid>`. A retried `POST` is safe because the MCP server upserts on the UUID.

***

### Webhook

POSTs the vCon to each URL after the chain finishes. The [`webhook` link](standard-links.md#webhook) does the same mid-chain. Use the storage when the endpoint is a destination whose failures you want queued and replayed.

```yaml
storages:
  notify:
    module: storage.webhook
    options:
      webhook-urls:
        - "https://api.example.com/vcons"
      headers:
        Authorization: "Bearer example-token"
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `webhook-urls` | URLs to POST to, in order. If empty the write is skipped with a warning | `[]` |
| `headers` | Headers for every request | `{}` |
| `timeout` | Seconds per request | `30` |

A non-2xx reply raises, so the UUID goes to `DLQ:storage:<name>`. A vCon whose `vcon` field is `0.0.1` is sent as `0.3.0`. The call time is recorded in `conserver.webhook.duration`. Unlike the link, this module does not apply `EGRESS_FORMAT_VERSION`.

***

## Using several storages

```yaml
chains:
  main:
    ingress_lists: [incoming]
    links: [transcribe_audio, summarize]
    storages: [archive_s3, query_postgres, search_milvus]
    egress_lists: [processed]
```

The chain's egress list is filled before the storage writes begin, so a consumer can see a UUID before its vCon is stored. See [Concepts](concepts.md#chain).

## Redis as the working copy

Redis holds the working copy under `vcon:<uuid>`. When the API needs a vCon that is not there, it asks each storage that has `get`, restores the first hit into Redis with a TTL of `VCON_REDIS_EXPIRY` (default 3600 seconds), and serves later requests from Redis. Inside a chain, `VCON_STORAGE_FALLBACK_ENABLED` (default `true`) gives links the same fallback.
