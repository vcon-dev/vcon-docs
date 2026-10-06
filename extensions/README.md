---
description: >-
  Shows which vCon extension drafts exist, what token each one uses, and where
  its data sits in a vCon, so you can add structured data without breaking
  readers.
icon: arrows-from-line
---

# Extensions

The core draft, [`draft-ietf-vcon-vcon-core-04`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), keeps the container small. Anything beyond the core parties, dialog, analysis and attachment fields belongs in an extension: a separate draft that defines new parameters, attachment `purpose` values or analysis `type` values, and registers a token for them.

## How extensions are declared

Two top-level arrays carry extension names (core-04 Sections 4.1.3 and 4.1.4):

* `extensions` lists every extension whose parameters appear in the vCon. It SHOULD be present when any are used.
* `critical` lists the extensions a reader must support to interpret the vCon correctly. A reader that finds a name it does not support in `critical` MUST NOT process the vCon except to reject it or report it.

Core-04 calls an extension Compatible when an unaware reader can ignore it safely, and Incompatible when it changes meaning. Only Incompatible extensions, or ones whose draft says so, go in `critical`. All the extensions below are Compatible.

A top-level fragment for a vCon carrying an agent trace that downstream systems must honor, plus a lawful basis attachment:

```json
{
  "extensions": ["agent_session", "lawful_basis"],
  "critical": ["agent_session"]
}
```

There is no `must_understand` field. `must_support` is the 0.3.0 name for `critical`. See the [field reference](../vcons/field-reference.md#legacy-names) for every renamed field.

## Available extensions

| Extension | Token | Where its data lives | Draft |
| --- | --- | --- | --- |
| [Lawful Basis](lawful-basis.md) | `lawful_basis` | `attachments[]` with `purpose: "lawful_basis"` | [draft-howe-vcon-lawful-basis-02](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/) |
| [WTF Transcription](wtf-transcription.md) | `wtf_transcription` | `analysis[]` with `type: "wtf_transcription"` | [draft-howe-vcon-wtf-extension-02](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/) |
| [Agent Session](agent-session.md) | `agent_session` | agent `parties[]`, `analysis[]` with `type: "agent_trace"`, `attachments[]` with `agent_*` purposes | [draft-howe-vcon-agent-session-00](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/) |
| [SIP Signaling](sip-signaling.md) | `sip-signaling` | `sip_*` party and dialog parameters, `attachments[]` with `sip-*` and `stir-*` purposes | [draft-howe-vcon-sip-signaling-00](https://datatracker.ietf.org/doc/draft-howe-vcon-sip-signaling/) |
| Contact Center | `CC` | party `role`, `contact_list`; dialog `campaign`, `interaction_type`, `interaction_id`, `skill` | [draft-ietf-vcon-cc-extension-02](https://datatracker.ietf.org/doc/draft-ietf-vcon-cc-extension/) |
| Provenance | `provenance` | a `provenance` object on an analysis or dialog entry | [draft-howe-vcon-provenance-00](https://datatracker.ietf.org/doc/draft-howe-vcon-provenance/) |
| [Lifecycle](lifecycle.md) | none | events on a SCITT transparency service, outside the vCon | [draft-howe-vcon-lifecycle-01](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/) |

Lifecycle is listed here because it is usually read alongside Lawful Basis, but it defines no extension token and adds nothing to the vCon itself.

Every attachment uses `purpose`, never `type`. That includes `lawful_basis`.

## Related work outside the vCon WG

[`draft-howe-sipcore-mcp-extension`](https://datatracker.ietf.org/doc/draft-howe-sipcore-mcp-extension/) is a SIP protocol extension that carries MCP payloads inside SIP sessions. It adds nothing to a vCon.

[`draft-birkholz-verifiable-agent-conversations`](https://datatracker.ietf.org/doc/draft-birkholz-verifiable-agent-conversations/) defines Verifiable Agent Conversation records. It is not a vCon extension; the Agent Session extension embeds its records.

## Writing a new extension

Check first whether your data fits as an analysis entry (anything derived from the conversation) or an attachment with a new `purpose` (anything supplied alongside it). Most data does, and needs no new draft.

If you need new parameters, write an Internet-Draft. Core-04 Section 2.5 says what it must contain: the extension name and whether it goes in `extensions`, `critical` or both; each new parameter and where it appears; compatibility and incompatibility considerations; and privacy and integrity considerations. Register the token in the vCon Extensions Names Registry (core-04 Section 6.4). Bring it to the [IETF VCON working group](https://datatracker.ietf.org/wg/vcon/about/).
