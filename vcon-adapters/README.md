---
icon: puzzle-piece-simple
description: Patterns and existing adapters that pull conversation data into vCons from other systems.
---

# 🧩 vCon Adapters

An *adapter* in vCon-land is anything that takes conversation data from a foreign system — a phone PBX, an email server, a chat platform, a SIPREC stream, an LLM transcript — and produces a vCon. This section covers how to build adapters and which ones already exist.

## Reading

- [vCon Adapter Development Guide](vcon-adapter-development-guide.md) — patterns, schemas, and worked examples for building your own adapter.
- [LLM Guide: Creating vCon Adapters](llm-guide-creating-vcon-adapters.md) — drop into a model's context window when you want it to generate adapter code.

## Available adapters

The [Tools](../tools/README.md) section has the per-adapter detail. Quick index:

- [vCon Anthropic Chats](../tools/vcon-anthropic-chats.md) — Claude AI exports → vCon (uses the [Agent Session extension](../extensions/agent-session.md))
- [vCon SIPREC Adapter](../tools/vcon-siprec-adapter.md) — SIPREC streams → vCon (uses the [SIP Signaling extension](../extensions/sip-signaling.md))
- [vCon Faker](../tools/vcon-faker.md) — synthetic conversation generator (not strictly an adapter, but the same shape — produces vCons from a foreign input)

Other adapters in the broader ecosystem include `vcon-twilio-adapter`, `vcon-eleven-labs-adapter`, `vcon-audio-adapter`, `vcon-telephony-adapters`, and `matrix_vcon_emitter`. These are listed on the [vcon-dev GitHub org](https://github.com/vcon-dev) and are at varying levels of maturity.
