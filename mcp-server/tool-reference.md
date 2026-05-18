---
description: Every tool the vCon MCP server exposes, grouped by purpose. Quick lookup for LLM-driven workflows.
---

# Tool Reference

The vCon MCP server exposes **37 tools** to LLM clients. They split into seven groups by purpose. For each tool's authoritative response schema, call `describe_response_shape(tool_name)` against the live server — see [Contract Tools](contract-tools.md).

The May 2026 contract redesign added a six-tool family (`vcon_fetch`, `vcon_search`, `vcon_capabilities`, `vcon_taxonomy`, `vcon_graph_shape`, `describe_response_shape`) that LLM clients should prefer over the legacy equivalents wherever they overlap.

## vCon CRUD (8 tools)

Create, fetch, update, and delete vCons; append the three sub-resources (dialog, analysis, attachment).

| Tool | Purpose |
|------|---------|
| `create_vcon` | Create a new vCon from a JSON object. |
| `create_vcon_from_template` | Create a vCon from one of the server's named templates (`phone_call`, `chat_conversation`, `email_thread`, `video_meeting`, `custom`). |
| `get_vcon` | Fetch a vCon by UUID. **For LLM workflows, prefer `vcon_fetch`** — it returns a stable envelope and lets you control byte budget. |
| `update_vcon` | Partially update an existing vCon. Accepts both spec-correct (`amended`, `critical`) and legacy (`appended`, `must_support`) field names — see [Field-Name Migration](field-name-migration.md). |
| `delete_vcon` | Delete a vCon by UUID. Cascades to all related rows (parties, dialog, analysis, attachments). |
| `add_dialog` | Append a dialog entry to an existing vCon. |
| `add_analysis` | Append an analysis entry. `vendor` is REQUIRED. |
| `add_attachment` | Append an attachment. Uses `purpose` (or `type` for the lawful_basis extension exception). |

## Search — legacy (4 tools)

The pre-2026 search tools. Still supported. For new LLM-driven workflows prefer `vcon_search` from the [contract surface](contract-tools.md).

| Tool | Purpose |
|------|---------|
| `search_vcons` | Filter by subject, party, dates, and tags. Metadata-only. |
| `search_vcons_content` | Postgres full-text search over dialog and analysis bodies. |
| `search_vcons_semantic` | Vector / embedding similarity (pgvector, 384-dim, OpenAI by default). Use when the query is conceptual rather than literal. |
| `search_vcons_hybrid` | Combines full-text and semantic with a tunable `semantic_weight`. |

## Contract / discovery (6 tools, new May 2026)

The LLM-facing surface. Stable envelopes, cursor pagination, byte-budget enforcement, dealer filtering. See [Contract Tools](contract-tools.md) for the design rationale and envelope formats.

| Tool | Purpose |
|------|---------|
| `vcon_fetch` | Fetch a vCon (or a partial projection of one) with explicit `include` selection and `max_response_bytes`. Returns `{ok, item}`. |
| `vcon_search` | Unified search across `metadata`, `keyword`, `semantic`, and `hybrid` modes. Cursor-based, dealer-filterable, LLM-hardened. Returns `{ok, items, page}`. |
| `vcon_capabilities` | Returns what the server supports — tools, includes, search modes, byte budgets, migration hints. Call once per session and cache. |
| `vcon_taxonomy` | Controlled vocabulary the corpus actually uses (portal taxonomy, common tag keys, attachment types, preferred fields). |
| `vcon_graph_shape` | Live shape of the corpus — analysis types, attachment purposes, tag keys, and their co-occurrence edges. |
| `describe_response_shape` | JSON Schema + example payload for any tool. Use before parsing or planning a multi-step query. |

## Tags (5 tools)

Tags are stored as `attachments[]` entries with `purpose: "tags"` and a JSON body. The MCP server materializes them through `vcon_tags_mv` for fast lookup.

| Tool | Purpose |
|------|---------|
| `manage_tag` | Add, update, or remove a single tag. |
| `get_tags` | List tags on a specific vCon. |
| `get_unique_tags` | List all distinct tag keys/values across the database. |
| `search_by_tags` | Find vCons matching tag filters. Fast — backed by the materialized view. |
| `remove_all_tags` | Bulk-clear all tags on a vCon. |

## Analytics (6 tools)

Aggregations over the corpus. Reports rather than queries.

| Tool | Purpose |
|------|---------|
| `get_database_analytics` | High-level summary: size, growth, content distribution, health. |
| `get_monthly_growth_analytics` | Ingestion trend, with `granularity` of `daily`, `weekly`, or `monthly`. |
| `get_attachment_analytics` | Distribution of attachment purposes, mediatypes, and sizes. |
| `get_tag_analytics` | Tag frequency and value distribution. |
| `get_content_analytics` | Dialog mediatypes, party patterns, analysis vendors. |
| `get_database_health_metrics` | Query performance, index usage, optimization hints. |

## Database inspection (6 tools)

For ops, debugging, and capacity planning. Read-only.

| Tool | Purpose |
|------|---------|
| `get_database_shape` | Tables, columns, indexes, relationships, sizes. |
| `get_database_stats` | Cache hit rates, index usage, slow-query stats. |
| `get_database_size_info` | Row counts and storage totals, with recommendations for large datasets. |
| `get_smart_search_limits` | The result-size and depth bounds the server will apply. Useful before constructing a large search. |
| `analyze_query` | Returns a Postgres `EXPLAIN` plan for the SQL behind a given tool call. |
| `vcon_aggregate` | Server-side rollup grouped by dealer. Returns `filtered_count` and `baseline_count` per group for rate calculation, backed by the `aggregate_vcons_by_dealer_stats` RPC. |

## Schema & examples (2 tools)

| Tool | Purpose |
|------|---------|
| `get_schema` | Return the vCon JSON Schema (or TypeScript types) in the format the client requests. |
| `get_examples` | A handful of example vCons covering common patterns — `minimal`, `phone_call`, `chat`, `email`, `video`, `full_featured`. |

## When to use which search tool

The five search-shaped tools overlap. Quick guide:

| Situation | Use |
|-----------|-----|
| LLM-driven workflow, want stable envelopes and pagination | `vcon_search` |
| Literal keyword in conversation content | `vcon_search` mode=`keyword` (or legacy `search_vcons_content`) |
| Conceptual / fuzzy query | `vcon_search` mode=`semantic` (or legacy `search_vcons_semantic`) |
| Mixed, want best general result | `vcon_search` mode=`hybrid` (or legacy `search_vcons_hybrid`) |
| Just need metadata or a UUID lookup | `vcon_search` mode=`metadata` or `vcon_fetch` |
| Tag-only filtering, no full-text | `search_by_tags` |

## See also

- [Contract Tools](contract-tools.md) — design and envelope formats for the May 2026 family
- [Field-Name Migration](field-name-migration.md) — how the server handles `appended → amended` and `must_support → critical`
- [Transport and Deployment](transport-and-deployment.md) — auth, transport modes, Docker / npm
- [What the vCon MCP Server Can Do](what-the-vcon-mcp-server-can-do.md) — narrative overview
