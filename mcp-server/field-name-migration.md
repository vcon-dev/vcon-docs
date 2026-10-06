---
description: How the vCon MCP server handles the renames of appended to amended and must_support to critical, so you know which names to send and what you will get back.
---

# 🔁 Field-Name Migration

The vCon core draft renamed two top-level fields. The server stores and returns the new names.

| Old name | Current name |
| -------- | ------------ |
| `appended` | `amended` |
| `must_support` | `critical` |

## What the server returns

Every read returns `critical` and `amended`. A row written before the rename that has a value only
in the old `appended` column comes back as `amended`.

## What the server accepts

* `create_vcon` takes `critical`. It also takes `must_support` as a deprecated alias and stores it
  as `critical` when `critical` is absent. It does this silently, with no warning in the response.
* `update_vcon` applies `subject`, `extensions` and `critical`. Its published schema still lists
  `must_support`, but the handler ignores that key, so send `critical`.
* `create_vcon` has no `amended` parameter.

## What the database holds

Migration `20251120150100_field_renames.sql` does not rename columns. The `critical` and `amended`
columns are the live ones; the old `must_support` and `appended` columns remain, marked deprecated.
The migration adds:

* `vcons_legacy`, a read-only view that exposes `critical as must_support` and
  `amended as appended` alongside the new names, for SQL readers that have not moved.
* `get_vcon_critical(uuid)` and `get_vcon_amended(uuid)`, which prefer the new column and fall
  back to the old one.

The server itself does not read `vcons_legacy`. Point old SQL at it if you cannot change the query
yet, and move to the new column names on `vcons`.

Migration `20260925000000_preserve_extra_fields.sql` adds an `extra` column to each table. Keys
that have no column of their own, such as party `meta` or an extension's own fields, are kept
there and merged back on read, so a stored vCon reads back with the keys it was written with.

## Related field names

* **Analysis `schema`.** An analysis that uses `schema_version` instead of `schema` fails
  validation and is rejected.
* **Attachment `purpose`.** Send `purpose` on every attachment, including `lawful_basis` and
  `tags`. The `type` field is still accepted for old data. A database trigger mirrors `tags` across
  both columns so tag queries see either form.

## See also

* [Tool Reference](tool-reference.md)
* [Contract Tools](contract-tools.md)
* [vCon Library Quickstart](../vcon-library/quickstart.md), the same renames in Python
