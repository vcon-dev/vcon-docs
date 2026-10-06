---
description: >-
  How to run the vCon MCP server: transports, authentication, tool profiles,
  tenancy, every environment variable, and recipes for npm, Docker and a store
  shared with the conserver.
---

# 🚀 Transport and Deployment

The server runs over stdio by default and over Streamable HTTP when `MCP_TRANSPORT=http`. It needs
a Supabase Postgres project with the repository's migrations applied. Everything else is optional.
Values on this page are read from `src/` at release 1.9.2 of
[vcon-dev/vcon-mcp](https://github.com/vcon-dev/vcon-mcp).

## Transports

| Transport | Use it for | Set |
| --------- | ---------- | --- |
| stdio (default) | A host that launches the server as a subprocess: Claude Desktop, Claude Code, Cursor, the MCP Inspector | `MCP_TRANSPORT=stdio` or unset |
| Streamable HTTP | Remote agents, web clients, shared deployments | `MCP_TRANSPORT=http` |

Over stdio there is no network listener and no authentication; the host process owns access.

Over HTTP the server listens on `MCP_HTTP_HOST:MCP_HTTP_PORT`. By default it is stateful: the first
response carries an `Mcp-Session-Id` header, the client sends it on every later request, and the
session lives in that process's memory. Behind a load balancer without sticky sessions set
`MCP_HTTP_STATELESS=true`, so each request stands alone.

| Variable | Default | Effect |
| -------- | ------- | ------ |
| `MCP_HTTP_HOST` | `127.0.0.1` (`0.0.0.0` in the Docker image) | Bind address |
| `MCP_HTTP_PORT` | `3000` | Bind port |
| `MCP_HTTP_STATELESS` | `false` | No session state; every request independent |
| `MCP_HTTP_JSON_ONLY` | `false` | Plain JSON responses instead of server-sent events |
| `MCP_HTTP_DNS_PROTECTION` | `false` | DNS rebinding protection, checked against the two lists below |
| `MCP_HTTP_ALLOWED_HOSTS` | unset | Comma-separated `Host` values accepted |
| `MCP_HTTP_ALLOWED_ORIGINS` | unset | Comma-separated `Origin` values accepted |

The MCP endpoint answers on any path outside the REST base path, for example
`http://localhost:3000/mcp`.

### REST API

In HTTP mode the same port also serves a REST API under `REST_API_BASE_PATH` (default `/api/v1`):
vCon create, batch create, get, delete and child appends, plus search, tag, analytics, discovery
and database routes. It is on unless `REST_API_ENABLED=false`, uses the same keys as the MCP
endpoint, and sends `Access-Control-Allow-Origin` from `CORS_ORIGIN` (default `*`). `health`,
`version`, `schema` and `examples` need no key. The conserver's `vcon_mcp` storage writes through
this API; see [Sharing a store with the conserver](#sharing-a-store-with-the-conserver).

## Authentication

Authentication applies to HTTP. It is on unless `API_AUTH_REQUIRED=false`.

| Variable | Default | Effect |
| -------- | ------- | ------ |
| `API_KEYS` | unset | Comma-separated keys with every enabled tool |
| `API_KEYS_READONLY` | unset | Comma-separated keys that see no `write` tools and only GET on the REST API |
| `API_ANONYMOUS_READONLY` | `false` | A request with no key gets a read-only session |
| `API_KEY_HEADER` | `authorization` | Header holding the key. With `authorization` the value is `Bearer <key>` |
| `API_AUTH_REQUIRED` | `true` | `false` turns authentication off. Local development only |

If authentication is required and neither key list is set, the server still starts but answers
every MCP request with HTTP 503 and a message saying authentication is not configured. A wrong or
missing key gets 401. A session opened with a read-only key cannot be joined by a read-write key,
or the reverse.

### OAuth

The server can also accept OAuth 2.1 access tokens from an external issuer such as Supabase Auth.
It does not issue tokens. It serves protected-resource metadata (RFC 9728) and verifies each token's
signature against the issuer's JWKS, its issuer, expiry and audience. Static keys are checked first.

| Variable | Effect |
| -------- | ------ |
| `OAUTH_ISSUER` | Issuer URL. Setting it turns OAuth on |
| `OAUTH_RESOURCE` | This server's resource URL, for example `https://mcp.example.com/mcp`. Required with `OAUTH_ISSUER` |
| `OAUTH_READONLY` | Default `true`: OAuth sessions are read-only. `false` gives them write tools |
| `OAUTH_ALLOWED_EMAIL_DOMAINS` | Comma-separated domains allowed in. Empty allows any user the issuer signs in |
| `OAUTH_CONSENT_ANON_KEY`, `OAUTH_CONSENT_PROVIDERS` | Settings for the consent page; providers default to `email` |

## Which tools a client sees

Tools carry one of five categories: `read`, `write`, `schema`, `analytics`, `infra`.
`MCP_TOOLS_PROFILE` picks a preset.

| Profile | Categories | Notes |
| ------- | ---------- | ----- |
| `full` | all five | The default |
| `readonly` | `read`, `schema` | |
| `user` | `read`, `write`, `schema` | |
| `admin` | `read`, `analytics`, `infra`, `schema` | |
| `minimal` | `read`, `write` | |
| `public` | `read`, `schema` | Also disables `vcon_aggregate` and `vcon_taxonomy`. For hosted read-only datasets |

Without a profile, `MCP_ENABLED_CATEGORIES` or `MCP_DISABLED_CATEGORIES` (comma-separated) choose
categories directly. `MCP_DISABLED_TOOLS` removes named tools and works with or without a profile.
Prompts are filtered by the same rules. Read-only keys and OAuth sessions then drop the `write`
category on top of whatever the deployment enabled. `MCP_SERVER_INSTRUCTIONS` sets the text the
server returns to every client on `initialize`; use it to tell an agent what the store holds.

## Database

| Variable | Effect |
| -------- | ------ |
| `SUPABASE_URL` | Project URL. Required |
| `SUPABASE_SERVICE_ROLE_KEY` | Service-role key. Bypasses RLS unless tenancy is on |
| `SUPABASE_ANON_KEY` | Anonymous key, subject to RLS |
| `SUPABASE_DB_SCHEMA` | Postgres schema, default `public`. Also prefixes Redis keys, so instances on different schemas can share one Redis |
| `DB_TYPE` | `supabase` (default) or `mongodb` with `MONGO_URL` and `MONGO_DB_NAME` |

Apply the migrations in `supabase/migrations/` with `supabase db push` before first start.

### Tenancy

One instance serves one tenant. With `RLS_ENABLED=true` and `CURRENT_TENANT_ID` set, the server
sets `app.current_tenant_id` on its database session and the Row Level Security policies restrict
every table to that tenant. On write, the tenant is read from the vCon's attachment whose `purpose`
(or legacy `type`) equals `TENANT_ATTACHMENT_TYPE` (default `tenant`), at the JSON path
`TENANT_JSON_PATH` (default `id`) in its body. There is no per-request tenant header. For several
tenants, run several instances.

### Semantic search

Query embeddings are 384 dimensions, matching the `vcon_embeddings` column.

| Setting | Provider |
| ------- | -------- |
| Default, no `OPENAI_API_KEY` | Supabase's built-in `gte-small` through the `embed-query` edge function. Needs `SUPABASE_SERVICE_ROLE_KEY` |
| `OPENAI_API_KEY` set | OpenAI `text-embedding-3-small`, truncated to 384 dimensions |
| `EMBEDDING_PROVIDER=supabase` or `openai` | Forces one or the other |

Stored vCons are embedded outside the write path by the `embed-vcons` Supabase edge function, which
either fills in text that has no embedding yet (`mode=backfill`, the default) or embeds one vCon
(`mode=embed`). Something has to call it, for example a scheduled job; until it runs, new vCons are
invisible to semantic search. Use the same model for stored and query embeddings, or similarity
scores mean nothing.

### Caching

With `REDIS_URL` set, `get_vcon` and the other single-vCon reads check Redis first, store misses for
`VCON_REDIS_EXPIRY` seconds (default 3600). Creates, updates, deletes and child edits delete the
cached copy. Tag changes through `manage_tag` and `remove_all_tags` do not, so a cached vCon can
show old tags until it expires. Searches always query Postgres.

## Observability

OpenTelemetry is on unless `OTEL_ENABLED=false`, but the default exporter, `console`, discards
traces and metrics. Set `OTEL_EXPORTER_TYPE=otlp` to send them to `OTEL_ENDPOINT` (default
`http://localhost:4318`). `OTEL_SERVICE_NAME` defaults to `vcon-mcp-server`.

Each tool call produces a span with `mcp.tool.name`, `mcp.tool.success` and, on failure,
`mcp.tool.error.type`, plus the metrics `tool.execution.duration` and `tool.execution.count`.
Reads add `cache.hit`; searches add `search.type`, `search.results.count`, `search.threshold` and
`search.semantic_weight`.

## Other variables

| Variable | Effect |
| -------- | ------ |
| `ENV_FILE` | Env file to load, default `.env`. Lets several instances on one host use different files |
| `LOG_LEVEL` | Log level, default `info` in production |
| `MCP_DEBUG`, `RLS_DEBUG` | Extra logging of stdio input and tenant visibility |
| `VCON_PLUGINS_PATH` | Comma-separated plugin modules to load at startup |
| `VCON_LICENSE_KEY`, `VCON_OFFLINE_MODE` | Passed to plugins that need them |
| `VCON_INSTANCE_LABEL` | A label for this instance in logs and the REST `health` route |

## Running it

### From a host over stdio

```json
{
  "mcpServers": {
    "vcon": {
      "command": "npx",
      "args": ["-y", "vcon-mcp"],
      "env": {
        "SUPABASE_URL": "https://your-project.supabase.co",
        "SUPABASE_SERVICE_ROLE_KEY": "..."
      }
    }
  }
}
```

This works in Claude Desktop's `claude_desktop_config.json` and in other hosts that take the same
`mcpServers` shape.

### From source

```bash
git clone https://github.com/vcon-dev/vcon-mcp
cd vcon-mcp
npm install
npm run build
cp .env.example .env   # set SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, API_KEYS
npm run dev
```

### Docker

The image defaults to HTTP on `0.0.0.0:3000`.

```bash
docker run --rm -p 3000:3000 \
  -e SUPABASE_URL="https://your-project.supabase.co" \
  -e SUPABASE_SERVICE_ROLE_KEY="$SUPABASE_SERVICE_ROLE_KEY" \
  -e API_KEYS="$MCP_API_KEY" \
  public.ecr.aws/r4g1k2s3/vcon-dev/vcon-mcp:1.9.2
```

CI pushes `latest` and `main-<sha>` from every commit on `main`, and `1.9.2`, `1.9` and `1` from a
`v1.9.2` release tag. Image tags carry no `v`. Pin a release tag in production.

### Sharing a store with the conserver

The [conserver](../conserver/README.md) hands finished vCons to this server through its `vcon_mcp`
storage, which posts each vCon to the REST API. Both services can also use one Redis: they cache a
vCon as a JSON string at `vcon:<uuid>`, so a read on either side can hit what the other wrote.

Conserver `config.yml` (literal values; the conserver does not expand `${VAR}` in this file):

```yaml
storages:
  vcon_mcp:
    module: storage.vcon_mcp
    options:
      base_url: http://vcon-mcp:3000/api/v1
      api_key: "mcp-key-for-the-conserver"
      timeout: 30
```

Add `vcon_mcp` to the `storages` list of each chain that should feed the server.

MCP server environment:

```bash
MCP_TRANSPORT=http
API_KEYS=mcp-key-for-the-conserver,mcp-key-for-agents
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_ROLE_KEY=...
REDIS_URL=redis://redis:6379   # the conserver's REDIS_URL
VCON_REDIS_EXPIRY=3600
```

Leave `SUPABASE_DB_SCHEMA` unset when sharing Redis with the conserver; setting it prefixes the MCP
server's keys and the two caches stop overlapping.

## See also

* [Tool Reference](tool-reference.md)
* [Contract Tools](contract-tools.md)
* [How the vCon MCP Server is Built](how-the-vcon-mcp-server-is-built.md)
