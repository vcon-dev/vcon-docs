---
description: >-
  Every tool, resource and prompt the vCon MCP server exposes, with the key
  parameters of each, so you can pick the right call without reading the source.
---

# 🧰 Tool Reference

The server exposes 46 tools in seven groups, 19 resources and 10 prompts. This page is built from
each tool's `inputSchema` in [vcon-dev/vcon-mcp](https://github.com/vcon-dev/vcon-mcp) at release
1.9.2. For the response shape of a contract or search tool, call `describe_response_shape` against
the live server; see [Contract Tools](contract-tools.md).

Which tools a client sees depends on the deployment. `MCP_TOOLS_PROFILE`, the category variables
and read-only keys filter the list; see
[Transport and Deployment](transport-and-deployment.md#which-tools-a-client-sees). The category in
brackets after each group heading is the one those filters use.

## vCon CRUD (17 tools)

Create, read, update and delete a vCon and its parts. Category `write`, except `get_vcon` (`read`).

| Tool | Key parameters | What it does |
| ---- | -------------- | ------------ |
| `create_vcon` | `parties` (required), `subject`, `dialog`, `analysis`, `attachments`, `extensions`, `critical` | Validates and stores a new vCon. `must_support` is accepted as a deprecated alias for `critical` |
| `create_vcon_from_template` | `template_name` (required: `phone_call`, `chat_conversation`, `email_thread`, `video_meeting`, `custom`), `parties` (required, non-empty), `subject`, `metadata` | Builds a vCon from a named template and stores it |
| `get_vcon` | `uuid` (required), `response_format` (`full` default, `summary`, `metadata`) | Returns one vCon. `summary` keeps only analyses of type `summary`. LLM clients should prefer `vcon_fetch` |
| `update_vcon` | `uuid`, `updates` (both required), `return_updated` (default true) | Updates top-level `subject`, `extensions` and `critical` only. Use the child tools for everything else |
| `delete_vcon` | `uuid` (required) | Deletes the vCon and its parties, dialog, analysis and attachments. No confirmation parameter; it cannot be undone |
| `add_dialog` | `vcon_uuid`, `dialog` (both required; `dialog.type` required) | Appends a dialog: `recording`, `text`, `transfer` or `incomplete` |
| `add_analysis` | `vcon_uuid`, `analysis` (both required; `analysis.type` and `analysis.vendor` required) | Appends an analysis |
| `add_attachment` | `vcon_uuid`, `attachment` (both required) | Appends an attachment. Set `purpose`; `type` is a legacy field kept for old data |
| `update_dialog` | `vcon_uuid`, `index`, `dialog` | Replaces the dialog at `index`. Omitted fields are cleared |
| `remove_dialog` | `vcon_uuid`, `index` | Strips the dialog to a placeholder that keeps `type`, so later indexes do not shift |
| `update_analysis` | `vcon_uuid`, `index`, `analysis` | Replaces the analysis at `index`. `vendor` required |
| `remove_analysis` | `vcon_uuid`, `index` | Deletes the analysis and renumbers the rest |
| `update_attachment` | `vcon_uuid`, `index`, `attachment` | Replaces the attachment at `index` |
| `remove_attachment` | `vcon_uuid`, `index` | Deletes the attachment and renumbers the rest. Removing the `tags` attachment clears the vCon's tags |
| `add_party` | `vcon_uuid`, `party` | Appends a party and returns its index |
| `update_party` | `vcon_uuid`, `index`, `party` | Replaces the party at `index` |
| `remove_party` | `vcon_uuid`, `index`, `anonymize` (default false) | Leaves an empty placeholder party, or `{name: "anonymous"}` with `anonymize`, so dialog and attachment references do not shift |

Every write is validated before it reaches the database: at least one party, and each party with
an identifier; a valid dialog type; `disposition` on an `incomplete` dialog; `vendor` on every
analysis; encodings of `base64url`, `json` or `none`; either `body` and `encoding` or `url` and
`content_hash`; ISO 8601 dates. An analysis that uses `schema_version` instead of `schema` is
rejected.

## Contract and discovery (7 tools)

The surface built for LLM clients: one envelope, cursor pagination, a byte budget. Category `read`.
Design and envelopes are on [Contract Tools](contract-tools.md).

| Tool | Key parameters | What it does |
| ---- | -------------- | ------------ |
| `vcon_capabilities` | none | Supported tools, include groups, search modes, limits, byte budgets and the legacy-to-contract tool mapping |
| `vcon_taxonomy` | none | Fixed guidance on tag and attachment conventions for one dealer-call dataset, plus a coverage snapshot. Disabled under the `public` profile |
| `vcon_graph_shape` | none | Analysis types, attachment purposes and tag keys actually present, with counts and co-occurrence edges. Same payload as resource `vcon://v1/graph/shape` |
| `describe_response_shape` | `tool_name`, `include_example` (default true) | JSON Schema and an example payload for a contract tool or a legacy read tool. No `tool_name` lists the tools it can describe |
| `vcon_fetch` | `id` (required), `include`, `max_response_bytes` (default 250000, min 1024) | One vCon in `{ok, item}` with only the include groups you ask for |
| `vcon_search` | `mode` (`metadata` default, `keyword`, `semantic`, `hybrid`), `query`, `embedding`, `tags`, `filters`, `include`, `limit` (default 25, max 100), `cursor`, `max_response_bytes`, `threshold` (0.7), `semantic_weight` (0.6) | All four search modes behind one `{ok, items, page}` envelope |
| `vcon_aggregate` | `group_by` (only `dealer`), `tags`, `filters.start_date`, `filters.end_date`, `having.min_count` (default 1), `limit` (default 20, max 500) | Per-dealer `filtered_count` and `baseline_count`, so a client can compute a rate in one call. Needs the `aggregate_vcons_by_dealer_stats` RPC. Disabled under the `public` profile |

## Search, legacy (4 tools)

The original search tools. Still supported. New clients should use `vcon_search`. Category `read`.

| Tool | Key parameters | What it does |
| ---- | -------------- | ------------ |
| `search_vcons` | `subject`, `party_name`, `party_email`, `party_tel`, `start_date`, `end_date`, `tags`, `limit` (default 10, max 1000), `response_format` (`metadata` default, `full`, `ids_only`), `include_count` | Metadata filters only |
| `search_vcons_content` | `query` (required), dates, `tags`, `limit` (default 50), `response_format` (`snippets` default, `full`, `metadata`, `ids_only`), `include_count` | Keyword search over subject, parties, dialog and analysis |
| `search_vcons_semantic` | `query` or `embedding` (384 numbers), `tags`, `threshold` (default 0.7), `limit` (default 50), `response_format`, `include_count` | Similarity search over stored embeddings |
| `search_vcons_hybrid` | `query` (required), `embedding`, `tags`, `semantic_weight` (default 0.6), `limit` (default 50), `response_format`, `include_count` | Keyword and semantic scores blended by `semantic_weight` |

Keyword search is Postgres full text search (`tsvector` and `plainto_tsquery`). It stems, so
"refunds" matches "refund", and it does not correct misspellings. Semantic search embeds the query
with the provider set by `EMBEDDING_PROVIDER` (see
[Transport and Deployment](transport-and-deployment.md#semantic-search)) and compares it with the
384-dimension vectors in `vcon_embeddings`. A vCon with no embedding yet cannot be found this way.

## Tags (5 tools)

A vCon's tags live in one attachment with `purpose: "tags"`, `encoding: "json"` and a body that is
an array of `"key:value"` strings, for example `["department:sales", "priority:high"]`. The
materialized view `vcon_tags_mv` turns them into rows so tag filters are cheap. There is no tool
that tags many vCons in one call.

| Tool | Key parameters | What it does | Category |
| ---- | -------------- | ------------ | -------- |
| `manage_tag` | `vcon_uuid`, `action` (`set` or `remove`), `key` (all required), `value` (string, number or boolean; stored as a string) | Sets or removes one tag on one vCon | `write` |
| `remove_all_tags` | `vcon_uuid` | Clears every tag on one vCon | `write` |
| `get_tags` | `vcon_uuid` (required), `key`, `default_value` | All tags on a vCon, or one tag's value | `read` |
| `get_unique_tags` | `include_counts`, `key_filter`, `min_count` (default 1) | Distinct tag keys and values across the store | `read` |
| `search_by_tags` | `tags` (required), `limit` (default 50), `return_full_vcons`, `max_full_vcons` (default 20) | UUIDs of vCons that carry every given tag, optionally the vCons themselves | `read` |

## Analytics (6 tools)

Reports over the whole store. Category `analytics`.

| Tool | Key parameters | What it does |
| ---- | -------------- | ------------ |
| `get_database_analytics` | `include_growth_trends`, `include_content_analytics`, `include_attachment_stats`, `include_tag_analytics`, `include_health_metrics`, `months_back` (default 12) | One combined report |
| `get_monthly_growth_analytics` | `months_back` (default 12), `include_projections`, `granularity` (`monthly` default, `weekly`, `daily`) | Ingestion over time |
| `get_attachment_analytics` | `include_size_distribution`, `include_type_breakdown`, `include_temporal_patterns`, `top_n_types` (default 10) | Attachment types, media types and sizes |
| `get_tag_analytics` | `include_frequency_analysis`, `include_value_distribution`, `include_temporal_trends`, `top_n_keys` (default 20), `min_usage_count` | Tag key and value frequencies |
| `get_content_analytics` | `start_date`, `end_date`, `include_dialog_analysis`, `include_analysis_breakdown`, `include_party_patterns`, `include_conversation_metrics`, `include_temporal_content` | Dialog types, analysis types and vendors, party patterns, counts of vCons with no dialog or no analysis |
| `get_database_health_metrics` | `include_performance_metrics`, `include_storage_efficiency`, `include_index_health`, `include_connection_metrics`, `include_recommendations` | Performance, storage, index and cache indicators with recommendations |

## Database inspection (5 tools)

For operators. Category `infra`.

| Tool | Key parameters | What it does |
| ---- | -------------- | ------------ |
| `get_database_shape` | `include_counts`, `include_sizes`, `include_indexes` (all default true), `include_columns` (default false) | Tables, row counts, sizes and indexes |
| `get_database_stats` | `include_query_stats`, `include_index_usage`, `include_cache_stats`, `table_name` | Table access, index usage and cache hit ratios |
| `get_database_size_info` | `include_recommendations` (default true) | Store size with suggested query limits |
| `get_smart_search_limits` | `query_type` (`basic`, `content`, `semantic`, `hybrid`, `analytics`), `estimated_result_size` | A suggested result limit for a planned query |
| `analyze_query` | `query` (required, SQL starting with `SELECT`), `analyze_mode` (`explain` default, `explain_analyze`) | Postgres plan for the caller's SQL. `explain_analyze` runs the query |

## Schema and examples (2 tools)

Category `schema`.

| Tool | Key parameters | What it does |
| ---- | -------------- | ------------ |
| `get_schema` | `format` (`json_schema` default, `typescript`), `version` (default `latest`) | The vCon schema the server validates against |
| `get_examples` | `example_type` (required: `minimal`, `phone_call`, `chat`, `email`, `video`, `full_featured`), `format` (`json` default, `yaml`) | An example vCon |

## Resources

Read-only data at `vcon://v1/` URIs. Resource reads run no plugin hooks.

| URI | Returns |
| --- | ------- |
| `vcon://v1/vcons/recent`, `.../recent/{n}` | The newest vCons in full, 10 by default, at most 100 |
| `vcon://v1/vcons/recent/ids`, `.../recent/ids/{n}` | UUID, timestamp and subject of the newest vCons |
| `vcon://v1/vcons/ids`, `.../ids/{n}`, `.../ids/{n}/after/{timestamp}` | Every vCon ID, paged by timestamp, 100 by default, at most 1000 |
| `vcon://v1/vcons/{uuid}` | One vCon |
| `vcon://v1/vcons/{uuid}/metadata`, `/parties`, `/dialog`, `/analysis`, `/attachments` | One part of a vCon |
| `vcon://v1/vcons/{uuid}/attachments/purpose/{purpose}` | Attachments with that purpose |
| `vcon://v1/vcons/{uuid}/attachments/type/{type}` | Attachments with that legacy type |
| `vcon://v1/vcons/{uuid}/analysis/type/{type}` | Analyses of that type |
| `vcon://v1/vcons/{uuid}/transcript`, `/summary`, `/tags` | The transcript analysis, the summary analysis, the parsed tags |
| `vcon://v1/discovery/attachments/purposes`, `.../attachments/types`, `.../analysis/types` | Distinct attachment purposes, legacy attachment types and analysis types in the store |
| `vcon://v1/graph/shape` | The same shape graph `vcon_graph_shape` returns |

## Prompts

Prompts are query templates a client can offer its user. Each returns guidance on which tools to
call with which arguments. They are filtered by the same profile as the tools.

| Prompt | Arguments |
| ------ | --------- |
| `find_by_exact_tags` | `tag_criteria`, `date_range` |
| `find_by_semantic_search` | `search_description`, `date_range` |
| `find_by_keywords` | `keywords`, `filters` |
| `find_recent_by_topic` | `topic`, `timeframe` |
| `find_by_party` | `party_identifier`, `date_range` |
| `discover_available_tags` | `tag_category` |
| `complex_search` | `search_criteria` |
| `find_similar_conversations` | `reference`, `limit` |
| `daily_activity_report` | `date`, `focus_areas` |
| `help_me_search` | `what_you_want` |

## Which search tool

| Situation | Use |
| --------- | --- |
| LLM client that needs stable envelopes and pagination | `vcon_search` |
| A literal word or phrase | `vcon_search` mode `keyword` |
| A concept, phrased however | `vcon_search` mode `semantic` |
| Unsure which | `vcon_search` mode `hybrid` |
| Dates, subject, participant | `vcon_search` mode `metadata` |
| One known UUID | `vcon_fetch` |
| Tags only | `vcon_search` with `tags`, or `search_by_tags` |
| A count or rate per dealer | `vcon_aggregate` |
