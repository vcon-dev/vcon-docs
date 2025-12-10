---
description: The standard storages supported by the conserver
---

# Storage

The Conserver supports multiple storage backends for persisting vCons after processing. Each storage backend can be configured in the `config.yml` file and assigned to one or more chains.

## How Storage Works

When a chain finishes processing a vCon, it stores the result in all configured storage backends for that chain. This allows you to:

- Store vCons in multiple locations simultaneously (e.g., S3 for archival and PostgreSQL for querying)
- Choose the right storage for your use case
- Implement backup and redundancy strategies

## Storage Interface

All storage backends implement a standard interface:

```python
def save(vcon_uuid: str, opts: dict) -> None
def get(vcon_uuid: str, opts: dict) -> Optional[dict]
def delete(vcon_uuid: str, opts: dict) -> bool  # Some backends
```

## Available Storage Backends

### MongoDB

Document-oriented NoSQL database, ideal for storing vCons in their native JSON structure.

```yaml
storages:
  mongo:
    module: storage.mongo
    options:
      url: "mongodb://localhost:27017/"
      database: "conserver"
      collection: "vcons"
```

| Option | Description | Default |
|--------|-------------|---------|
| `url` | MongoDB connection URL | `mongodb://localhost:27017/` |
| `database` | Database name | `conserver` |
| `collection` | Collection name | `vcons` |

---

### PostgreSQL

Relational database with JSON support, ideal for complex queries and reporting.

```yaml
storages:
  postgres:
    module: storage.postgres
    options:
      database: "vcon_db"
      user: "postgres"
      password: "your_password"
      host: "localhost"
      port: 5432
      table_name: "vcons"
```

| Option | Description | Default |
|--------|-------------|---------|
| `database` | Database name | `vcon_db` |
| `user` | Database username | `postgres` |
| `password` | Database password | (required) |
| `host` | Database host | `localhost` |
| `port` | Database port | `5432` |
| `table_name` | Table name for vCons | `vcons` |

The PostgreSQL storage automatically creates the table if it doesn't exist, with columns for:
- `id` (UUID, primary key)
- `vcon` (text)
- `uuid` (UUID)
- `created_at` (datetime)
- `updated_at` (datetime)
- `subject` (text)
- `vcon_json` (JSONB for querying)

---

### Elasticsearch

Full-text search and analytics engine. Indexes vCon components (parties, dialog, analysis, attachments) into separate indices for powerful searching.

```yaml
storages:
  elasticsearch:
    module: storage.elasticsearch
    options:
      # Option 1: Elastic Cloud
      cloud_id: "your_cloud_id"
      api_key: "your_api_key"

      # Option 2: Self-hosted
      url: "https://localhost:9200"
      username: "elastic"
      password: "your_password"
      ca_certs: "/path/to/ca.crt"  # Optional

      index_prefix: "myapp_"  # Optional prefix for index names
```

| Option | Description | Default |
|--------|-------------|---------|
| `cloud_id` | Elastic Cloud deployment ID | (empty) |
| `api_key` | Elastic Cloud API key | (empty) |
| `url` | Self-hosted Elasticsearch URL | (none) |
| `username` | Basic auth username | (none) |
| `password` | Basic auth password | (none) |
| `ca_certs` | Path to CA certificate | (none) |
| `index_prefix` | Prefix for all index names | (empty) |

Creates multiple indices:
- `vcon_parties_{role}` - Party information by role
- `vcon_attachments_{type}` - Attachments by type
- `vcon_analysis_{type}` - Analysis results by type
- `vcon_dialog` - Dialog entries

---

### Milvus

Vector database for semantic search. Stores vCon content as embeddings for similarity-based retrieval.

