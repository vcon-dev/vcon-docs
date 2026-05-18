---
icon: server
description: The vCon MCP Server — an open-source Model Context Protocol server that gives AI assistants standardized access to vCon conversation data.
---

# MCP Server

The vCon MCP Server is the canonical way for AI assistants — Claude, Cursor, custom agents — to read, write, and reason about vCon conversation data. It speaks the [Model Context Protocol](https://modelcontextprotocol.io) over stdio or Streamable HTTP, exposes **37 tools** grouped into seven categories, and is backed by a Supabase Postgres deployment with optional Redis caching and pgvector-based semantic search.

The code lives at [vcon-dev/vcon-mcp](https://github.com/vcon-dev/vcon-mcp). It's MIT-licensed, TypeScript, and ships as both an npm package (`vcon-mcp`) and a Docker image (`public.ecr.aws/r4g1k2s3/vcon-dev/vcon-mcp:main`).

## When to use it

- You want Claude Desktop (or any MCP client) to query a vCon database in natural language.
- You're building an LLM agent that needs durable, structured access to conversation history.
- You have vCons in a [conserver](../conserver/README.md)-managed Supabase backend and want a query layer that's safe to expose to language models.
- You need the May 2026 [contract / discovery surface](contract-tools.md) — stable response envelopes, cursor pagination, byte-budget enforcement — for production LLM workflows.

## Documentation layout

**Get started:**

- [What is the vCon MCP Server?](what-is-the-vcon-mcp-server.md) — overview, requirements, getting started
- [MCP and AI](mcp-and-ai.md) — how MCP and AI assistants connect
- [Business Cases](business-cases-for-mcp-servers-and-vcon.md) — when to reach for this

**Use it:**

- [What the vCon MCP Server Can Do](what-the-vcon-mcp-server-can-do.md) — narrative overview of capabilities
- [Tool Reference](tool-reference.md) — every tool, grouped by purpose
- [Contract Tools](contract-tools.md) — the May 2026 LLM-facing surface (vcon_fetch, vcon_search, etc.)

**Run it:**

- [Transport and Deployment](transport-and-deployment.md) — auth, transport modes, environment variables, Docker, npm, Claude Desktop config
- [Field-Name Migration](field-name-migration.md) — how the server handles legacy `appended` / `must_support` field names

**Understand it:**

- [How the vCon MCP Server is Built](how-the-vcon-mcp-server-is-built.md) — architecture, schema, request flow
- [MCP and Conserver Together](mcp-and-conserver-together.md) — how the MCP server and conserver compose

**External:**

- [GitHub: vcon-dev/vcon-mcp](https://github.com/vcon-dev/vcon-mcp) — source
- [mcp.conserver.io](https://mcp.conserver.io/) — auto-generated technical reference
