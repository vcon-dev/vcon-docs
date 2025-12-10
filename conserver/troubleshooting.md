---
description: Common issues and solutions
icon: wrench
---

# Troubleshooting

This guide covers common issues when running the Conserver and how to resolve them.

## Diagnostic Commands

### Check System Health

```bash
# Redis connectivity
redis-cli ping
# Expected: PONG

# API health
curl -H "x-conserver-api-token: $TOKEN" http://localhost:8000/api/config

# Queue status
redis-cli LLEN incoming_calls
redis-cli LLEN incoming_calls:dlq
```

### View Logs

```bash
# Docker Compose logs
docker compose logs -f conserver-api
docker compose logs -f conserver-worker

# Filter for errors
docker compose logs conserver-worker 2>&1 | grep -i error
```

---

## Connection Issues

### Redis Connection Failed

**Symptoms:**
- Workers won't start
- "Connection refused" errors
- API returns 500 errors

**Diagnosis:**
```bash
# Check Redis is running
docker compose ps redis

# Check Redis connectivity
redis-cli -h localhost -p 6379 ping

# Check Redis logs
docker compose logs redis
```

**Solutions:**

1. **Verify Redis URL:**
   ```bash
   # In .env
   REDIS_URL=redis://localhost:6379

   # For Docker networking
   REDIS_URL=redis://redis:6379
   ```

2. **Check Docker networking:**
   ```bash
   docker network ls
   docker network inspect vcon-server_default
   ```

3. **Redis memory full:**
   ```bash
   redis-cli INFO memory
   # If used_memory > maxmemory, increase or clear old data
   ```

### API Authentication Failed

**Symptoms:**
- 403 Forbidden responses
- "Invalid API Key" errors

**Solutions:**

1. **Check token configuration:**
   ```bash
   # Verify environment variable
   echo $CONSERVER_API_TOKEN

   # Check token file exists
   cat /path/to/api_tokens.txt
   ```

2. **Check header name:**
   ```bash
   # Default header
   curl -H "x-conserver-api-token: $TOKEN" ...

   # Custom header (if CONSERVER_HEADER_NAME is set)
   curl -H "x-custom-header: $TOKEN" ...
   ```

3. **For external ingress, check ingress_auth config:**
   ```yaml
   ingress_auth:
     my_ingress_list: "correct-api-key"
   ```

---

## Processing Issues

### vCons Not Being Processed

**Symptoms:**
- Queue depth keeps growing
- No worker activity in logs
- vCons stuck in ingress list

**Diagnosis:**
```bash
# Check queue depth
redis-cli LLEN incoming_calls

# Check workers are running
docker compose ps

# Check worker logs
docker compose logs conserver-worker --tail 100
```

**Solutions:**

1. **Chain not enabled:**
   ```yaml
   chains:
     main:
       enabled: 1  # Must be 1, not 0 or false
   ```

2. **Ingress list mismatch:**
   ```yaml
   chains:
     main:
       ingress_lists:
         - incoming_calls  # Must match where vCons are pushed
   ```

3. **Workers crashed:**
   ```bash
   docker compose restart conserver-worker
   ```

4. **Configuration not loaded:**
   ```bash
   # Check current config
   curl -H "x-conserver-api-token: $TOKEN" http://localhost:8000/api/config
   ```

### High DLQ Count

**Symptoms:**
- vCons accumulating in dead letter queue
- Processing errors in logs

**Diagnosis:**
```bash
# Check DLQ depth
redis-cli LLEN incoming_calls:dlq

# Get DLQ contents
curl -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/dlq?ingress_list=incoming_calls"

# Check recent errors
docker compose logs conserver-worker 2>&1 | grep -i error | tail 50
```

**Solutions:**

1. **Identify the failing link:**
   - Check logs for which link is failing
   - Look for exceptions and stack traces

2. **Common link failures:**

   **API key missing/invalid:**
   ```yaml
   links:
     analyze:
       options:
         OPENAI_API_KEY: ${OPENAI_API_KEY}  # Check env var is set
   ```

   **External service down:**
   ```bash
   # Test connectivity
   curl https://api.openai.com/v1/models -H "Authorization: Bearer $OPENAI_API_KEY"
   ```

   **Timeout:**
   ```yaml
   chains:
     main:
       timeout: 600  # Increase for long transcriptions
   ```

3. **Reprocess after fixing:**
   ```bash
   curl -X POST -H "x-conserver-api-token: $TOKEN" \
     "http://localhost:8000/api/dlq/reprocess?ingress_list=incoming_calls"
   ```

### Link Timeout

**Symptoms:**
- "Processing timeout" errors
- vCons moving to DLQ after timeout period

**Solutions:**

1. **Increase chain timeout:**
   ```yaml
   chains:
     main:
       timeout: 600  # Seconds
   ```

2. **Optimize processing:**
   - Use faster models
   - Skip unnecessary processing steps
   - Add sampling to process fewer vCons

3. **Check external API performance:**
   ```bash
   # Time an API call
   time curl https://api.example.com/endpoint
   ```

---

## Transcription Issues

### No Transcription Generated

**Symptoms:**
- vCon has no transcript analysis
- Transcription link completes but adds nothing

**Diagnosis:**
```bash
# Check vCon dialogs
curl -H "x-conserver-api-token: $TOKEN" \
  "http://localhost:8000/api/vcon/$VCON_UUID" | jq '.dialog'
```

