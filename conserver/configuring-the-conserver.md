---
description: A Complete Guide
icon: wrench
---

# Configuring the Conserver

The Conserver is configured through two mechanisms:

1. **Environment Variables** - Server-level settings (Redis, API keys, paths)
2. **YAML Configuration File** - Processing configuration (chains, links, storage)

## Environment Variables

Set these in your `.env` file or system environment.

### Core Settings

| Variable | Description | Default |
|----------|-------------|---------|
| `REDIS_URL` | Redis connection URL | `redis://localhost` |
| `CONSERVER_CONFIG_FILE` | Path to YAML config file | `./example_config.yml` |
| `HOSTNAME` | Server hostname | `http://localhost:8000` |
| `ENV` | Environment name (dev/staging/prod) | `dev` |
| `LOG_LEVEL` | Logging level | `DEBUG` |

### API Settings

| Variable | Description | Default |
|----------|-------------|---------|
| `CONSERVER_API_TOKEN` | Main API authentication token | (none) |
| `CONSERVER_API_TOKEN_FILE` | Path to file with API tokens (one per line) | (none) |
| `CONSERVER_HEADER_NAME` | HTTP header name for API token | `x-conserver-api-token` |
| `API_ROOT_PATH` | API URL prefix | `/api` |

### Redis/Caching Settings

| Variable | Description | Default |
|----------|-------------|---------|
| `VCON_REDIS_EXPIRY` | Cache TTL for vCons in Redis (seconds) | `3600` (1 hour) |
| `VCON_INDEX_EXPIRY` | Search index TTL (seconds) | `86400` (24 hours) |
| `VCON_SORTED_SET_NAME` | Name of Redis sorted set for vCons | `vcons` |
| `VCON_SORTED_FORCE_RESET` | Reset sorted set on startup | `true` |
| `TICK_INTERVAL` | Processing loop interval (ms) | `5000` |

### External Service API Keys

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | OpenAI API key |
| `DEEPGRAM_KEY` | Deepgram speech-to-text API key |

### Example .env File

```bash
# Core
REDIS_URL=redis://localhost:6379
CONSERVER_CONFIG_FILE=./config.yml
ENV=production
LOG_LEVEL=INFO

# API
CONSERVER_API_TOKEN=your-secret-token
API_ROOT_PATH=/api

# Caching
VCON_REDIS_EXPIRY=3600
VCON_INDEX_EXPIRY=86400

# External APIs
OPENAI_API_KEY=sk-...
DEEPGRAM_KEY=...
```

---

## YAML Configuration File

The YAML configuration file defines processing chains, links, storage backends, and authentication. The Conserver looks for the path specified in `CONSERVER_CONFIG_FILE` (default: `./example_config.yml`).

### Configuration File Structure

```yaml
# External partner authentication
ingress_auth:
  ingress_list_name: "api-key" or ["key1", "key2"]

# Dynamic module imports
imports:
  import_name:
    module: module_name
    pip_name: package-name

# Processing link definitions
links:
  link_name:
    module: links.module_name
    options: {}

# Storage backend definitions
storages:
  storage_name:
    module: storage.backend_name
    options: {}

# Tracer definitions
tracers:
  tracer_name:
    module: tracers.module_name
    options: {}

# Processing chain definitions
chains:
  chain_name:
    links: [link1, link2]
    storages: [storage1]
    ingress_lists: [queue1]
    egress_lists: [queue2]
    enabled: 1
    timeout: 300

# Follower configuration (federation)
followers:
  follower_name:
    url: https://upstream-server
    egress_list: source_queue
    follower_ingress_list: target_queue
```

---

## Section Reference

### ingress_auth

Configures API keys for external partner access to the `/vcon/external-ingress` endpoint.

```yaml
ingress_auth:
  # Single key per ingress list
  customer_data: "customer-api-key"

  # Multiple keys per ingress list
  partner_ingress:
    - "partner-key-1"
    - "partner-key-2"
    - "partner-key-3"
```

