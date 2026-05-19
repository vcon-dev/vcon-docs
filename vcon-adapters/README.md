---
icon: puzzle-piece-simple
description: Patterns, templates, and operational guidance for building services that turn foreign conversation data into vCons.
---

# 🧩 vCon Adapters

An **adapter** is anything that takes conversation data out of a foreign system — a phone PBX, a softswitch, a contact-center suite, an LLM transcript, a chat platform, a SIPREC stream — and produces a [vCon](../vcons/README.md) on the other side. Adapters are how the rest of the vCon ecosystem (conservers, MCP servers, analytics, archives) gets fed.

This section is the playbook for building, deploying, and operating one.

## Spec target

Adapters in this section target IETF [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) with the `vcon` syntax parameter set to `"0.4.0"`. If you see code or examples elsewhere referring to `0.2.0` or `0.3.0`, treat it as out of date.

## The canonical flow

Every adapter — webhook receiver, polling job, file watcher, batch CLI — boils down to the same four stages:

```
   ┌───────────┐    ┌──────────────┐    ┌────────────────┐    ┌──────────────────┐
   │ Source    │ →  │ Build vCon   │ →  │ Sign / store   │ →  │ Deliver          │
   │ event     │    │ (lib helpers)│    │ (optional JWS) │    │ (HMAC webhook)   │
   └───────────┘    └──────────────┘    └────────────────┘    └──────────────────┘
```

The work that's actually adapter-specific is the leftmost box: knowing *your* source platform's events, IDs, timestamps, and recording URLs. Everything to the right of that — vCon construction, signing, retries, delivery — is solved. **Don't write it from scratch.**

## Start here: use the template

The canonical scaffold lives at **[vcon-dev/vcon-adapter-template](https://github.com/vcon-dev/vcon-adapter-template)**. It's a GitHub template repo: click "Use this template" or run `gh repo create --template vcon-dev/vcon-adapter-template …`. You get:

- A `vcon_builder.py` thin wrapper over the official `vcon` Python library — spec-correct by construction
- HMAC-SHA256-signed async webhook delivery with `Idempotency-Key`, exponential-backoff retries, and a dead-letter queue
- `/healthz` and Prometheus `/metrics` endpoints out of the box
- YAML config with `${ENV_VAR}` substitution
- 14 spec-compliance smoke tests that fail loudly if you drift from the spec
- Dockerfile + `docker-compose.yml` + GitHub Actions CI

→ [Quick Start From Template](quick-start-from-template.md) is a one-page recipe to get a new adapter scaffolded in under five minutes.

## Reading order

| Page | When to read it |
|------|-----------------|
| [Quick Start From Template](quick-start-from-template.md) | First adapter, or every new adapter. Five-minute scaffold. |
| [Operational Patterns](operational-patterns.md) | Production deployment. Delivery, signing, retries, DLQ, health, metrics. |
| [Spec Compliance Checklist](spec-compliance-checklist.md) | Code review and PR gate. Print and pin to wall. |
| [Extensions Cookbook](extensions-cookbook.md) | Adding transcripts, consent records, SIP signaling, agent sessions. |
| [vCon Adapter Development Guide](vcon-adapter-development-guide.md) | Going beyond the template — custom architectures, polling vs. webhook listeners, batch CLI, multi-source. |
| [LLM Guide: Creating vCon Adapters](llm-guide-creating-vcon-adapters.md) | Drop into a model's context window when you want it to generate adapter code. |

## Existing adapters in the ecosystem

These live in their own repos under the [vcon-dev GitHub org](https://github.com/vcon-dev) at varying levels of maturity. Several were written before the canonical template existed, so they're useful as architecture references but **not** as compliance references — when their patterns disagree with the [checklist](spec-compliance-checklist.md), trust the checklist.

| Repo | Source | Pattern |
|------|--------|---------|
| [`signalwire_adapter`](https://github.com/vcon-dev/signalwire_adapter) | SignalWire telephony | Polling job |
| [`vcon-eleven-labs-adapter`](https://github.com/vcon-dev/vcon-eleven-labs-adapter) | ElevenLabs voice AI | Polling + CLI batch |
| [`vcon-audio-adapter`](https://github.com/vcon-dev/vcon-audio-adapter) | Audio files on disk | Directory watcher |
| [`sippy-conserver-adapter`](https://github.com/vcon-dev/sippy-conserver-adapter) | Sippy softswitch (S3) | S3 bucket monitor |
| [`ietf2vcon`](https://github.com/vcon-dev/ietf2vcon) | IETF meeting recordings | Batch CLI per-meeting |
| [`matrix_vcon_emitter`](https://github.com/vcon-dev/matrix_vcon_emitter) | Matrix chat | Event stream |

For per-adapter documentation pages (`vCon Faker`, `vCon Anthropic Chats`, `vCon SIPREC Adapter`, etc.), see the [Tools](../tools/README.md) section.

## Related

- [vCon Library (Python)](../vcon-library/README.md) — the official Python library every adapter should use
- [vCon-JS Library](../vcon-js-library/README.md) — TypeScript equivalent for Node-based adapters
- [Extensions](../extensions/README.md) — WTF transcription, lawful basis, SIP signaling, agent session, lifecycle
- [Conserver](../conserver/README.md) — the typical downstream consumer of adapter output
