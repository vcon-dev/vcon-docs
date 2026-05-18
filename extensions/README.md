---
description: vCon extensions add structured data for specific use cases without breaking the core spec.
---

# 🧩 Extensions

The vCon core specification ([`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/)) keeps the container small on purpose. Anything beyond parties, dialog, analysis, and attachments lives in an extension — a separately documented spec that adds new fields, attachment purposes, analysis types, or external lifecycle behavior.

## How extensions work

Two top-level fields on a vCon coordinate extensions:

- **`extensions[]`** — strings listing every extension this vCon uses. Consumers can use this to decide whether they have enough support to safely process the vCon.
- **`must_understand[]`** — a subset of `extensions[]` that consumers MUST be able to interpret. If a consumer sees a value here that it doesn't understand, it must refuse to process the vCon rather than silently drop data. (In older drafts and library code this field was called `must_support` or `critical` — they mean the same thing.)

A typical signal looks like this:

```json
{
  "vcon": "0.4.0",
  "uuid": "...",
  "extensions": ["lawful_basis", "wtf"],
  "must_understand": ["lawful_basis"],
  "parties": [...],
  "dialog": [...],
  "analysis": [...],
  "attachments": [...]
}
```

## Available extensions

| Extension | Purpose | Where it lives | Draft |
|-----------|---------|----------------|-------|
| [Lawful Basis](lawful-basis.md) | Records the legal grounds for processing conversation data (GDPR / privacy) | `attachments[]` with `type: "lawful_basis"` | [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/) |
| [Lifecycle](lifecycle.md) | SCITT-anchored append-only ledger of vCon lifecycle events (create, send, consent, delete) | External SCITT ledger; metadata-only on the vCon | [`draft-howe-vcon-lifecycle`](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/) |
| [WTF Transcription](wtf-transcription.md) | World Transcription Format — provider-agnostic shape for speech-to-text output | `analysis[]` with `type: "wtf_transcription"` (or `type: "transcript"` + `schema:` pointing at the WTF draft) | [`draft-howe-vcon-wtf-extension`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf/) |
| [Agent Session](agent-session.md) | Captures an AI agent's session trace (prompts, tool calls, artifacts) alongside the human conversation | `parties[].meta.agent_session`, `analysis[].type: "agent_trace"`, `attachments[].purpose: "agent_*"` | [`draft-howe-vcon-agent-session`](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/) |
| [SIP Signaling](sip-signaling.md) | SIP call metadata, STIR/SHAKEN certificates, and signaling traces | `parties[].sip_*`, `dialog[].sip_*`, `attachments[].purpose: "sip-*"` | [`draft-howe-vcon-sip-signaling`](https://datatracker.ietf.org/doc/draft-howe-vcon-sip-signaling/) |

## A note on field naming

The core spec uses `purpose` on attachments (never `type`). The **Lawful Basis** extension is the one documented exception — it uses `type: "lawful_basis"` because the attachment is treated as a typed structural object rather than a free-form payload. Every other extension that touches `attachments[]` uses `purpose`.

If you see code or older docs that put `type:` on an attachment that isn't lawful_basis, that's a legacy pattern from pre-spec-02 libraries; the spec-correct form is `purpose:`.

## Related work (not vCon extensions, but commonly confused with them)

- [`draft-howe-sipcore-mcp-extension`](https://datatracker.ietf.org/doc/draft-howe-sipcore-mcp-extension/) is a SIP protocol extension (SIPCORE WG, not the vCon WG). It defines a SIP option tag, headers, and a media type for carrying MCP payloads inside SIP sessions. It does **not** add fields to a vCon — but it does come up in conversations about MCP-enhanced telephony, so it's worth knowing where it lives.

## When to define a new extension

Don't add fields to a vCon outside of an extension; consumers won't know what to do with them and may reject the vCon. If you have data that doesn't fit one of the extensions above:

1. Check whether the data fits in `analysis[]` (anything derived from the conversation) or `attachments[]` (anything supplied alongside it). Most use cases do.
2. If you genuinely need new top-level fields or new attachment purposes/analysis types, write an extension draft. The `vcon-speckit` repo has templates; the IETF VCON working group is the venue.
