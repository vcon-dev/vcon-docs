---
icon: screwdriver-wrench
description: Standalone tools and adapters that produce, consume, or operate on vCons.
---

# 🛠️ Tools

The vCon ecosystem is more than the spec and the libraries — it's a set of practical tools you can run today. This section indexes them.

## Generators

Tools that produce vCons.

- [vCon Faker](vcon-faker.md) — synthetic vCons from LLM-generated dialog + TTS audio
- [vCon Anthropic Chats](vcon-anthropic-chats.md) — converts Claude AI conversation exports into vCons
- [vCon SIPREC Adapter](vcon-siprec-adapter.md) — ingests SIPREC-formatted call recordings

## Administration & operations

Tools for managing vCons at scale.

- [vCon Admin](vcon-admin.md) — admin UI for browsing, editing, and exporting vCons
- [Mongo ↔ Redis Sync](mongo-redis-sync.md) — keep a Mongo vCon store and a Redis cache in sync
- [vCon MCP Adapters](vcon-mcp-adapters.md) — observability adapters (OpenTelemetry) for the MCP server

## Apps and stores

- [vCon Apps and Stores](../vcon-apps-and-stores/README.md) — community-built apps and the vCon App Template

## Adding a tool

If you've built something that produces, consumes, or operates on vCons and you'd like it indexed here, open a pull request against [vcon-dev/vcon-docs](https://github.com/vcon-dev/vcon-docs) adding a new page under `tools/`. Keep entries to: what it does in one sentence, when you'd use it, install / link, and one minimal usage example.
