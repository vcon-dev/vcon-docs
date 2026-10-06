---
icon: puzzle-piece-simple
description: Patterns, templates, and operational guidance for building services that turn foreign conversation data into vCons.
---

# 🧩 vCon Adapters

An **adapter** is anything that takes conversation data out of a foreign system, a phone PBX, a softswitch, a contact-center suite, an LLM transcript, a chat platform, a SIPREC stream, and produces a [vCon](../vcons/README.md) on the other side. Adapters are how the rest of the vCon ecosystem (conservers, MCP servers, analytics, archives) gets fed.

This section is the playbook for building, deploying, and operating one.

## Spec target

Adapters in this section target `draft-ietf-vcon-vcon-core-04`, syntax `"0.4.0"`. The [Spec Compliance Checklist](spec-compliance-checklist.md) holds the full target, with the JSON-body rule and every field rule. Code that cites `0.2.0`, `0.3.0`, `-02` or `-03` is out of date.

## The canonical flow

Every adapter (webhook receiver, polling job, file watcher, batch CLI) boils down to the same four stages:

```
   ┌───────────┐    ┌──────────────┐    ┌────────────────┐    ┌──────────────────┐
   │ Source    │ →  │ Build vCon   │ →  │ Lawful basis / │ →  │ Deliver          │
   │ event     │    │ (lib helpers)│    │ finalize       │    │ (webhook or      │
   │           │    │              │    │                │    │  conserver)      │
   └───────────┘    └──────────────┘    └────────────────┘    └──────────────────┘
```

