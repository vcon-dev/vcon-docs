---
description: Captures an AI agent's session trace — prompts, tool calls, results, artifacts — alongside the human-facing conversation.
---

# 🤖 Agent Session Extension

**Draft:** [`draft-howe-vcon-agent-session`](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/) · **Extension name:** `"agent_session"`

## What it is

A growing number of "conversations" involve an AI agent on at least one side. The human-facing surface of that interaction — the words exchanged, the time, the parties — fits cleanly into a normal vCon. The *agent's* internal session — the prompts it received, the tools it called, the responses those tools returned, the files it touched, the reasoning it produced — does not. That data is just as important for audit, debugging, training, and compliance, but it sits in a different shape.

The Agent Session extension brings that data into the vCon as a peer to the human dialog. It uses vCon's existing primitives:

- The agent is represented as a **party** with `role: "agent"` and structured metadata describing the model.
- The session trace lives in **analysis** as a Verifiable Agent Conversations (VAC) document.
- Agent-produced artifacts and environment snapshots live in **attachments**.

That means agent sessions inherit everything vCon already has — consent, signing, redaction, lifecycle, SCITT.

## When to use it

- Documenting LLM-driven contact-center deflection: the LLM's full prompt-and-tool-call trace is auditable alongside the call recording.
- Coding-agent sessions (Claude Code, Cursor, etc.) where the conversation, the tool invocations, and the file edits are all part of the same record.
- AI-assisted workflows where the agent's output needs to be reproducible: the trace tells you exactly what the agent did and why.
- Bridging vCon and [VAC (Verifiable Agent Conversations)](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/) — the two specs are designed to compose.

## Spec surface

Agent Session adds data in three places.

### 1. Party metadata

Each agent participant is a normal party, distinguished by `role: "agent"` and a structured `meta.agent_session` block:

```json
{
  "parties": [
    {
      "name": "Customer",
      "role": "customer"
    },
    {
      "name": "Claude",
      "role": "agent",
      "meta": {
        "agent_session": {
          "model_id": "claude-opus-4-7",
          "provider": "anthropic",
          "recording_agent": "claude-code/1.5.0",
          "environment": "production"
        }
      }
    }
  ]
}
```

### 2. Session trace in `analysis[]`

The full VAC session trace is a JSON document stored as an analysis entry with `type: "agent_trace"`:

```json
{
  "analysis": [
    {
      "type": "agent_trace",
      "dialog": 0,
      "vendor": "anthropic",
      "product": "claude-opus-4-7",
      "encoding": "json",
      "schema": "https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/",
      "body": "{\"version\":\"1.0\",\"session_trace\":{\"messages\":[...],\"tool_calls\":[...]}}"
    }
  ]
}
```

The `body` is the VAC document, encoded as a JSON string. `vendor`, `product`, and `schema` identify the agent and the trace format.

### 3. Artifacts in `attachments[]`

Files the agent produced, environment snapshots, and tool-call payloads go in `attachments[]` with one of these purposes:

| Purpose | Meaning |
|---------|---------|
| `agent_file_change` | A file the agent created, modified, or deleted |
| `agent_artifact` | A standalone artifact the agent produced (e.g. a generated document) |
| `agent_environment` | A snapshot of the agent's runtime environment at the time of the session |

```json
{
  "attachments": [
    {
      "purpose": "agent_file_change",
      "party": 1,
      "dialog": 0,
      "filename": "src/payment_handler.py",
      "mediatype": "text/x-python",
      "encoding": "none",
      "body": "def process_payment(amount, ...): ..."
    }
  ]
}
```

Declare the extension at the top level:

```json
{
  "vcon": "0.4.0",
  "extensions": ["agent_session"]
}
```

## Relationship to other extensions

- **Lawful basis.** Agent-session data is personal data when the agent participated in a conversation with a real person. Use the [Lawful Basis extension](lawful-basis.md) the same way you would for the human-only case.
- **Lifecycle.** Agent traces benefit especially from a [Lifecycle](lifecycle.md) ledger — being able to prove what the agent saw and did, when, is the entire compliance story for AI-assisted workflows.
- **WTF.** If the agent session also produced spoken output (TTS), that recording is normal vCon dialog, optionally transcribed via [WTF](wtf-transcription.md).

## See also

- The `vcon-anthropic-chats` adapter (see [Tools](../tools/README.md)) converts Claude AI conversation exports into vCons using this extension.
- The `vcon-vac` project is the reference implementation tying VAC traces into vCons.
