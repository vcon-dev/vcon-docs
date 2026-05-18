---
description: Every tool the vCon MCP server exposes, grouped by purpose. Quick lookup for LLM-driven workflows.
---

# 🧰 Tool Reference

The vCon MCP server exposes **37 tools** to LLM clients. The full set is grouped by purpose below. For the tool's exact JSON Schema, run `describe_response_shape` against the live server — that's the authoritative source.

For the design philosophy behind the May-2026 contract redesign (`vcon_fetch`, `vcon_search`, `vcon_capabilities`, `vcon_taxonomy`, `describe_response_shape`), see [Contract Tools](contract-tools.md).

## vCon CRUD

| Tool | Purpose |
|------|---------|
| `create_vcon` | Create a new vCon from a JSON object. |
| `create_vcon_from_template` | Create a vCon from one of the server's named templates (text-chat, call-recording, etc.). |
| `get_vcon` | Fetch a vCon by UUID. **For LLM workflows, prefer `vcon_fetch`** — it returns a paginated, capability-aware response. |
| `update_vcon` | Partially update an existing vCon. |
| `delete_vcon` | Delete a vCon by UUID. Logs a `vcon_deleted` lifecycle event if the [Lifecycle extension](../extensions/lifecycle.md) is configured. |

## vCon parts

| Tool | Purpose |
|------|---------|
| `add_dialog` | Append a dialog entry to an existing vCon. |
| `add_analysis` | Append an analysis entry. `vendor` is required. |
| `add_attachment` | Append an attachment. Uses `purpose` (or `type` for the lawful_basis extension exception). |

## Tags

| Tool | Purpose |
|------|---------|
| `manage_tag` | Add, update, or remove a tag on a vCon. |
| `get_tags` | List tags on a specific vCon. |
| `get_unique_tags` | List all distinct tag keys/values across the database. |
| `get_tag_analytics` | Frequency and co-occurrence stats for tags. |
| `remove_all_tags` | Bulk-clear tags on a vCon. |
| `search_by_tags` | Find vCons matching tag filters. Fast — backed by a materialized view. |

## Search

| Tool | Purpose |
|------|---------|
| `search_vcons` | Lightweight metadata search (UUID, parties, dates). |
| `search_vcons_content` | Full-text search across dialog and analysis bodies. |
| `search_vcons_semantic` | Vector / embedding search. Use when the query is conceptual rather than literal. |
| `search_vcons_hybrid` | Combines full-text and semantic; usually the right default for LLM clients. |
| `vcon_search` | **(New, May 2026)** Tag-first paginated search with optional dealer filtering. The hardened LLM-facing endpoint. See [Contract Tools](contract-tools.md). |
| `analyze_query` | Pre-flight a query to see how the server would interpret and route it. |
| `get_smart_search_limits` | Returns the result-size and depth bounds the server will apply. |

## Contract / discovery (May 2026 redesign)

These tools let an LLM client introspect the server and write more accurate calls. See [Contract Tools](contract-tools.md) for the design rationale.

| Tool | Purpose |
|------|---------|
| `vcon_fetch` | Fetch a vCon (or a partial projection) with response-shape negotiation. |
| `vcon_capabilities` | List what this server supports — tools, extensions, dialect quirks. |
| `vcon_taxonomy` | Returns the controlled vocabulary the server uses for dialog types, analysis types, attachment purposes, tag keys, etc. |
| `describe_response_shape` | Schema for the response a given tool will return. Use before parsing. |
| `vcon_graph_shape` | High-level graph of how vCons in this database relate (groups, amendments, redactions). |
| `get_database_shape` | Lower-level: tables, columns, indexes. |
| `get_examples` | A handful of example vCons covering common patterns. |
| `get_schema` | Returns the vCon JSON schema in your preferred format (JSON Schema, TypeScript types). |

## Analytics

| Tool | Purpose |
|------|---------|
| `vcon_aggregate` | Group-and-count over arbitrary fields (dealer, region, party.role, …). Uses the `vcon_aggregate_by_dealer_stats` RPC for known surfaces. |
| `get_database_analytics` | Top-level usage stats. |
| `get_database_health_metrics` | Latency, error rates, queue depth. |
| `get_database_size_info` | Row counts, storage, index sizes. |
| `get_database_stats` | Snapshot of common stats. |
| `get_monthly_growth_analytics` | vCons ingested per month, with trend. |
| `get_content_analytics` | What's actually inside vCons — dialog mediatypes, analysis vendors, etc. |
| `get_attachment_analytics` | Distribution of attachment purposes and sizes. |

## When to use which search tool

The four search tools overlap. Quick guide:

- **Literal keyword in conversation content?** → `search_vcons_content`
- **Conceptual / fuzzy query?** → `search_vcons_semantic`
- **Mixed, want best general result?** → `search_vcons_hybrid`
- **Tag-filtered, paginated, LLM-driven?** → `vcon_search` (the May 2026 hardened endpoint)
- **Just need metadata or a UUID lookup?** → `search_vcons` or `vcon_fetch`

## See also

- [Contract Tools](contract-tools.md) — design and usage of the May 2026 discovery surfaces
- [Field-Name Migration](field-name-migration.md) — how the server handles `appended` → `amended` and `must_support` → `critical`
- [What the vCon MCP Server Can Do](what-the-vcon-mcp-server-can-do.md) — narrative overview
