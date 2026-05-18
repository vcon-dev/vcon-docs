---
description: Convert Claude AI conversation exports into vCons — the canonical adapter for LLM chat sessions.
---

# 🤖 vCon Anthropic Chats

**Repo:** [vcon-dev/vcon-anthropic-chats](https://github.com/vcon-dev/vcon-anthropic-chats) · **First release:** May 2026

A standalone adapter that takes Claude AI conversation exports — the JSON files you get from the Claude web app's export feature, or from the API — and converts them into spec-compliant vCons.

## When to use it

- You want to put your team's Claude conversations into the same store as your call recordings, emails, and chats.
- You're building a corpus of LLM interactions for audit, training, or compliance.
- You need to apply [Lawful Basis](../extensions/lawful-basis.md) consent to AI conversations the same way you do to human ones.

## What the output looks like

The adapter uses the [Agent Session extension](../extensions/agent-session.md), so an exported Claude conversation produces:

- A `parties[]` array with the human user and one or more agent parties (each agent gets `role: "agent"` and a `meta.agent_session` block identifying the model and provider).
- A `dialog[]` array with each message as a text dialog entry.
- An `analysis[]` entry of type `agent_trace` containing the full session trace (tool calls, tool results, reasoning) as a JSON-encoded VAC document.
- Optional `attachments[]` for files generated or modified during the session (purpose: `agent_file_change`, `agent_artifact`, etc.).
- An `extensions: ["agent_session"]` declaration.

## Install and usage

See the repo README for the current CLI. The typical invocation:

```bash
vcon-anthropic-chats < claude-export.json > conversation.vcon.json
```

For batch processing, the same module is usable as a Python import.

## See also

- [Agent Session extension](../extensions/agent-session.md) — the spec the adapter produces against
- [vCon Adapter Development Guide](../vcon-adapters/vcon-adapter-development-guide.md) — patterns for building your own adapters