Each key grants access only to its designated ingress list. Partners cannot access other API endpoints.

---

### imports

Dynamically imports Python packages at runtime. Useful for:
- Installing missing dependencies automatically
- Using external link/storage packages
- Managing version requirements

```yaml
imports:
  # PyPI package
  custom_analysis:
    module: my_analysis_module
    pip_name: my-analysis-package>=1.0.0

  # GitHub repository
  github_link:
    module: github_link
    pip_name: git+https://github.com/org/repo.git@v2.0.0

  # GitHub with branch
  dev_link:
    module: dev_link
    pip_name: git+https://github.com/org/repo.git@main

  # Version ranges
  constrained:
    module: constrained_module
    pip_name: package>=1.0.0,<2.0.0
```

The Conserver will automatically install missing packages when first referenced.

---

### links

Links are the processing units of the conserver. Each link is a module that performs a specific operation on a vCon. See [Standard Links](standard-links.md) for built-in options.

```yaml
links:
  # Built-in link with options
  deepgram:
    module: links.deepgram_link
    options:
      DEEPGRAM_KEY: your_key_here
      minimum_duration: 30
      api:
        model: "nova-2"
        smart_format: true
        detect_language: true

  # External link (auto-installed via imports)
  custom_analyzer:
    module: custom_analysis
    pip_name: my-analysis-package
    options:
      model: "gpt-4"

  # Multiple instances of same link with different configs
  summary_brief:
    module: links.analyze
    options:
      OPENAI_API_KEY: your_key_here
      prompt: "Summarize in one sentence"
      analysis_type: "brief_summary"

  summary_detailed:
    module: links.analyze
    options:
      OPENAI_API_KEY: your_key_here
      prompt: "Provide a detailed analysis"
      analysis_type: "detailed_summary"
```

Each link configuration needs:

* A unique name (e.g., 'deepgram', 'analyze')
* The module path that implements the link functionality
* An options dictionary containing the link's specific configuration

---

### storages

Storages define where vCons are saved after processing. See [Storage](storage.md) for all backends.

```yaml
storages:
  postgres:
    module: storage.postgres
    options:
      user: postgres
      password: your_password
      host: your_host
      port: 5432
      database: postgres

  s3:
    module: storage.s3
    options:
      aws_access_key_id: your_key_id
      aws_secret_access_key: your_secret
      aws_bucket: your_bucket

  milvus:
    module: storage.milvus
    options:
      host: localhost
      port: "19530"
      api_key: sk-...
      embedding_model: text-embedding-3-small
```

Each storage needs:

* A unique name
* The storage module implementation
* Connection and authentication options specific to the storage type

---

### tracers

Defines tracer modules for auditing and compliance tracking.

```yaml
tracers:
  jlinc:
    module: tracers.jlinc
    options:
      data_store_api_url: http://jlinc-server:9090
      data_store_api_key: "key"
      archive_api_url: http://jlinc-server:9090
      archive_api_key: "key"
      system_prefix: "VCONProd"
      hash_event_data: true
      dlq_vcon_on_error: true
```

---

### chains

Chains are where you define your processing workflows. They connect links together and specify where the results should be stored:

```yaml
chains:
  transcription_chain:
    # Links to execute in order
    links:
      - deepgram
      - analyze
      - webhook_store

    # Input Redis lists where new vCons arrive
    ingress_lists:
      - transcription_input

    # Storage backends for processed vCons
    storages:
      - postgres
      - s3

    # Output Redis lists for downstream processing
    egress_lists:
      - transcription_output

    # Enable/disable this chain
    enabled: 1

    # Processing timeout per vCon (seconds)
    timeout: 300
```

A chain configuration includes:

* The links to execute, in order
* Input lists (ingress\_lists) where new vCons arrive
* Storage locations for the processed vCons
* Output lists (egress\_lists) for downstream processing
* An enabled flag and optional timeout

