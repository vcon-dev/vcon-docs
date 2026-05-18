---
description: How to run the vCon MCP server — transport modes, authentication, environment variables, Docker and npm.
---

# Transport and Deployment

The vCon MCP server runs in one of two transport modes and authenticates clients via a configurable bearer-token scheme. This page is the reference for both.

## Transport modes

The server speaks the [Model Context Protocol](https://modelcontextprotocol.io) over **stdio** by default and **Streamable HTTP** when configured. Pick the transport based on how the client connects:

| Transport | When to use | Configured via |
|-----------|-------------|----------------|
| **stdio** (default) | Claude Desktop, the MCP inspector CLI, any client that launches the server as a subprocess | `MCP_TRANSPORT=stdio` (or unset) |
| **HTTP** (Streamable HTTP, spec 2025-03-26) | Web clients, remote LLM agents, anything that connects over the network | `MCP_TRANSPORT=http` |

### stdio

The default. The server reads JSON-RPC from stdin and writes responses to stdout. The host process (Claude Desktop, an MCP-aware CLI, a test harness) is responsible for spawning the server. Nothing to configure besides credentials and the database connection.

### HTTP (Streamable HTTP)

Streamable HTTP is the 2025-03-26 MCP HTTP transport. Configurable surface:

| Variable | Description | Default |
|----------|-------------|---------|
| `MCP_HTTP_HOST` | Bind address | `127.0.0.1` |
| `MCP_HTTP_PORT` | Bind port | `3000` |
| `MCP_HTTP_STATELESS` | If `true`, every request is independent — no `Mcp-Session-Id` header tracking | `false` |
| `MCP_HTTP_JSON_ONLY` | If `true`, disable Server-Sent Events; responses are plain JSON | `false` |
| `MCP_HTTP_CORS` | Enable CORS | `false` |
| `MCP_HTTP_CORS_ORIGIN` | Allowed origin when CORS is enabled | (unset) |
| `MCP_HTTP_DNS_PROTECTION` | Enable DNS rebinding protection | `false` |

In stateful mode (the default for HTTP), the server returns an `Mcp-Session-Id` header on the first response. Clients include it on every subsequent request. In stateless mode, each request stands alone — useful behind load balancers or for serverless deployments where you can't pin requests to a single instance.

## Authentication

Authentication is **bearer-token based** and **optional**. Configure via environment variables.

| Variable | Description | Default |
|----------|-------------|---------|
| `API_KEYS` | Comma-separated list of valid bearer tokens | (unset) |
| `API_KEY_HEADER` | Header name to read | `authorization` |
| `API_AUTH_REQUIRED` | If `false`, auth checks are skipped entirely | `true` |

By default the server reads `Authorization: Bearer <token>`. Override with `API_KEY_HEADER=x-mcp-token` (or similar) if your environment requires a different header.

If `API_KEYS` is unset and `API_AUTH_REQUIRED=true`, the server **refuses to start** — that's intentional, to prevent accidental unauthenticated deployments. If you genuinely want an open server (local development), set `API_AUTH_REQUIRED=false`.

Per-tool scoping is **not currently enforced** — every valid token can call every tool. If you need per-tool ACLs, run a separate server instance with a different token set.

### Multi-tenancy

For multi-tenant deployments, supply a tenant identifier per request:

| Variable | Description |
|----------|-------------|
| `TENANT_ISOLATION` | When `true`, enables tenant filtering on all queries |
| `TENANT_ID` | Default tenant id used when no per-request tenant is supplied |
| `x-tenant-id` (header) | Per-request tenant override |

Tenant filtering is enforced by Postgres Row Level Security on the Supabase backend; the MCP server passes the resolved tenant id into every query. Make sure your RLS policies are configured before you turn this on.

## Database connection

The server expects a Supabase Postgres deployment, configured by:

| Variable | Description | Required? |
|----------|-------------|-----------|
| `SUPABASE_URL` | Project URL (e.g. `https://abc123.supabase.co`) | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Service-role key — bypasses RLS for trusted server work | One of these |
| `SUPABASE_ANON_KEY` | Anonymous key — relies on RLS for access control | One of these |
| `DB_TYPE` | `supabase` (default) or `mongodb` (experimental) | No |

The database schema is managed by the migrations under `supabase/migrations/` in the [vcon-mcp repo](https://github.com/vcon-dev/vcon-mcp); apply them with `supabase db push` or the Supabase migration UI before pointing the server at a fresh project.

For semantic search you'll additionally want to set an embedding provider — currently the server uses OpenAI by default and stores 384-dimension vectors in `vcon_embeddings`. The relevant env vars (`OPENAI_API_KEY`, `EMBEDDING_MODEL`, etc.) are documented in the repo README.

## Caching (optional)

If `REDIS_URL` is set, the server uses Redis as a read-through cache in front of Postgres. Cache hits skip the database call entirely; misses populate the cache after the query. Defaults are sensible for most deployments — see the repo README for the full list of `REDIS_*` env vars and TTL controls.

## Running the server

### Local development

```bash
git clone https://github.com/vcon-dev/vcon-mcp
cd vcon-mcp
npm install
npm run build

# Configure
cp .env.example .env
# edit .env: SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, API_KEYS

# Run
npm run dev
```

The compiled entry point is `dist/index.js`. The package ships a `vcon-mcp` bin entry, so once installed globally (`npm install -g vcon-mcp`) you can also start it with just `vcon-mcp`.

### Docker

A published image lives on AWS Public ECR:

```bash
docker run --rm -p 3000:3000 \
  -e SUPABASE_URL="https://your-project.supabase.co" \
  -e SUPABASE_SERVICE_ROLE_KEY="$SUPABASE_SERVICE_ROLE_KEY" \
  -e API_KEYS="$MCP_API_KEY_1,$MCP_API_KEY_2" \
  -e MCP_TRANSPORT=http \
  public.ecr.aws/r4g1k2s3/vcon-dev/vcon-mcp:main
```

The `main` tag tracks the trunk branch. For production, pin to an explicit semver tag (the CI tags every release as `vX.Y.Z`).

### Claude Desktop

To use the server with Claude Desktop, run it in stdio mode and reference it from your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "vcon": {
      "command": "npx",
      "args": ["-y", "vcon-mcp"],
      "env": {
        "SUPABASE_URL": "https://your-project.supabase.co",
        "SUPABASE_SERVICE_ROLE_KEY": "...",
        "API_AUTH_REQUIRED": "false"
      }
    }
  }
}
```

Claude Desktop launches the server as a subprocess and speaks stdio with it — no HTTP, no exposed ports.

## Observability

The server emits OpenTelemetry spans and metrics for every tool call. The [`vcon-mcp-adapters`](../tools/vcon-mcp-adapters.md) package ships ready-made OTLP exporters; alternatively configure your own collector via the standard `OTEL_EXPORTER_OTLP_ENDPOINT` env var.

Per-tool span attributes include:

- `tool_name`, `tool_category`
- `status` (`ok` | `error` | `validation_failed`)
- `duration_ms`
- `response_bytes`, `truncated` (for byte-budgeted tools)
- `cache_hit` (when Redis caching is enabled)

## See also

- [Tool Reference](tool-reference.md) — every tool, grouped by purpose
- [Contract Tools](contract-tools.md) — design and envelopes for the May 2026 LLM-facing surface
- [How the vCon MCP Server is Built](how-the-vcon-mcp-server-is-built.md) — internal architecture
- [Field-Name Migration](field-name-migration.md) — back-compat behavior
