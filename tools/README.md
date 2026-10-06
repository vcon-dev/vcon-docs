---
icon: screwdriver-wrench
description: Standalone tools and adapters that produce, consume, or operate on vCons.
---

# 🛠️ Tools

The vCon ecosystem is more than the spec and the libraries. It's a set of practical tools you can run today. This section indexes them.

## Generators and converters

Tools that produce vCons.

- [vCon Faker](vcon-faker.md): synthetic vCons from LLM-generated dialog + TTS audio
- [vCon Anthropic Chats](vcon-anthropic-chats.md): converts Claude Code sessions and claude.ai exports into vCons
- [vCon SIPREC Adapter](vcon-siprec-adapter.md): records SIPREC sessions as vCons
- [vCon MCP Adapters](vcon-mcp-adapters.md): converts AI agent traces (Anthropic, OpenAI, Claude Code, Helicone, Langfuse, LangSmith) into vCons

## Administration & operations

Tools for managing vCons at scale.

- [vCon Admin](vcon-admin.md): Streamlit toolkit for importing, exporting and inspecting vCons
- [Redis to Mongo Sync](mongo-redis-sync.md): archived one-way copy from Redis to MongoDB

## Datasets

- [Public vCon Datasets](vcon-datasets.md): public GitHub datasets of vCons, and how to load one with vcon-data

## Apps and stores

- [vCon Apps and Stores](../vcon-apps-and-stores/README.md): what vCon stores and apps are, the example app, and the TADHack dataset

## Adding a tool

If you've built something that produces, consumes, or operates on vCons and you'd like it indexed here, open a pull request against [vcon-dev/vcon-docs](https://github.com/vcon-dev/vcon-docs) adding a new page under `tools/`. Keep entries to: what it does in one sentence, when you'd use it, install / link, and one minimal usage example.