**Chain Processing Flow:**
1. vCon UUID arrives in an ingress list
2. Links execute sequentially (any can stop processing by returning `None`)
3. vCon is stored in all configured storage backends
4. UUID is added to egress lists
5. If processing fails, UUID moves to DLQ (`{ingress_list}:dlq`)

---

### followers

Followers allow one conserver to monitor and process vCons from another conserver for federated deployments:

```yaml
followers:
  remote_conserver:
    # Upstream server URL
    url: "https://remote-conserver.example.com"

    # Authentication token for upstream
    auth_token: "your_auth_token"

    # Remote egress list to pull from
    egress_list: "remote_output"

    # Local ingress list to add vCons to
    follower_ingress_list: "local_input"

    # Polling interval in seconds
    pulling_interval: 60

    # Number of vCons to fetch per request
    fetch_vcon_limit: 10
```

Each follower needs:

* The URL of the remote conserver
* Authentication credentials
* The remote list to monitor (egress\_list)
* The local list to populate (follower\_ingress\_list)
* Polling configuration (interval and batch size)

---

## Complete Example

```yaml
# External partner authentication
ingress_auth:
  partner_data:
    - "partner-key-abc"
    - "partner-key-xyz"
  internal_systems: "internal-key-123"

# Dynamic imports
imports:
  sentiment_analyzer:
    module: sentiment
    pip_name: vcon-sentiment>=1.0.0

# Links configuration
links:
  transcribe:
    module: links.deepgram_link
    options:
      DEEPGRAM_KEY: ${DEEPGRAM_KEY}
      minimum_duration: 30
      api:
        model: "nova-2"
        smart_format: true

  summarize:
    module: links.analyze
    options:
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      prompt: "Summarize this conversation in 3 bullet points."
      analysis_type: "summary"
      model: "gpt-4-turbo"

  detect_complaints:
    module: links.check_and_tag
    options:
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      tag_name: "complaint"
      tag_value: "detected"
      evaluation_question: "Does this contain a customer complaint?"

# Storage backends
storages:
  postgres:
    module: storage.postgres
    options:
      database: "vcons"
      user: "postgres"
      password: ${POSTGRES_PASSWORD}
      host: "postgres"
      port: 5432

  s3:
    module: storage.s3
    options:
      aws_access_key_id: ${AWS_ACCESS_KEY_ID}
      aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
      aws_bucket: "vcon-archive"

# Processing chains
chains:
  main:
    links:
      - transcribe
      - summarize
      - detect_complaints
    storages:
      - postgres
      - s3
    ingress_lists:
      - incoming_calls
      - partner_data
    egress_lists:
      - processed
    enabled: 1
    timeout: 600
```

---

## Environment Variable Substitution

The configuration supports environment variable substitution using `${VAR_NAME}` syntax:

```yaml
links:
  analyze:
    module: links.analyze
    options:
      OPENAI_API_KEY: ${OPENAI_API_KEY}
```

This allows sensitive values to be kept in environment variables rather than the config file.

---

## Configuration Best Practices

1. **Use meaningful names** for your chains, links, and storage configurations to make the system easier to understand and maintain.

2. **Organize links logically** - arrange links in order where each step builds on the previous ones.

3. **Use multiple storage backends** when needed - for example, storing in both S3 for long-term storage and Postgres for quick querying.

4. **Configure appropriate timeouts** for your chains based on the expected processing time of your links.

5. **Use environment variables** for sensitive values like API keys and passwords.

6. **Use the follower configuration** when you need to process vCons across multiple conserver instances.

---

## Hot Reloading

The configuration file is loaded at startup and can be updated via the API endpoint `/config`. Changes take effect immediately for new vCon processing:

```bash
curl -X POST "http://localhost:8000/api/config" \
  -H "x-conserver-api-token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d @new_config.json
```

Remember that the conserver uses Redis as its working storage, so all the lists referenced in ingress\_lists and egress\_lists are Redis lists.
