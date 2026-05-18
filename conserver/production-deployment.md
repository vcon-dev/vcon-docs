---
description: Deploying the Conserver in Production
icon: rocket
---

# Production Deployment

This guide covers deploying the Conserver in production environments with considerations for scalability, reliability, and security.

## Prerequisites

- Docker and Docker Compose
- Redis server (or Redis cluster for high availability)
- Storage backends configured (PostgreSQL, S3, etc.)
- Domain name and TLS certificates
- Monitoring infrastructure (optional but recommended)

## Image strategy: two Dockerfiles

As of the May 2026 image optimization (`docker/Dockerfile.api` and `docker/Dockerfile.conserver`), the conserver ships **two separate images**:

| Image | What it contains | Use it for |
|-------|------------------|------------|
| **`Dockerfile.api`** | FastAPI app, storage backends, vCon library — no audio/ML stack | The API tier. Light, starts fast, scales horizontally. |
| **`Dockerfile.conserver`** | Everything in the API image *plus* transformers, openai, deepgram, ffmpeg, pydub | The worker tier. Heavy but only needed where audio processing and LLM calls run. |

Both images use [**uv**](https://github.com/astral-sh/uv) (Astral's Python package manager) for reproducible builds and place the virtualenv at `/opt/venv` so it survives volume mounts.

In production, deploy the API image for the `conserver-api` service and the conserver image for the `conserver-worker` service. The example below does exactly that.

## Architecture Overview

```
                    ┌─────────────────┐
                    │   Load Balancer │
                    │    (nginx/ALB)  │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
    │Conserver│        │Conserver│        │Conserver│
    │   API   │        │   API   │        │   API   │
    └────┬────┘        └────┬────┘        └────┬────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                    ┌────────▼────────┐
                    │      Redis      │
                    │  (Queues/Cache) │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
    │PostgreSQL│        │   S3    │        │ Milvus  │
    └─────────┘        └─────────┘        └─────────┘
```

## Docker Compose Production Setup

### docker-compose.yml

```yaml
version: '3.8'

services:
  conserver-api:
    image: your-registry/conserver-api:latest    # built from docker/Dockerfile.api
    command: uvicorn api:app --host 0.0.0.0 --port 8000
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
    environment:
      - REDIS_URL=redis://redis:6379
      - CONSERVER_CONFIG_FILE=/app/config.yml
      - CONSERVER_API_TOKEN_FILE=/run/secrets/api_tokens
      - LOG_LEVEL=INFO
      - ENV=production
    volumes:
      - ./config.yml:/app/config.yml:ro
    secrets:
      - api_tokens
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    depends_on:
      redis:
        condition: service_healthy

  conserver-worker:
    image: your-registry/conserver:latest        # built from docker/Dockerfile.conserver
    command: python main.py
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G
    environment:
      - REDIS_URL=redis://redis:6379
      - CONSERVER_CONFIG_FILE=/app/config.yml
      - LOG_LEVEL=INFO
      - ENV=production
      - CONSERVER_WORKERS=4              # forked worker processes per container
      - CONSERVER_PARALLEL_STORAGE=true  # write to storages concurrently
      - OPENAI_API_KEY_FILE=/run/secrets/openai_key
      - DEEPGRAM_KEY_FILE=/run/secrets/deepgram_key
    volumes:
      - ./config.yml:/app/config.yml:ro
    secrets:
      - openai_key
      - deepgram_key
    depends_on:
      redis:
        condition: service_healthy

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 2gb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - conserver-api

volumes:
  redis_data:

secrets:
  api_tokens:
    file: ./secrets/api_tokens.txt
  openai_key:
    file: ./secrets/openai_key.txt
  deepgram_key:
    file: ./secrets/deepgram_key.txt
```

### nginx.conf

```nginx
upstream conserver {
    least_conn;
    server conserver-api:8000;
}

server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;

    location /api/ {
        limit_req zone=api burst=200 nodelay;

        proxy_pass http://conserver;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts for long-running requests
        proxy_connect_timeout 60s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;

        # Buffer settings
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }

    # Health check endpoint (no auth required)
    location /health {
        proxy_pass http://conserver/health;
        proxy_http_version 1.1;
    }
}
```

---

## Scaling Considerations

### Horizontal Scaling

The Conserver supports horizontal scaling because:
- All state is stored in Redis
- Multiple instances can process from the same queues
- API requests are stateless

Scale workers based on queue depth:
```bash
# Monitor queue depth
redis-cli LLEN incoming_calls

# Scale workers
docker compose up -d --scale conserver-worker=10
```

### Redis Configuration

For production Redis deployments:

```conf
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru
appendonly yes
appendfsync everysec

# For Redis Cluster
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
```

### Queue Monitoring

Monitor queue lengths to detect backlogs:

```python
import redis

r = redis.Redis(host='localhost', port=6379)

# Check ingress queue depth
queue_length = r.llen('incoming_calls')
print(f"Queue depth: {queue_length}")

# Check DLQ for failures
dlq_length = r.llen('incoming_calls:dlq')
print(f"DLQ depth: {dlq_length}")
```

---

## Security Hardening

### API Token Management

1. **Use token files instead of environment variables:**
   ```yaml
   environment:
     - CONSERVER_API_TOKEN_FILE=/run/secrets/api_tokens
   ```

2. **Rotate tokens regularly:**
   ```bash
   # Generate new token
   openssl rand -hex 32 > secrets/api_tokens.txt

   # Restart API containers
   docker compose restart conserver-api
   ```

3. **Use separate tokens for different purposes:**
   - Internal API token for system operations
   - Partner-specific tokens via `ingress_auth`

### Network Security

1. **Isolate Redis:**
   ```yaml
   networks:
     internal:
       internal: true
     external:

   services:
     redis:
       networks:
         - internal
     conserver-api:
       networks:
         - internal
         - external
   ```

2. **Enable Redis AUTH:**
   ```yaml
   redis:
     command: redis-server --requirepass ${REDIS_PASSWORD}
   ```

3. **Use TLS for external connections**

### Secret Management

Consider using:
- Docker secrets (as shown above)
- HashiCorp Vault
- AWS Secrets Manager
- Kubernetes secrets

---

## Monitoring and Observability

### Health Checks

The Conserver exposes three public system endpoints (no auth required) for monitoring:

```bash
# Health check — basic up/down + version
curl http://localhost:8000/health

# Build metadata — version, git commit, build time
curl http://localhost:8000/version

# Queue depth (point load balancers / autoscalers at this)
curl "http://localhost:8000/stats/queue?list_name=incoming_calls"

# Redis liveness
redis-cli ping
```

These endpoints intentionally live at the application root (not under `API_ROOT_PATH`), so they're stable regardless of how you've configured the API prefix.

### Metrics Integration

Every standard link and storage emits OpenTelemetry spans and metrics (latency, errors, cache hits where applicable). Wire up an OTLP collector — see [vCon MCP Adapters](../tools/vcon-mcp-adapters.md) for a turnkey integration — and the conserver will populate your dashboards out of the box.

Per-link metrics include:

```python
# Automatic OTEL attributes per link:
# - link_name
# - vcon_uuid (when relevant)
# - status (ok | error | skipped)
# - duration_ms
# - cache_hit (for transcription links with caching)
# - Storage operation latency
```

### Log Aggregation

Configure structured JSON logging:

```yaml
environment:
  - LOG_LEVEL=INFO
  - LOG_FORMAT=json
```

Logs include:
- Request IDs for tracing
- Processing times
- Error details with stack traces
- vCon UUIDs for correlation

### Alerting

Set up alerts for:

| Metric | Threshold | Action |
|--------|-----------|--------|
| Queue depth > 1000 | Warning | Scale workers |
| DLQ depth > 100 | Critical | Investigate failures |
| API latency p99 > 5s | Warning | Check resources |
| Error rate > 5% | Critical | Check logs |

---

## Graceful Shutdown

The Conserver handles SIGTERM for graceful shutdown:

1. Stops accepting new vCons
2. Completes in-flight processing
3. Returns unprocessed items to queues
4. Closes connections cleanly

Configure Docker stop timeout:

```yaml
services:
  conserver-worker:
    stop_grace_period: 5m  # Allow time for long transcriptions
```

---

## Backup and Recovery

### Redis Persistence

Enable AOF for durability:

```yaml
redis:
  command: redis-server --appendonly yes
  volumes:
    - redis_data:/data
```

### Backup Strategy

1. **Redis RDB snapshots:**
   ```bash
   redis-cli BGSAVE
   ```

2. **Storage backend backups:**
   - PostgreSQL: pg_dump
   - S3: Enable versioning
   - Elasticsearch: Snapshot API

3. **Configuration backup:**
   ```bash
   # Backup config via API
   curl -H "x-conserver-api-token: $TOKEN" \
     http://localhost:8000/api/config > config_backup.json
   ```

### Disaster Recovery

1. Deploy Redis with persistence
2. Use storage backends with replication
3. Keep configuration in version control
4. Document recovery procedures

---

## Deployment Checklist

### Pre-deployment

- [ ] Configure TLS certificates
- [ ] Set up API tokens securely
- [ ] Configure storage backends
- [ ] Test configuration locally
- [ ] Set up monitoring/alerting
- [ ] Document rollback procedures

### Deployment

- [ ] Deploy Redis first
- [ ] Deploy workers
- [ ] Deploy API servers
- [ ] Verify health checks pass
- [ ] Test sample vCon processing

### Post-deployment

- [ ] Monitor queue depths
- [ ] Check error rates
- [ ] Verify storage writes
- [ ] Test API endpoints
- [ ] Confirm metrics flowing

---

## Troubleshooting

### Common Issues

**Workers not processing:**
```bash
# Check Redis connectivity
docker exec conserver-worker redis-cli -h redis ping

# Check queue contents
docker exec conserver-worker redis-cli -h redis LRANGE incoming_calls 0 10
```

**High DLQ count:**
```bash
# Check DLQ contents
curl -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/dlq?ingress_list=incoming_calls"

# Reprocess after fixing issues
curl -X POST -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/dlq/reprocess?ingress_list=incoming_calls"
```

**Memory issues:**
```bash
# Check Redis memory
redis-cli INFO memory

# Check container memory
docker stats
```

See [Troubleshooting](troubleshooting.md) for more detailed solutions.
