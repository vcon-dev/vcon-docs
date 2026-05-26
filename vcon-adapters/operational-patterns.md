---
description: >-
  How to deliver vCons reliably in production — signing, idempotency, retries,
  dead-letter queues, health, metrics.
---

# ⚙️ Operational Patterns

The vCon construction story is well-defined: use the [`vcon`](../vcon-library/) library helpers and stay on the [Spec Compliance Checklist](spec-compliance-checklist.md). The _delivery_ story — how a built vCon actually leaves your adapter and reaches a conserver, MCP server, archive, or downstream pipeline — is where most adapters historically went wrong. This page documents the patterns the [vcon-adapter-template](https://github.com/vcon-dev/vcon-adapter-template) ships with and that new adapters should adopt.

Every pattern below is implemented in code at [`webhook_delivery.py`](https://github.com/vcon-dev/vcon-adapter-template/blob/main/src/__ADAPTER_PACKAGE__/webhook_delivery.py) and [`health_server.py`](https://github.com/vcon-dev/vcon-adapter-template/blob/main/src/__ADAPTER_PACKAGE__/health_server.py). If you scaffolded from the template, you already have this — read this page to know _why_ it works the way it does.

## The delivery contract

An adapter SHOULD POST each vCon as a single JSON body to one or more configured webhook endpoints. The HTTP request looks like this:

```http
POST /vcons HTTP/1.1
Content-Type: application/json
Idempotency-Key: 6f1c5a8b-3e2d-4a1f-9c4f-7b8a2f0e1d3c
X-Hub-Signature-256: sha256=2c8f9...

{"vcon":"0.4.0","uuid":"6f1c5a8b-3e2d-4a1f-9c4f-7b8a2f0e1d3c", ...}
```

Three things are non-negotiable for production:

1. The body is **compact JSON** — no extra whitespace. The signature is computed over the exact bytes on the wire, so any reformatting on either side breaks verification.
2. The `Idempotency-Key` is the **vCon's `uuid`**. Same vCon ⇒ same key ⇒ the receiver MUST treat retries as no-ops.
3. The `X-Hub-Signature-256` header is computed exactly the same way GitHub computes it for webhooks. This is intentional — most receivers already have well-tested verification code for this format.

## HMAC body signing

Every adapter SHOULD sign its outgoing vCon webhooks with `X-Hub-Signature-256: sha256=<hexdigest>`. The signing recipe:

```python
import hashlib
import hmac

def sign(body: bytes, secret: str) -> str:
    mac = hmac.new(secret.encode("utf-8"), body, hashlib.sha256)
    return "sha256=" + mac.hexdigest()
```

On the receiver side, the verification flow is:

1. Read the raw request body (do NOT re-serialize the parsed JSON — the bytes must match the signed bytes exactly)
2. Recompute the HMAC with the shared secret
3. Compare in constant time with `hmac.compare_digest`

The shared secret lives in `VCON_WEBHOOK_HMAC_SECRET` (or per-endpoint in `config.yaml`). It is independent of any JWS keys used for vCon-level signing — they solve different problems (transport authentication vs. content provenance).

## Idempotency

Every webhook delivery MUST carry an `Idempotency-Key` header equal to the vCon's `uuid`. This serves two purposes:

* **Retries are safe.** When the adapter retries after a 5xx response or a timeout, the receiver sees the same key and can skip work it has already done.
* **Replay is detectable.** If an attacker captures and replays a signed payload, the receiver can spot the duplicate `uuid` and reject it.

vCon UUIDs are v4 — sufficiently random that collisions are operationally impossible. The receiver can use them as primary keys directly.

## Retries with exponential backoff

Networks fail. Downstream services are deployed, restart, get overloaded. The template's delivery layer retries up to `max_attempts` times (default 5) with exponential backoff:

| Attempt | Wait before retry |
| ------- | ----------------- |
| 1       | — (immediate)     |
| 2       | 1 s               |
| 3       | 2 s               |
| 4       | 4 s               |
| 5       | 8 s               |

Backoff doubles each attempt, capped at `max_backoff_seconds` (default 60). Both knobs are configurable in `config.yaml`:

```yaml
webhook:
  retry:
    max_attempts: 5
    initial_backoff_seconds: 1
    max_backoff_seconds: 60
```

A response counts as success when the status code is `2xx`. Anything else — `4xx`, `5xx`, timeout, connection error — counts as a failure and triggers the next retry (or the dead-letter queue if attempts are exhausted).

> Note: 4xx responses are retried by default. If the receiver returns a 4xx for a malformed vCon, retrying won't change the outcome — but it also won't hurt, and it keeps the delivery path simple. If you need 4xx fast-fail semantics for your downstream, customize the retry predicate.

## Dead-letter queue

When all retry attempts are exhausted across all configured endpoints, the vCon is written to disk under `dead_letter_path` (default `./dlq`) as `<uuid>.vcon.json`. This guarantees that:

* No vCon is ever silently lost
* An operator can inspect, requeue, or hand-deliver failed messages
* Audit trails survive receiver outages

The DLQ is a directory of JSON files, not a queue service. This keeps adapters small and stateless. Plug in your own re-injection cron, ops dashboard, or alerting on the directory's size — see the [Conserver](../conserver/) docs for one way to wire DLQ replay into a broader pipeline.

The `vcons_dlq_total` Prometheus counter (see below) tracks DLQ writes — alert on it.

## Multiple endpoints (fan-out)

The template treats `endpoints` as a list. A vCon is delivered to **every** endpoint in the list; success on at least one keeps it out of the DLQ.

```yaml
webhook:
  endpoints:
    - url: https://primary.example/vcons
      hmac_secret: ${PRIMARY_HMAC_SECRET}
      timeout_seconds: 30
    - url: https://backup-archive.example/vcons
      hmac_secret: ${ARCHIVE_HMAC_SECRET}
      timeout_seconds: 60
```

This is the standard pattern for shipping the same conversation to a primary processing pipeline and a long-term archive simultaneously. Each endpoint has its own HMAC secret — the body is signed independently per endpoint.

## Health and metrics

The template starts an HTTP server on `${SERVER_HOST}:${SERVER_PORT}` (default `0.0.0.0:8000`) exposing two endpoints:

### `/healthz`

Returns `200 OK` with `{"status": "ok"}` JSON. Designed for Kubernetes liveness/readiness probes and load-balancer health checks. Wire it into your Dockerfile's `HEALTHCHECK` instruction (the template already does).

### `/metrics`

Returns Prometheus exposition format. Three counters out of the box:

| Counter                 | Increments on                              | Labels     |
| ----------------------- | ------------------------------------------ | ---------- |
| `vcons_built_total`     | Every successful vCon construction         | —          |
| `vcons_delivered_total` | Every 2xx response from a webhook endpoint | `endpoint` |
| `vcons_dlq_total`       | Every write to the dead-letter queue       | —          |

A reasonable alerting rule pair:

```yaml
- alert: VconAdapterDlqGrowing
  expr: rate(vcons_dlq_total[5m]) > 0
  for: 5m
- alert: VconAdapterDeliveryStalled
  expr: rate(vcons_delivered_total[10m]) == 0 and rate(vcons_built_total[10m]) > 0
  for: 10m
```

The first catches the failure mode where the receiver is broken; the second catches the failure mode where the adapter is broken upstream.

## JWS signing of vCons (optional, content-level)

HMAC webhook signing authenticates the _transport_ — it tells the receiver "this body came from someone holding the shared secret." It does not travel with the vCon if the receiver later forwards it.

For end-to-end provenance, sign the vCon itself with JWS (JSON Web Signature, RS256). The template includes an optional signing path; enable it in config:

```yaml
vcon:
  signing:
    enabled: true
    private_key_path: ${VCON_SIGNING_KEY_PATH}
    key_id: ${VCON_SIGNING_KEY_ID}
```

The signed form is a JWS with the vCon as its detached payload. Receivers verify with the corresponding public key — usually distributed out of band or via a JWKS endpoint. JWS-signed vCons carry their authenticity across re-forwarding, archival, and downstream tooling.

JWS signing is independent of HMAC webhook signing. Most production deployments use both: HMAC for the transport hop, JWS for content provenance.

## Configuration surface

The template's [`config.example.yaml`](https://github.com/vcon-dev/vcon-adapter-template/blob/main/config.example.yaml) is the canonical source of truth. The relevant sections for operational concerns:

```yaml
webhook:
  endpoints:
    - url: ${VCON_WEBHOOK_URL}
      hmac_secret: ${VCON_WEBHOOK_HMAC_SECRET}
      timeout_seconds: 30
  retry:
    max_attempts: 5
    initial_backoff_seconds: 1
    max_backoff_seconds: 60
  dead_letter_path: /app/vcons/dlq

server:
  host: 0.0.0.0
  port: 8000

logging:
  level: INFO
  format: json
```

`${ENV_VAR}` substitution happens at startup, so secrets never need to be in the YAML file itself.

## Logging

The template uses [`structlog`](https://www.structlog.org/) with JSON output by default. Every delivery attempt logs a structured event:

```json
{"event":"delivered","url":"https://...","uuid":"6f1c...","attempt":1,"level":"info"}
{"event":"delivery_failed","url":"https://...","status":502,"attempt":2,"level":"warning"}
{"event":"dlq_write","uuid":"6f1c...","path":"/app/vcons/dlq/6f1c.vcon.json","level":"error"}
```

This is grep-friendly _and_ fits cleanly into Loki/Datadog/Splunk pipelines. The `uuid` field is the join key for tracing a single vCon end-to-end.

## What this page is not

This is the operational story for the **template's delivery layer**. If you're shipping vCons over a different transport (Kafka, NATS, gRPC, files on a shared filesystem, an S3 bucket), the principles above still apply — sign the bytes, key off the UUID for idempotency, retry with backoff, persist failures — but the concrete code will differ. See the [Adapter Development Guide](vcon-adapter-development-guide.md) for non-webhook delivery patterns.
