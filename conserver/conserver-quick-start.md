---
description: Clone vcon-server, start it with Docker Compose, and push one vCon through a working chain.
---

# 🐰 Conserver Quick Start

This page starts the conserver on one machine with the Docker Compose file that ships in [vcon-server](https://github.com/vcon-dev/vcon-server), then sends one vCon through a one-link chain. The checked-in `docker-compose.yml` is a development stack: it bind-mounts the repository into the containers and restarts the worker when a Python file changes. For a deployment, see [Production Deployment](production-deployment.md).

You need Docker with Compose, and git.

## Clone and configure

```bash
git clone https://github.com/vcon-dev/vcon-server.git
cd vcon-server
cp .env.example .env
cp example_config.yml config.yml
docker network create conserver
```

The compose file bind-mounts `./config.yml`. If the file does not exist, Docker creates a directory with that name and the conserver cannot start, so copy it before the first `up`.

Edit `.env`:

| Variable | Set it to |
| -------- | --------- |
| `CONSERVER_API_TOKEN` | A secret you choose, for example the output of `openssl rand -hex 32`. Leave it blank to switch API authentication off. |
| `CONSERVER_CONFIG_FILE` | `./config.yml`. The shipped value is empty, and an empty value is not the same as unset: the conserver then tries to open a file with no name. |
| `LOGGING_CONFIG_FILE` | Delete the line, or set `common/logging_dev.conf` for readable logs. The shipped value, `server/logging_dev.conf`, is not a path in the repository. |
| `COMPOSE_PROFILES` | `postgres` starts a local Postgres, `elasticsearch` and `langfuse` start those services, and an empty value starts Redis only. Redis, the API and the conserver always start. |

`REDIS_URL=redis://redis` is already correct for the compose network. [Configuring the Conserver](configuring-the-conserver.md) lists every variable.

## Write a minimal chain

Replace the contents of `config.yml` with one link and no storage:

```yaml
links:
  tag_demo:
    module: links.tag
    options:
      tags:
        - "source:quickstart"

chains:
  demo:
    ingress_lists: [demo_in]
    links: [tag_demo]
    egress_lists: [demo_out]
```

The conserver re-reads this file between vCons, so you can change it without a restart. A chain, its lists and the rules for what a link returns are defined on [Concepts](concepts.md).

## Start the stack

```bash
docker compose up -d --build
docker compose ps
curl http://localhost:8000/api/health
```

The services are `redis`, `api` and `conserver`. The health call returns `{"status": "healthy", ...}` with the build version. To run more workers, raise `CONSERVER_WORKERS` in `.env`, or scale the container with `docker compose up -d --scale conserver=4`.

## Send a vCon

Save this as `demo.json`. The lawful basis attachment is test data for this walkthrough. Replace it with the real basis for your conversations before you send real ones.

```json
{
  "vcon": "0.4.0",
  "uuid": "0192f4f0-7a52-8c3d-9a1b-2c3d4e5f6a7b",
  "created_at": "2026-01-02T12:00:00Z",
  "extensions": ["lawful_basis"],
  "parties": [
    { "name": "Alice", "mailto": "alice@example.com" },
    { "name": "Bob", "mailto": "bob@example.com" }
  ],
  "dialog": [
    {
      "type": "text",
      "start": "2026-01-02T12:00:00Z",
      "parties": [0, 1],
      "originator": 0,
      "mediatype": "text/plain",
      "encoding": "none",
      "body": "Hello Bob."
    }
  ],
  "attachments": [
    {
      "purpose": "lawful_basis",
      "start": "2026-01-02T12:00:00Z",
      "party": 0,
      "dialog": 0,
      "encoding": "json",
      "body": {
        "lawful_basis": "legitimate_interests",
        "expiration": null,
        "purpose_grants": [
          { "purpose": "analysis", "granted": true, "granted_at": "2026-01-02T12:00:00Z" }
        ],
        "proof_mechanisms": [
          {
            "proof_type": "external_system",
            "timestamp": "2026-01-02T12:00:00Z",
            "proof_data": { "note": "quick-start test data" }
          }
        ]
      }
    }
  ]
}
```

Post it to the `demo_in` list:

```bash
curl -X POST "http://localhost:8000/api/vcon?ingress_lists=demo_in" \
  -H "x-conserver-api-token: $CONSERVER_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d @demo.json
```

A worker pops the UUID within a second or two. Pop it from the egress list, then fetch the vCon:

```bash
curl -H "x-conserver-api-token: $CONSERVER_API_TOKEN" \
  "http://localhost:8000/api/vcon/egress?egress_list=demo_out"

curl -H "x-conserver-api-token: $CONSERVER_API_TOKEN" \
  "http://localhost:8000/api/vcon/0192f4f0-7a52-8c3d-9a1b-2c3d4e5f6a7b"
```

The vCon now carries a second attachment with `purpose: "tags"` and the body `["source:quickstart"]`. Because the chain names no `storages`, the only copy is the working copy in Redis, which expires after `VCON_REDIS_EXPIRY` seconds (default 3600). Add a storage to the chain to keep it. See [Storage](storage.md).

## Read the logs

```bash
docker compose logs -f conserver api
```

Logs are JSON by default. A line such as `Completed processing vCon ... Chain: demo` confirms the run. If the vCon did not move, start with [Troubleshooting](troubleshooting.md).
