---
description: The May 2026 discovery and contract surface — how LLM clients introspect the server before making expensive calls.
---

# 📜 Contract Tools

In May 2026 the vCon MCP server added a family of tools designed specifically for LLM clients that don't know what they're talking to in advance. They let a client ask "what do you support?", "what shape will you return?", and "what's your controlled vocabulary?" before making queries that might fail or return junk.

These are the **contract tools**: `vcon_fetch`, `vcon_search`, `vcon_capabilities`, `vcon_taxonomy`, and `describe_response_shape`.

## Why they exist

The earlier MCP surface (and most MCP servers in general) assumes the client knows the server's shape ahead of time. That assumption breaks when an LLM is the client:

- The model doesn't always know which extensions a particular server supports.
- The model can't reliably guess how a server normalizes things like dealer IDs or campaign names.
- Token-budget-aware clients need to know the response shape *before* receiving the response.

The contract tools fix this by exposing the server's contract explicitly. A well-behaved LLM client calls one or more of them once per session and caches the result.

## The five tools

### `vcon_capabilities`

Returns what this server supports:

- Which tools are available (by name)
- Which vCon extensions are supported (`lawful_basis`, `wtf`, `agent_session`, …)
- Server-specific dialect notes (e.g. "this deployment uses Strolid dealer IDs", "tag keys are case-sensitive")
- Pagination model and limits

Call this first. Cache the result for the session.

### `vcon_taxonomy`

Returns the controlled vocabulary the database actually uses:

- Tag keys and their observed values
- Attachment purposes seen in the database
- Analysis types and vendors
- Dialog mediatypes
- Custom fields surfaced by this deployment

LLMs use this to write better filters. Without it, the model would guess values; with it, the model knows the real surface area.

### `describe_response_shape`

Given a tool name, returns the JSON Schema (or equivalent) of the response that tool will produce. Use this when:

- Your client needs to parse the response into typed objects.
- You're building a multi-step plan and need to know which fields will be available downstream.
- You're operating under a token budget and need to limit what to ask for.

### `vcon_fetch`

The contract-aware fetch. Like `get_vcon` but:

- Accepts a `fields` filter so you only pay for the parts you need.
- Returns the response in a shape that matches `describe_response_shape("vcon_fetch")`.
- Negotiates response size against the client's token budget if one is declared.

Prefer this over `get_vcon` from any LLM-driven workflow.

### `vcon_search`

The contract-aware search. Designed for LLM use:

- **Tag-first.** Tag filters are the cheapest dimension to filter on; the tool pushes them first.
- **Paginated.** Returns a cursor; LLMs are bad at paging via offsets, so the cursor is the only supported model.
- **Dealer-filterable.** In multi-tenant deployments, `dealer_id` is a first-class filter rather than a free-form tag.
- **LLM-hardened.** Rejects ambiguous queries with a structured error rather than returning bad results — the model can recover from a clean error, but can't recover from a wrong answer it doesn't realize is wrong.
- Backed by the `vcon_aggregate_by_dealer_stats` RPC for fast top-of-funnel counts before drilling in.

## Recommended flow for an LLM session

```
1. vcon_capabilities()        -> cache server contract
2. vcon_taxonomy()            -> cache controlled vocab
3. describe_response_shape("vcon_search")  -> understand search response shape
4. vcon_search(filters)       -> get a paginated list of UUIDs and summaries
5. vcon_fetch(uuid, fields)   -> pull the specific data you need
6. repeat 4/5
```

Steps 1–3 happen once per session and are cheap. Steps 4–5 are the working loop.

## Field-name compatibility

In May 2026 the server also shipped a database migration that handles the `appended → amended` and `must_support → critical` field renames at the storage layer (see [Field-Name Migration](field-name-migration.md)). The contract tools always emit and accept the spec-correct names; the legacy names are translated on read via a `vcons_legacy` view.

## See also

- [Tool Reference](tool-reference.md) — the full tool list
- [Field-Name Migration](field-name-migration.md) — back-compat behavior
- [What the vCon MCP Server Can Do](what-the-vcon-mcp-server-can-do.md) — narrative overview
