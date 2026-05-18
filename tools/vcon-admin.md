---
description: Admin UI for browsing, editing, and exporting vCons in a conserver-backed deployment.
---

# 🗂️ vCon Admin

**Repo:** [vcon-dev/vcon-admin](https://github.com/vcon-dev/vcon-admin)

`vcon-admin` is a web UI for the conserver and its storage backends. Use it when you need to:

- Browse the vCons in a database without writing SQL or a custom UI.
- Inspect a vCon's full JSON.
- Edit metadata on a vCon (tags, subject, custom fields).
- Export selected vCons to JSON.
- Run ad-hoc searches.

It's a complementary tool to the [conserver](../conserver/README.md) — the conserver runs the data pipeline, vcon-admin gives humans a window into the result.

## Install

See the repo for current install instructions. The typical deployment is a Docker container alongside a conserver instance, pointing at the same storage backend.

## When NOT to use vcon-admin

- For LLM-driven querying, the [vCon MCP Server](../mcp-server/README.md) is the right surface.
- For pipeline orchestration (transcribe → analyze → store), the [conserver](../conserver/README.md) is what you want; vcon-admin is read/edit, not workflow.

## See also

- [Conserver](../conserver/README.md)
- [vCon MCP Server](../mcp-server/README.md) — programmatic / LLM-facing alternative
