---
description: How the MCP server handles the appended→amended and must_support→critical field renames.
---

# 🔁 Field-Name Migration

The vCon working group renamed two fields in `draft-ietf-vcon-vcon-core-02`:

| Old name (pre-spec-02) | New name (spec-02 / current) |
|------------------------|-------------------------------|
| `appended` | `amended` |
| `must_support` | `critical` (often surfaced as `must_understand[]`) |

The vCon MCP server has shipped a migration that handles both names at the storage layer, so existing data and clients keep working while new code uses the spec-correct names. This page documents how that works in case you're debugging it or migrating a downstream consumer.

## What the migration does

Database migration `20251120150100_field_renames.sql`:

1. Renames the underlying columns: `appended` → `amended`, `must_support` → `critical`.
2. Creates a `vcons_legacy` view that re-exposes the old names for any reader that hasn't migrated.
3. Updates the indexes accordingly.

The TypeScript schema handlers in `src/tools/handlers/schema.ts` (around lines 34–44) translate between the old and new names on the way in and out. The tools themselves always operate on the new names.

## What this means for you

### If you read from the MCP server

Your client receives responses using the **new** field names: `amended`, `critical`. If your downstream code still expects `appended` / `must_support`, either:

- Update your code to use the new names (preferred), or
- Point your reader at the `vcons_legacy` view, which still uses the old names.

### If you write to the MCP server

The server accepts either name on input — for now. Submitting `appended` or `must_support` produces a deprecation warning in the response. New code should write the spec-correct names.

### If you query the database directly

Use the new column names on the canonical tables. Use the `vcons_legacy` view if you have queries you don't want to rewrite yet.

## Other field-name pitfalls

The rename is only the tip of the field-name issue. Some other names commonly get wrong:

- **Analysis field is `schema`**, never `schema_version`. The `vcon` Python library pre-0.9.1 sometimes emitted `schema_version`; the MCP server treats that as `schema` on read.
- **Attachment field is `purpose`**, never `type` — except the `lawful_basis` extension, which is the documented exception. The server flags any other attachment with a `type` field as suspicious.

## See also

- [Tool Reference](tool-reference.md) — the full surface
- [Contract Tools](contract-tools.md) — the May 2026 design that bakes the new names into the contract
- [vCon Library Quickstart](../vcon-library/quickstart.md) — the same migration as it applies to Python code
