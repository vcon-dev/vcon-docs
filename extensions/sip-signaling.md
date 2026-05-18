---
description: SIP call signaling metadata, STIR/SHAKEN certificates, and signaling traces attached to a vCon.
---

# 📞 SIP Signaling Extension

**Draft:** [`draft-howe-vcon-sip-signaling`](https://datatracker.ietf.org/doc/draft-howe-vcon-sip-signaling/) · **Extension name:** `"sip-signaling"`

## What it is

A vCon built from a SIP call contains the audio (in dialog) and the parties, but by default loses everything the SIP infrastructure knew about the call — Call-IDs, From/To tags, CSeq numbers, the SDP that negotiated the media, the STIR/SHAKEN certificate chain that authenticated the caller. That metadata matters for fraud investigation, regulatory compliance (TRACED Act), and correlation with carrier-side logs.

The SIP Signaling extension carries that data into the vCon as structured party fields, dialog fields, and attachments.

## When to use it

- Carrier and enterprise SIP environments where calls flow through SIP recording (SIPREC) or media gateways
- TRACED Act / robocall mitigation: STIR/SHAKEN attestation data attached to the vCon
- Fraud investigation: correlating vCons with carrier-side CDR / call traces
- Operational debugging: keeping the signaling envelope alongside the recording when something went wrong

## Spec surface

### Party fields

Each party can carry SIP-specific identity and User-Agent info:

```json
{
  "parties": [
    {
      "tel": "+15551234567",
      "name": "Alice",
      "role": "customer",
      "sip": "alice@example.com",
      "sip_contact": "sip:alice@192.0.2.1:5060",
      "sip_user_agent": "ExamplePhone/2.1",
      "sip_display_name": "Alice Anderson"
    }
  ]
}
```

### Dialog fields

Each dialog (call leg) can carry the SIP dialog identifiers:

```json
{
  "dialog": [
    {
      "type": "recording",
      "start": "2026-05-18T14:00:00Z",
      "parties": [0, 1],
      "mediatype": "audio/x-wav",
      "url": "https://example.com/recording.wav",
      "content_hash": "sha512-...",
      "sip_call_id": "a84b4c76e66710@pc33.example.com",
      "sip_from_tag": "1928301774",
      "sip_to_tag": "a6c85cf",
      "sip_cseq": "314159 INVITE"
    }
  ]
}
```

### Signaling attachments

Raw SIP messages, SDP, and STIR/SHAKEN data attach via `purpose:`:

| `purpose` value | Content |
|-----------------|---------|
| `sip-invite` | The initial INVITE message |
| `sip-response` | A SIP response (200 OK, 4xx, etc.) |
| `sip-message-trace` | A full message sequence trace |
| `sip-sdp` | SDP offer/answer payload |
| `sip-headers` | A relevant header subset |
| `stir-certificate` | STIR certificate chain (PEM or x5c) |
| `stir-verification-report` | Verification result from a STIR validator |
| `stir-passport-extended` | Extended STIR PASSporT |

```json
{
  "attachments": [
    {
      "purpose": "sip-invite",
      "party": 0,
      "dialog": 0,
      "mediatype": "message/sip",
      "encoding": "none",
      "body": "INVITE sip:bob@example.com SIP/2.0\r\nVia: SIP/2.0/UDP ...\r\n..."
    },
    {
      "purpose": "stir-passport-extended",
      "party": 0,
      "dialog": 0,
      "mediatype": "application/passport+jwt",
      "encoding": "none",
      "body": "eyJhbGciOi..."
    }
  ]
}
```

Declare the extension at the top level:

```json
{
  "vcon": "0.4.0",
  "extensions": ["sip-signaling"]
}
```

## Working with SIPREC

The [`vcon-siprec-adapter`](https://github.com/vcon-dev/vcon-siprec-adapter) tool consumes SIPREC-formatted recordings and produces vCons that already include the SIP signaling extension data. If you're ingesting from a SIP recording infrastructure, that adapter is the right starting point — it handles the field mapping for you.

## See also

- [vCon Adapter Development Guide](../vcon-adapters/vcon-adapter-development-guide.md) — patterns for building telephony adapters.
- [Authenticating and Certifying Conversations](../use-cases-studies/authenticating-and-certifying-conversations.md) — the STIR/SHAKEN integration story.
