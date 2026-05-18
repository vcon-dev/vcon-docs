---
description: The May 2026 discovery and contract surface — how LLM clients introspect the server before making expensive calls.
---

# Contract Tools

In May 2026 the vCon MCP server added a family of six tools designed specifically for LLM clients that don't know what they're talking to in advance. They let a client ask "what do you support?", "what shape will you return?", and "what's your controlled vocabulary?" before making queries that might fail or return junk.

These are the **contract tools**: `vcon_fetch`, `vcon_search`, `vcon_capabilities`, `vcon_taxonomy`, `vcon_graph_shape`, and `describe_response_shape`. Together they give an LLM enough information to write accurate, token-budget-aware queries against any deployment of the server, regardless of its specific extensions, tag vocabulary, or attached domain conventions.

## Why they exist

The pre-2026 MCP surface (and most MCP servers in general) assumes the client knows the server's shape ahead of time. That assumption breaks when an LLM is the client:

- The model doesn't always know which extensions a particular server supports.
- The model can't reliably guess how a server normalizes things like dealer IDs or campaign names.
- Token-budget-aware clients need to know the response shape *before* receiving the response, or they'll overflow context windows.
- LLMs are bad at offset-based pagination — they need cursor-based.

The contract tools fix this by exposing the server's contract explicitly. A well-behaved LLM client calls one or more of them once per session and caches the result.

## Stable response envelopes

All contract tools use one of two envelope shapes — predictable across tools, predictable across versions:

**Single-item (fetch):**

```json
{
  "ok": true,
  "item": { ... }
}
```

**Multi-item (search, list):**

```json
{
  "ok": true,
  "items": [ ... ],
  "page": {
    "next_cursor": "opaque-string-or-null"
  }
}
```

**Error:**

```json
{
  "ok": false,
  "error": {
    "code": "RESPONSE_TOO_LARGE",
    "message": "Response would exceed max_response_bytes",
    "details": { ... }
  }
}
```

If a client encounters `ok: false`, it can recover from a clean structured error. The previous design returned partial / malformed results in the same shape as success, which an LLM had no way to detect.

## The six tools

### `vcon_capabilities`

Returns what this server supports:

- `supported_includes` — the field groups `vcon_fetch` and `vcon_search` understand: `core`, `parties`, `summary`, `tags`, `dealer`, `counts`, `dialog`, `analysis`, `attachments`
- `search_modes` — `metadata`, `keyword`, `semantic`, `hybrid`
- `pagination_semantics` — confirms cursor-based pagination, names the cursor field
- `byte_budgets` — the default and max values of `max_response_bytes`
- `migration_hints` — notes about any active field-name migrations (see [Field-Name Migration](field-name-migration.md))

Call this first. Cache the result for the session.

### `vcon_taxonomy`

Returns the controlled vocabulary the database actually uses:

- `portal_taxonomy` — domain-specific enum values surfaced by this deployment (e.g. a portal with categories `complaint`, `inquiry`, `praise`)
- `common_tag_keys` — the tag keys that appear in the corpus, with sample values
- `attachment_types` — recognized attachment purposes, including extension-defined ones like `strolid_dealer`
- `preferred_fields` — hints about which fields the deployment expects to be populated

LLMs use this to write better filters. Without it, the model would guess values; with it, the model knows the real surface area.

### `vcon_graph_shape`

Returns a live picture of how data is shaped in **this** corpus — not what's theoretically possible, but what's actually present:

```json
{
  "ok": true,
  "item": {
    "nodes": [
      { "id": "analysis:transcript", "type": "analysis_type", "count": 12483 },
      { "id": "tag:priority", "type": "tag_key", "count": 821 },
      { "id": "attachment:strolid_dealer", "type": "attachment_purpose", "count": 12200 }
    ],
    "edges": [
      { "source": "analysis:transcript", "target": "tag:priority", "strength": 0.42 }
    ]
  }
}
```

The shape evolves with the deployment. Use it when an LLM needs to ground its assumptions about analysis types and tag keys in what's really there.

### `describe_response_shape`

Given a tool name, returns the JSON Schema (or equivalent) of the response that tool will produce, plus a concrete example payload. Use this when:

- Your client needs to parse the response into typed objects.
- You're building a multi-step plan and need to know which fields will be available downstream.
- You're operating under a token budget and need to limit what to ask for.

Calling without a `tool_name` returns the list of tools with published shapes.

### `vcon_fetch`

The contract-aware fetch. Like `get_vcon` but:

- Accepts an `include` array so you only pay for the parts you need. Valid values are the same `supported_includes` returned by `vcon_capabilities`.
- Accepts a `max_response_bytes` budget (default 250 000). If the response would exceed the budget, the call returns `{ok: false, error: {code: "RESPONSE_TOO_LARGE"}}` rather than truncating silently.
- Returns the response in the `{ok, item}` envelope so the schema matches `describe_response_shape("vcon_fetch")`.

Prefer this over `get_vcon` from any LLM-driven workflow.

### `vcon_search`

The contract-aware search. Designed for LLM use:

- **Modes:** `metadata`, `keyword`, `semantic`, `hybrid`. Hybrid takes a `semantic_weight` (0–1) blending the two scores.
- **Cursor-based.** Returns `page.next_cursor`; the client passes the cursor back to get the next page. No offsets, no page numbers.
- **Dealer filterable.** In multi-tenant deployments where Strolid-style dealer attachments are present, `filters.dealer_id` (exact match) or `filters.dealer_name` (substring) are first-class filters rather than free-form tags. Backed by the `aggregate_vcons_by_dealer_stats` RPC for fast top-of-funnel counts.
- **Tag-first.** Tag filters are the cheapest dimension to filter on; the search pushes them first.
- **LLM-hardened.** Rejects ambiguous queries with a structured error rather than returning bad results — the model can recover from a clean error, but can't recover from a wrong answer it doesn't realize is wrong.

Same `include` and `max_response_bytes` controls as `vcon_fetch`. Same `{ok, items, page}` envelope.

## Recommended flow for an LLM session

```
1. vcon_capabilities()                         -> cache server contract
2. vcon_taxonomy()                             -> cache controlled vocab
3. vcon_graph_shape()                          -> understand what's actually there
4. describe_response_shape("vcon_search")      -> understand search response shape
5. vcon_search(filters, include, max_bytes)    -> paginated list of UUIDs + summaries
6. vcon_fetch(uuid, include, max_bytes)        -> pull the specific data you need
7. repeat 5/6
```

Steps 1–4 happen once per session and are cheap. Steps 5–6 are the working loop.

## Field-name compatibility

In May 2026 the server also shipped database migration `20251120150100_field_renames.sql` that handles the `appended → amended` and `must_support → critical` renames at the storage layer (see [Field-Name Migration](field-name-migration.md)). The contract tools always emit and accept the spec-correct names; the legacy names are translated on read via the `vcons_legacy` view.

## See also

- [Tool Reference](tool-reference.md) — the full tool list
- [Field-Name Migration](field-name-migration.md) — back-compat behavior
- [Transport and Deployment](transport-and-deployment.md) — how to actually run a server you can call these tools on
- [What the vCon MCP Server Can Do](what-the-vcon-mcp-server-can-do.md) — narrative overview