The work that's actually adapter-specific is the leftmost box: knowing *your* source platform's events, IDs, timestamps, and recording URLs. Everything to the right of that, vCon construction, lawful basis, retries, delivery, is solved. Content signing (JWS) is not built into the template; see [Operational Patterns](operational-patterns.md#content-signing-add-it-yourself). **Don't write it from scratch.**

## Start here: use the template

Every adapter in this ecosystem should start from **[vcon-dev/vcon-adapter-template](https://github.com/vcon-dev/vcon-adapter-template)**. It's a GitHub template repo: click "Use this template" or run `gh repo create --template vcon-dev/vcon-adapter-template …`. You get:

- A `vcon_builder.py` thin wrapper over the official `vcon` Python library, with `finalize_vcon()` to close out a vCon and `json_body()` to read an attachment or analysis body back whether it was written as a JSON value or a legacy string
- A lawful-basis builder: `LAWFUL_BASIS*` environment variables or a YAML block build the attachment; leave it unset and the adapter logs a warning and skips the attachment rather than guessing a basis. Never defaulted in code
- A copyable `assert_spec_compliant()` test, run against the vendored JSON schema, so a new adapter inherits the compliance gate
- Two delivery paths: signed webhooks (HMAC-SHA256, `Idempotency-Key`, exponential-backoff retries, dead-letter queue), or conserver-direct delivery (`POST /vcon?ingress_lists=...` with an `x-conserver-api-token` header) for adapters that talk straight to a vcon-server instance
- `/healthz` and Prometheus `/metrics` endpoints out of the box
- YAML config with `${ENV_VAR}` substitution
- Dockerfile, `docker-compose.yml`, and a GitHub Actions workflow that runs lint, type check and tests

→ [Quick Start From Template](quick-start-from-template.md) is a one-page recipe to get a new adapter scaffolded in under five minutes.

## Building a new adapter

Start from the template above, not from an existing adapter repo: several of those predate the template and carry patterns the [checklist](spec-compliance-checklist.md) no longer allows. Configure lawful basis from the first commit (`LAWFUL_BASIS*` env vars or YAML; see the template's `USAGE.md`). The template never defaults a basis; set the one your deployment relies on.

## Reading order

| Page | When to read it |
|------|-----------------|
| [Quick Start From Template](quick-start-from-template.md) | First adapter, or every new adapter. Five-minute scaffold. |
| [Operational Patterns](operational-patterns.md) | Production deployment. Delivery, HMAC signing, retries, DLQ, health, metrics. |
| [Spec Compliance Checklist](spec-compliance-checklist.md) | Code review and PR gate. Print and pin to wall. |
| [Extensions Cookbook](extensions-cookbook.md) | Adding transcripts, consent records, SIP signaling, agent sessions. |
| [vCon Adapter Development Guide](vcon-adapter-development-guide.md) | Going beyond the template, custom architectures, polling vs. webhook listeners, batch CLI, multi-source. |
| [LLM Guide: Creating vCon Adapters](llm-guide-creating-vcon-adapters.md) | Short digest to paste into a model's context when you want it to generate adapter code. |

## Existing adapters in the ecosystem

These live in their own repos under the [vcon-dev GitHub org](https://github.com/vcon-dev).

### Modernized for `-04`

Built on the template, or brought up to its bar: lawful basis, `-04` JSON bodies, and the checklist mostly apply (see the telephony row for the one open gap).

| Repo | Source | Pattern |
|------|--------|---------|
| [`vcon-telephony-adapters`](https://github.com/vcon-dev/vcon-telephony-adapters) | Asterisk, Bandwidth, FreeSWITCH, Telnyx, Twilio | Webhook receivers, one monorepo. Webhook signature validation fails closed by default: an adapter refuses to start without its secret unless `ALLOW_UNSIGNED_WEBHOOKS=true` is set explicitly. Lawful basis on every platform. External media (`MEDIA_BACKEND=filesystem` or `s3`) produces fully schema-valid `-04` output with `url` and `content_hash`; the default `MEDIA_BACKEND=embed` path still writes inline audio as `encoding: "base64"`, not yet `base64url`, pending a follow-up release. Dockerfile and a GHCR image published on version tags |
| [`vcon-audio-adapter`](https://github.com/vcon-dev/vcon-audio-adapter) | Audio files on disk or S3 | Directory watcher. Spec-correct `-04` output, lawful basis, `content_hash` on referenced audio |
| [`vcon-vac-adapter`](https://github.com/vcon-dev/vcon-vac-adapter) | AI-agent sessions (Claude Code, Anthropic, OpenAI, OpenTelemetry) | Converts agent session transcripts into vCons with an `agent_trace` analysis under the [Agent Session](../extensions/agent-session.md) extension. Lawful basis; delivers by webhook or straight to a conserver; a daemon mode watches a configured directory of Claude Code session files |

### Architecture references, not compliance references

Written before the template existed and not modernized this round. Useful for their integration patterns; when they disagree with the [checklist](spec-compliance-checklist.md), trust the checklist.

| Repo | Source | Pattern |
|------|--------|---------|
| [`signalwire_adapter`](https://github.com/vcon-dev/signalwire_adapter) (archived) | SignalWire telephony | Polling job |
| [`vcon-eleven-labs-adapter`](https://github.com/vcon-dev/vcon-eleven-labs-adapter) | ElevenLabs voice AI | Polling + CLI batch |
| [`sippy-conserver-adapter`](https://github.com/vcon-dev/sippy-conserver-adapter) (archived) | Sippy softswitch (S3) | S3 bucket monitor |
| [`ietf2vcon`](https://github.com/vcon-dev/ietf2vcon) | IETF meeting recordings | Batch CLI per-meeting |
| [`matrix_vcon_emitter`](https://github.com/vcon-dev/matrix_vcon_emitter) | Matrix chat | Event stream |

For per-adapter documentation pages (`vCon Faker`, `vCon Anthropic Chats`, `vCon SIPREC Adapter`, `vCon MCP Adapters`, etc.), see the [Tools](../tools/README.md) section.

## Related

- [vCon Library (Python)](../vcon-library/README.md): the official Python library every adapter should use
- [vCon-JS Library](../vcon-js-library/README.md): TypeScript equivalent for Node-based adapters
- [Extensions](../extensions/README.md): WTF transcription, lawful basis, SIP signaling, agent session, lifecycle
- [Conserver](../conserver/README.md): the typical downstream consumer of adapter output