**Solutions:**

1. **Check dialog type:**
   ```json
   // Dialog must be type "recording"
   {"type": "recording", "url": "https://..."}
   ```

2. **Check duration requirement:**
   ```yaml
   links:
     transcribe:
       options:
         minimum_duration: 30  # Audio must be >= 30 seconds
   ```

3. **Check audio URL is accessible:**
   ```bash
   curl -I "https://your-audio-url.mp3"
   ```

4. **Check API key:**
   ```yaml
   links:
     deepgram:
       options:
         DEEPGRAM_KEY: ${DEEPGRAM_KEY}  # Verify env var
   ```

### Transcription Already Exists

**Symptoms:**
- "Dialog already transcribed" in logs
- Link skipping dialogs

**Explanation:**
Links skip processing if analysis already exists (idempotency).

**If you need to re-transcribe:**
1. Delete the vCon and re-submit
2. Or create a new link with different `analysis_type`

---

## Storage Issues

### Storage Write Failed

**Symptoms:**
- "Failed to save" errors
- vCon not appearing in storage backend

**Diagnosis:**
```bash
# Check storage backend connectivity
# For PostgreSQL:
psql -h localhost -U postgres -d vcons -c "SELECT 1"

# For S3:
aws s3 ls s3://your-bucket/
```

**Solutions:**

1. **Check credentials:**
   ```yaml
   storages:
     postgres:
       options:
         password: ${POSTGRES_PASSWORD}  # Verify env var
   ```

2. **Check network connectivity:**
   ```bash
   # From container
   docker exec conserver-worker ping postgres
   ```

3. **Check permissions:**
   - Database user has write permissions
   - S3 IAM role allows PutObject

### vCon Not Found

**Symptoms:**
- 404 when fetching vCon
- "vCon not found" errors

**Diagnosis:**
```bash
# Check Redis
redis-cli EXISTS vcon:$VCON_UUID

# Check sorted set
redis-cli ZSCORE vcons vcon:$VCON_UUID
```

**Solutions:**

1. **vCon expired from Redis:**
   - Check `VCON_REDIS_EXPIRY` setting
   - vCon should be auto-fetched from storage if configured

2. **vCon never stored:**
   - Check chain has storage backends configured
   - Check storage write didn't fail

3. **Wrong UUID format:**
   - Ensure UUID includes hyphens
   - Check for typos

---

## Configuration Issues

### Configuration Not Loading

**Symptoms:**
- Changes to config.yml not taking effect
- Wrong settings being used

**Solutions:**

1. **Check config file path:**
   ```bash
   echo $CONSERVER_CONFIG_FILE
   ```

2. **Validate YAML syntax:**
   ```bash
   python -c "import yaml; yaml.safe_load(open('config.yml'))"
   ```

3. **Restart workers:**
   ```bash
   docker compose restart conserver-worker
   ```

4. **Use API to update:**
   ```bash
   curl -X POST -H "x-conserver-api-token: $TOKEN" \
     -H "Content-Type: application/json" \
     -d @config.json \
     http://localhost:8000/api/config
   ```

### Import/Module Not Found

**Symptoms:**
- "ModuleNotFoundError" in logs
- Dynamic imports failing

**Solutions:**

1. **Check imports section:**
   ```yaml
   imports:
     my_module:
       module: my_module
       pip_name: my-package>=1.0.0  # Correct package name
   ```

2. **Check pip_name format:**
   ```yaml
   # PyPI
   pip_name: package-name>=1.0.0

   # GitHub
   pip_name: git+https://github.com/org/repo.git@tag
   ```

3. **Check network access:**
   - Container can reach PyPI/GitHub
   - No firewall blocking

---

## Performance Issues

### Slow Processing

**Symptoms:**
- High processing times
- Queue backing up

**Solutions:**

1. **Scale workers:**
   ```bash
   docker compose up -d --scale conserver-worker=5
   ```

2. **Use sampling:**
   ```yaml
   links:
     analyze:
       options:
         sampling_rate: 0.1  # Process 10%
   ```

3. **Use faster models:**
   ```yaml
   links:
     analyze:
       options:
         model: "gpt-3.5-turbo"  # Faster than gpt-4
   ```

4. **Skip unnecessary links:**
   ```yaml
   chains:
     fast_chain:
       links:
         - transcribe
         # Skip heavy analysis for speed
   ```

### Memory Issues

**Symptoms:**
- OOM errors
- Container restarts

**Solutions:**

1. **Increase container memory:**
   ```yaml
   services:
     conserver-worker:
       deploy:
         resources:
           limits:
             memory: 8G
   ```

2. **Configure Redis maxmemory:**
   ```bash
   redis-cli CONFIG SET maxmemory 2gb
   redis-cli CONFIG SET maxmemory-policy allkeys-lru
   ```

3. **Use diet link to reduce vCon size:**
   ```yaml
   links:
     slim_down:
       module: links.diet
       options:
         remove_dialog_bodies: true
   ```

---

## Getting Help

If you can't resolve an issue:

1. **Check logs carefully** - Most errors have clear messages
2. **Search GitHub issues** - Someone may have had the same problem
3. **Create a GitHub issue** with:
   - Conserver version
   - Configuration (sanitized)
   - Error messages
   - Steps to reproduce