```yaml
storages:
  milvus:
    module: storage.milvus
    options:
      host: "localhost"
      port: "19530"
      collection_name: "vcons"

      # OpenAI embedding configuration
      api_key: "sk-your-openai-key"
      organization: "org-xxx"  # Optional
      embedding_model: "text-embedding-3-small"
      embedding_dim: 1536

      # Collection settings
      create_collection_if_missing: true
      skip_if_exists: true

      # Index configuration
      index_type: "IVF_FLAT"  # IVF_FLAT, IVF_SQ8, IVF_PQ, HNSW, ANNOY, FLAT
      metric_type: "L2"       # L2, IP (Inner Product), COSINE
      nlist: 128              # For IVF indexes
      m: 16                   # For HNSW: edges per node
      ef_construction: 200    # For HNSW: construction candidate list size
```

| Option | Description | Default |
|--------|-------------|---------|
| `host` | Milvus server hostname | `localhost` |
| `port` | Milvus server port | `19530` |
| `collection_name` | Collection name | `vcons` |
| `api_key` | OpenAI API key for embeddings | (required) |
| `organization` | OpenAI organization ID | (none) |
| `embedding_model` | OpenAI embedding model | `text-embedding-3-small` |
| `embedding_dim` | Embedding vector dimensions | `1536` |
| `create_collection_if_missing` | Auto-create collection | `false` |
| `skip_if_exists` | Skip storing existing vCons | `true` |
| `index_type` | Vector index type | `IVF_FLAT` |
| `metric_type` | Distance metric | `L2` |
| `nlist` | IVF cluster count | `128` |
| `m` | HNSW edges per node | `16` |
| `ef_construction` | HNSW construction list size | `200` |

The Milvus storage extracts text from transcripts, summaries, party info, and dialog to create searchable embeddings.

---

### Amazon S3

Object storage for cloud-native archival and backup.

```yaml
storages:
  s3:
    module: storage.s3
    options:
      aws_access_key_id: "AKIAIOSFODNN7EXAMPLE"
      aws_secret_access_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
      aws_bucket: "my-vcon-bucket"
      s3_path: "vcons"  # Optional subdirectory
```

| Option | Description | Default |
|--------|-------------|---------|
| `aws_access_key_id` | AWS access key ID | (required) |
| `aws_secret_access_key` | AWS secret access key | (required) |
| `aws_bucket` | S3 bucket name | (required) |
| `s3_path` | Optional path prefix within bucket | (none) |

Files are stored as `{s3_path}/{YYYY/MM/DD}/{uuid}.vcon` based on the vCon's creation date.

---

### SFTP

Secure file transfer to remote servers.

```yaml
storages:
  sftp:
    module: storage.sftp
    options:
      url: "sftp.example.com"
      port: 22
      username: "vcon_user"
      password: "your_password"
      path: "/uploads/vcons"
      filename: "vcon"
      extension: "json"
      add_timestamp_to_filename: true
```

| Option | Description | Default |
|--------|-------------|---------|
| `url` | SFTP server hostname | `localhost` |
| `port` | SFTP server port | `22` |
| `username` | SFTP username | (required) |
| `password` | SFTP password | (required) |
| `path` | Remote directory path | `.` |
| `filename` | Base filename | `vcon` |
| `extension` | File extension | `json` |
| `add_timestamp_to_filename` | Append timestamp to filename | `true` |

---

### File System

Local filesystem storage for development or simple deployments.

```yaml
storages:
  file:
    module: storage.file
    options:
      path: "/var/vcons"
      filename: "vcon"
      extension: "json"
      add_timestamp_to_filename: true
```

| Option | Description | Default |
|--------|-------------|---------|
| `path` | Directory to store files | `.` |
| `filename` | Base filename | `vcon` |
| `extension` | File extension | `json` |
| `add_timestamp_to_filename` | Append timestamp to filename | `true` |

---

### Redis Storage

Dedicated Redis storage with custom prefix and expiration, separate from the main Redis working storage.

```yaml
storages:
  redis_storage:
    module: storage.redis_storage
    options:
      redis_url: "redis://localhost:6379"
      prefix: "vcon_archive"
      expires: 604800  # 7 days in seconds
```

