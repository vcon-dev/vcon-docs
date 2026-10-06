---
description: >-
  Shows how to keep an AI agent's internal session (tool calls, results,
  reasoning, file changes) in the same vCon as the conversation it served,
  so one signed record answers what the agent did.
---

# 🤖 Agent Session Extension

**Draft:** [`draft-howe-vcon-agent-session-00`](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/) · **Extension token:** `agent_session`

## Purpose of the extension

When an AI agent takes part in a conversation, two records matter: what was said, and what the agent did internally to produce it. vCon already carries the first. [Verifiable Agent Conversations](https://datatracker.ietf.org/doc/draft-birkholz-verifiable-agent-conversations/) (VAC, `draft-birkholz-verifiable-agent-conversations`) defines a record for the second: the agent's prompts, tool calls, tool results, reasoning and events. The Agent Session extension puts a VAC record inside the vCon, so both share one set of parties, one lawful basis and one signature.

It uses existing vCon structures:

* the agent is a party with `role: "agent"`;
* the user-facing turns are ordinary `dialog[]` entries;
* the internal trace is an `analysis[]` entry with `type: "agent_trace"`;
* files and artifacts the agent produced are `attachments[]` entries.

The extension is Compatible (draft Section 3). vCons using it SHOULD list `agent_session` in `extensions`, and SHOULD also list it in `critical` when consumers must process the trace, for example when the only record of an authorizing tool call is in it.

## Agent party

Section 4. Each distinct agent MUST be its own party with `role: "agent"`. An optional `meta.agent_session` object identifies it:

```json
{
  "name": "Claude Opus 4.6",
  "role": "agent",
  "validation": "system",
  "meta": {
    "agent_session": {
      "model_id": "claude-opus-4-6",
      "provider": "anthropic",
      "recording_agent": "claude-code/1.2.0",
      "environment": {
        "cwd": "/Users/example/project",
        "vcs_branch": "main",
        "vcs_commit": "abc123def456"
      }
    }
  }
}
```

`model_id` and `provider` are required inside `agent_session`; `recording_agent` is recommended; `environment` is optional. `role` and `meta` are not core-04 party parameters. `role` is defined by the contact center extension; `meta` is used by this draft without a registration.

## Trace in analysis

Section 6.1. One entry per session is recommended for archival; one per tool call is allowed when you expect to redact single entries.

```json
{
  "type": "agent_trace",
  "dialog": [0, 1],
  "vendor": "anthropic",
  "product": "claude-opus-4-6",
  "schema": "https://datatracker.ietf.org/doc/draft-birkholz-verifiable-agent-conversations/",
  "mediatype": "application/json",
  "encoding": "json",
  "body": {
    "version": "1.0",
    "id": "3f1c9a52-6a0e-4c1b-9d4e-2b7f0c8e1a55",
    "session": {
      "session-id": "session-001",
      "agent-meta": { "model-id": "claude-opus-4-6", "model-provider": "anthropic" },
      "entries": [
        { "type": "user", "id": "e1", "content": "How many open tickets are there?" },
        { "type": "tool-call", "id": "e2", "name": "list_tickets",
          "input": { "status": "open" }, "call-id": "c1" },
        { "type": "tool-result", "id": "e3", "call-id": "c1", "output": { "count": 4 }, "status": "success" },
        { "type": "assistant", "id": "e4", "parent-id": "e1", "content": "There are 4 open tickets." }
      ]
    }
  }
}
```

The draft requires `type: "agent_trace"`, `dialog` as an array of dialog indexes, `vendor` (the model provider), `product` (the model ID), `schema` set to the VAC specification URL, and `encoding: "json"`. The body is a VAC `verifiable-agent-record`: `version`, `id`, and `session`, whose `entries[]` hold message, tool-call, tool-result, reasoning and event entries with their `parent-id` and `children` links kept intact. Member names in the body use VAC's hyphenated spelling.

The draft's own example shows the body as a JSON string. Core-04 lets `encoding: "json"` carry the object directly, which is what this site recommends. For CBOR-encoded VAC records the draft allows `encoding: "base64url"` with `?encoding=cbor` on the `schema` URL (Section 6.3).

## Agent artifacts in attachments

Section 7. Each file or artifact the agent changed SHOULD be an attachment. `party` MUST be the agent's index; `dialog` SHOULD be the turn whose tool call made the change.

```json
{
  "purpose": "agent_file_change",
  "start": "2026-05-18T14:03:12Z",
  "party": 1,
  "dialog": 5,
  "mediatype": "application/json",
  "encoding": "json",
  "body": {
    "path": "src/foo.py",
    "contributor": "agent",
    "line_range": [10, 25],
    "operation": "edit",
    "commit": "abc123",
    "content_hash": "sha512-..."
  }
}
```

Purpose values registered by the draft (Sections 7.1 and 8):

| `purpose` | Contents |
| --- | --- |
| `agent_file_change` | A source file the agent modified |
| `agent_artifact` | Another artifact, such as a database write, API payload or generated document |
| `agent_environment` | A snapshot of the agent's environment |
| `scitt_receipt` | The SCITT receipt for an independently registered VAC record |
| `agent_trace_cose_sign1` | The COSE_Sign1 envelope of that VAC record |

The draft's attachment example omits `start` and `mediatype`; they are added above because core-04 requires `start` on every attachment and `mediatype` for inline content.

## Lawful basis

Section 9. An agent session that processes personal data MUST be governed by a documented lawful basis. Implementations SHOULD include a [lawful basis](lawful-basis.md) attachment that names the data subject by party index and grants at least `agent_session_recording` and `agent_session_analysis`, plus `agent_session_redistribution` where it applies.

## Implementations

* [`vcon-vac-adapter`](https://github.com/vcon-dev/vcon-vac-adapter) converts Claude Code sessions, Anthropic Messages API, OpenAI Responses and OpenAI Agents SDK transcripts into vCons with the `agent_session` extension and a VAC record in `analysis[]`.
* [`vcon-vac`](https://github.com/vcon-dev/vcon-vac) is early scaffolding for VAC CDDL definitions and examples.
* [`vcon-anthropic-chats`](https://github.com/VCONIC/vcon-anthropic-chats) ([tool page](../tools/vcon-anthropic-chats.md)) converts Claude Code sessions and claude.ai exports into vCons. Its current release keeps tool calls and reasoning as its own attachments and analysis entries and does not emit the `agent_session` extension.

## See also

* [Field reference](../vcons/field-reference.md)
* [Lifecycle](lifecycle.md) for recording what happened to the vCon after the session.
