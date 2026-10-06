---
description: >-
  Shows how to keep SIP call identifiers, raw signaling and STIR/SHAKEN data
  with the recording, so a vCon can be matched to carrier logs and caller
  authentication can be checked later.
---

# 📞 SIP Signaling Extension

**Draft:** [`draft-howe-vcon-sip-signaling-00`](https://datatracker.ietf.org/doc/draft-howe-vcon-sip-signaling/) · **Extension token:** `sip-signaling`

## Purpose of the extension

A vCon built from a SIP call keeps the audio and the parties, but core-04 only has room for the party `sip` and `stir` values and the dialog `session_id`. Call-IDs, dialog tags, CSeq numbers, the raw INVITE, the SDP and the STIR certificate chain are lost. That data is what you need to match a vCon to carrier logs, investigate fraud, or check caller authentication under rules such as the TRACED Act. This extension adds optional party and dialog parameters and a set of attachment `purpose` values to carry it.

The extension is Compatible (draft Section 3.1). A vCon that uses any of its parameters or purposes SHOULD list `sip-signaling` in `extensions`. The token MUST NOT be listed in `critical`.

## Example

A reduction of the draft's Appendix B.1, updated to core-04 (`"vcon": "0.4.0"`, `uuid`, `created_at`, a `sip:` scheme on party `sip`) and with a lawful basis attachment added:

```json
{
  "vcon": "0.4.0",
  "uuid": "01a1125f-3397-860d-832a-bc92ac6830cd",
  "created_at": "2026-01-15T14:30:00Z",
  "extensions": ["sip-signaling", "lawful_basis"],
  "parties": [
    {
      "tel": "+12025550100",
      "sip": "sip:alice@example.com",
      "name": "Alice",
      "stir": "eyJhbGciOiJFUzI1NiIsInBwdCI6InNoYWtlbiIsIn...",
      "sip_contact": "sip:alice@pc33.example.com",
      "sip_user_agent": "ExamplePhone/2.1",
      "sip_display_name": "Alice A."
    },
    {
      "tel": "+12025550199",
      "sip": "sip:bob@biloxi.example.com",
      "name": "Bob"
    }
  ],
  "dialog": [
    {
      "type": "recording",
      "start": "2026-01-15T14:30:05.200+00:00",
      "duration": 312.5,
      "parties": [0, 1],
      "mediatype": "audio/x-wav",
      "url": "https://example.com/recordings/call-1.wav",
      "content_hash": "sha512-GLy6IPaIUM1GqzZqfIPZlWjaDsNgNvZM0iCONNThnH0a75fhUM6cYzLZ5GynSURREvZwmOh54-2lRRieyj82UQ",
      "sip_call_id": "a84b4c76e66710@pc33.example.com",
      "sip_from_tag": "1928301774",
      "sip_to_tag": "a6c85cf",
      "sip_cseq": 314159
    }
  ],
  "attachments": [
    {
      "purpose": "sip-invite",
      "start": "2026-01-15T14:30:00.000+00:00",
      "party": 0,
      "dialog": 0,
      "mediatype": "message/sip",
      "encoding": "base64url",
      "body": "SU5WSVRFIHNpcDpib2JA..."
    },
    {
      "purpose": "sip-response",
      "start": "2026-01-15T14:30:05.200+00:00",
      "party": 1,
      "dialog": 0,
      "mediatype": "message/sip",
      "encoding": "base64url",
      "body": "U0lQLzIuMCAyMDAgT0sN..."
    },
    {
      "purpose": "lawful_basis",
      "start": "2026-01-15T14:30:00Z",
      "party": 0,
      "dialog": 0,
      "mediatype": "application/json",
      "encoding": "json",
      "body": {
        "lawful_basis": "legitimate_interests",
        "expiration": null,
        "purpose_grants": [
          { "purpose": "recording", "granted": true, "granted_at": "2026-01-15T14:30:00Z" }
        ]
      }
    }
  ]
}
```

The `stir`, `body` and `content_hash` values are truncated or placeholder values, as in the draft. The draft's B.1 example still says `"vcon": "0.0.2"` and writes party `sip` without the `sip:` scheme; core-04 defines `sip` as an RFC 3261 addr-spec, which includes the scheme.

## Party parameters

All optional (Section 4). They supplement the core `sip` and `stir` parameters.

| Parameter | Type | Value |
| --- | --- | --- |
| `sip_contact` | String | addr-spec from the Contact header, scheme included |
| `sip_user_agent` | String | Full User-Agent header value |
| `sip_display_name` | String | Display name as claimed in From or To, which may differ from `name` |

## Dialog parameters

All optional (Section 5). When both are available, include `session_id` and `sip_call_id`.

| Parameter | Type | Value |
| --- | --- | --- |
| `sip_call_id` | String | Complete Call-ID header value. Repeat it on every dialog from the same SIP dialog |
| `sip_from_tag` | String | `tag` parameter of the From header |
| `sip_to_tag` | String | `tag` parameter of the To header. May be absent if the dialog was never established |
| `sip_cseq` | UnsignedInt | Sequence number from the CSeq of the dialog-creating request, as an integer (`314159`, not `"314159 INVITE"`) |

## Attachment purposes

Signaling attachments SHOULD carry `purpose`, `start`, `party`, `dialog` and `mediatype` when available (Section 6). Note that core-04 requires `start`, `party` and `dialog` on every attachment; some of the draft's examples leave them out.

| `purpose` | Contents | `mediatype` |
| --- | --- | --- |
| `sip-invite` | An INVITE request | `message/sip` |
| `sip-response` | A provisional or final response | `message/sip` |
| `sip-ack` | An ACK request | `message/sip` |
| `sip-bye` | A BYE request | `message/sip` |
| `sip-cancel` | A CANCEL request | `message/sip` |
| `sip-update` | An UPDATE request (RFC 3311) | `message/sip` |
| `sip-refer` | A REFER request (RFC 3515) | `message/sip` |
| `sip-message-trace` | The full message sequence as JSON (`version`, `call_id`, `messages[]`) | `application/json` |
| `sip-sdp` | An SDP body | `application/sdp` |
| `sip-headers` | Selected headers as a JSON object | `application/json` |
| `stir-certificate` | The certificate or chain that signed the PASSporT | `application/pkix-cert` or `application/pem-certificate-chain` |
| `stir-verification-report` | A STIR verifier's result, including attestation level `A`, `B` or `C` | `application/json` |
| `stir-passport-extended` | A PASSporT beyond the compact one in the party's `stir` | `application/passport` |

Store whole SIP messages in wire format, with `encoding: "none"` for UTF-8 text or `base64url`. JSON bodies use `encoding: "json"` with the object as the value.

## SIPREC

[`vcon-siprec-adapter`](https://github.com/vcon-dev/vcon-siprec-adapter) ([tool page](../tools/vcon-siprec-adapter.md)) receives SIPREC sessions and writes vCons that use this extension.

## See also

* [Field reference](../vcons/field-reference.md) for core party and dialog fields.
* [Authenticating and Certifying Conversations](../use-cases-studies/authenticating-and-certifying-conversations.md) for the STIR/SHAKEN background.