| Option | Description | Default |
|--------|-------------|---------|
| `redis_url` | Redis connection URL | `redis://localhost:6379` |
| `prefix` | Key prefix for stored vCons | `vcon_storage` |
| `expires` | TTL in seconds | `604800` (7 days) |

---

### ChatGPT Files

Store vCons in OpenAI's file storage for use with ChatGPT assistants and vector stores.

```yaml
storages:
  chatgpt_files:
    module: storage.chatgpt_files
    options:
      organization_key: "org-xxxxx"
      project_key: "proj_xxxxxxx"
      api_key: "sk-proj-xxxxxx"
      vector_store_id: "vs_xxxxxx"
      purpose: "assistants"
```

| Option | Description | Default |
|--------|-------------|---------|
| `organization_key` | OpenAI organization ID | (required) |
| `project_key` | OpenAI project ID | (required) |
| `api_key` | OpenAI API key | (required) |
| `vector_store_id` | Vector store ID to add files to | (required) |
| `purpose` | File purpose | `assistants` |

---

### Microsoft Dataverse

Integration with Microsoft Dynamics 365 / Dataverse for enterprise CRM scenarios.

```yaml
storages:
  dataverse:
    module: storage.dataverse
    options:
      url: "https://org.crm.dynamics.com"
      api_version: "9.2"
      tenant_id: "your-tenant-id"
      client_id: "your-client-id"
      client_secret: "your-client-secret"
      entity_name: "vcon_storage"
      uuid_field: "vcon_uuid"
      data_field: "vcon_data"
      subject_field: "vcon_subject"
      created_at_field: "vcon_created_at"
```

| Option | Description | Default |
|--------|-------------|---------|
| `url` | Dataverse/Dynamics 365 URL | (required) |
| `api_version` | Dataverse API version | `9.2` |
| `tenant_id` | Azure AD tenant ID | (required) |
| `client_id` | Azure AD application ID | (required) |
| `client_secret` | Azure AD client secret | (required) |
| `entity_name` | Custom entity name | `vcon_storage` |
| `uuid_field` | Field for vCon UUID | `vcon_uuid` |
| `data_field` | Field for vCon JSON data | `vcon_data` |
| `subject_field` | Field for vCon subject | `vcon_subject` |
| `created_at_field` | Field for creation date | `vcon_created_at` |

Requires a custom entity to be created in Dataverse with the specified fields.

---

### Space and Time

Blockchain-based verifiable storage using Space and Time's decentralized database.

```yaml
storages:
  spaceandtime:
    module: storage.spaceandtime
    options:
      name: "spaceandtime"
```

Configuration is done via environment variables:

| Environment Variable | Description |
|---------------------|-------------|
| `SXT_API_KEY` | Space and Time API key |
| `SXT_VCON_TABLENAME` | Table name (provided by SXT) |
| `SXT_VCON_TABLE_WRITE_BISCUIT` | Write biscuit token (provided by SXT) |

---

## Using Multiple Storages

You can configure multiple storage backends and assign them to different chains:

```yaml
storages:
  archive_s3:
    module: storage.s3
    options:
      aws_bucket: "vcon-archive"
      # ... other options

  query_postgres:
    module: storage.postgres
    options:
      database: "vcon_analytics"
      # ... other options

  search_milvus:
    module: storage.milvus
    options:
      collection_name: "vcon_search"
      # ... other options

chains:
  main_pipeline:
    links: [transcribe, analyze]
    storages:
      - archive_s3      # Long-term storage
      - query_postgres  # Querying
      - search_milvus   # Semantic search
    ingress_lists: [incoming]
    egress_lists: [processed]
    enabled: 1
```

## Redis Caching

The Conserver uses Redis as its primary working storage. When a vCon is requested via the API but isn't in Redis, the system will:

1. Check each configured storage backend
2. If found, return the vCon to Redis with a configurable TTL
3. Serve subsequent requests from Redis

Configure the cache TTL with the `VCON_REDIS_EXPIRY` environment variable (default: 3600 seconds / 1 hour).
