---
description: >-
  The one page to check a vCon field, extension token or draft revision
  against, verified line by line against draft-ietf-vcon-vcon-core-04.
icon: table-list
---

# vCon Field Reference

Every other page on this site links here for field names. The rows below were checked against [`draft-ietf-vcon-vcon-core-04`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) and the extension drafts listed next. Where this page and a draft disagree, the draft wins. Section numbers refer to core-04.

## Current drafts

Revisions as listed on datatracker on 2026-10-06. Each link goes to the draft's datatracker page, which always shows the newest revision.

| Draft | Revision | What it covers |
| --- | --- | --- |
| [draft-ietf-vcon-vcon-core](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) | 04 | The JSON container. Syntax `"vcon": "0.4.0"`. |
| [draft-ietf-vcon-overview](https://datatracker.ietf.org/doc/draft-ietf-vcon-overview/) | 02 | Use cases and architecture. Informational. |
| [draft-ietf-vcon-privacy-primer](https://datatracker.ietf.org/doc/draft-ietf-vcon-privacy-primer/) | 01 | Privacy background for implementers. Informational. |
| [draft-ietf-vcon-cc-extension](https://datatracker.ietf.org/doc/draft-ietf-vcon-cc-extension/) | 02 | Contact center extension, token `CC`. |
| [draft-howe-vcon-lawful-basis](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/) | 02 | Lawful basis attachment, token `lawful_basis`. |
| [draft-howe-vcon-wtf-extension](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/) | 02 | World Transcription Format analysis, token `wtf_transcription`. |
| [draft-howe-vcon-lifecycle](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/) | 01 | Lifecycle events on a SCITT transparency service. No extension token. |
| [draft-howe-vcon-agent-session](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/) | 00 | AI agent sessions, token `agent_session`. |
| [draft-howe-vcon-sip-signaling](https://datatracker.ietf.org/doc/draft-howe-vcon-sip-signaling/) | 00 | SIP and STIR/SHAKEN signaling data, token `sip-signaling`. |
| [draft-howe-vcon-provenance](https://datatracker.ietf.org/doc/draft-howe-vcon-provenance/) | 00 | Generation provenance for model output, token `provenance`. |
| [draft-birkholz-verifiable-agent-conversations](https://datatracker.ietf.org/doc/draft-birkholz-verifiable-agent-conversations/) | 01 | Verifiable Agent Conversations (VAC). Not a vCon extension; agent-session embeds its records. |

All of these are Internet-Drafts, which means work in progress. Field names can still change between revisions.

## Conventions

* A parameter is mandatory unless the draft marks it optional (Section 2.2). Empty or null objects and arrays MAY be left out.
* `Date` is an RFC 3339 string, for example `2026-10-06T14:00:00Z`.
* Indexes (`parties`, `dialog`, `party`, `originator`) are zero-based positions in the vCon's arrays. Arrays keep the order entries were added. When redaction removes an array entry it SHOULD leave an empty placeholder so indexes do not shift (Section 4.1.8).
* Parameter names are snake_case (Section 2.5).

## Top-level object

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `vcon` | String | Optional | MUST be `"0.4.0"` for core-04 syntax. Deprecated once the draft becomes an RFC; extensions replace schema versioning (4.1.1). |
| `uuid` | String | Yes | Globally unique. SHOULD be a version 8 UUID built like a version 7 UUID with the custom bits taken from the SHA-1 hash of a host name the creator controls (4.1.2). |
| `extensions` | String[] | Optional | SHOULD list every extension whose parameters appear in the vCon (4.1.3). |
| `critical` | String[] | Optional | Extensions that are incompatible with core. A consumer that does not support one of them MUST NOT process the vCon except to reject it or report it (4.1.4). |
| `created_at` | Date | Yes | Does not change after creation (4.1.5). |
| `updated_at` | Date | Optional | Last modification, or last signing time for a signed vCon (4.1.6). |
| `subject` | String | Optional | Free text (4.1.7). |
| `redacted` | Redacted | Optional | Points to the less redacted prior version. Mutually exclusive with `amended` (4.1.8). |
| `amended` | Amended | Optional | Points to the prior version this one adds to. Mutually exclusive with `redacted` (4.1.9). |
| `group` | | Reserved | Registered as reserved for a future extension (6.3.2). Do not emit it. |
| `parties` | Party[] | Optional | |
| `dialog` | Dialog[] | Optional | |
| `analysis` | Analysis[] | Optional | |
| `attachments` | Attachment[] | Optional | |

An unsigned vCon SHOULD contain at least one of `parties`, `dialog`, `analysis` or `attachments` (Section 4).

**Redacted object:** `uuid` (SHOULD), `type` (kind of redaction), and optionally `url` with `content_hash`. **Amended object:** `uuid` (optional only when `url` is given), and optionally `url` with `content_hash`. In both, `content_hash` MUST be present when `url` is.

## Party object

Every field is optional. A party with no fields is valid and still counts as a party (4.2).

| Field | Type | Notes |
| --- | --- | --- |
| `tel` | String | tel URL. The `tel:` prefix is optional (4.2.1). |
| `sip` | String | SIP addr-spec per RFC 3261 Section 25.1, for example `sip:alice@example.com` (4.2.2). |
| `stir` | String | STIR PASSporT in JWS compact form (4.2.3). |
| `mailto` | String | Email address, bare or as a mailto URL (4.2.4). |
| `name` | String | Free text. `"anonymous"` signals a deliberately unidentified party (4.2.5). |
| `did` | String | Decentralized Identifier URI (4.2.6). |
| `validation` | String | Label for how identity was checked, such as `"DOB"`. SHOULD be present when `name` is. Never the validating data itself (4.2.7). |
| `gmlpos` | String | Position in PIDF-LO `gml:pos` form (4.2.8). |
| `civicaddress` | Object | Lowercase keys `country`, `a1` to `a6`, `prd`, `pod`, `sts`, `hno`, `hns`, `lmk`, `loc`, `flr`, `nam`, `pc` (4.2.9). |
| `uuid` | String | Stable participant ID. Any unique string, not limited to UUID syntax (4.2.10). |
| `type` | String | SHOULD be `"person"`, `"bot"` or `"organization"` (4.2.11). |
| `org` | String | Organization identifier (4.2.12). |
| `dept` | String | Department identifier (4.2.13). |

`role` and `contact_list` are not core fields. They come from the CC extension.

## Dialog object

`type` MUST be one of five values (4.3.1, Table 5):

| `type` | Meaning | Content |
| --- | --- | --- |
| `recording` | Audio or video of a segment of the conversation | `body` or `url` |
| `text` | Text from one party, such as a chat message or one email | `body` or `url` |
| `recording-set` | Groups several `recording` entries that together make one call | None. Lists them in `recordings` |
| `transfer` | Records a call transfer: who was transferred, by whom, to whom | None |
| `incomplete` | A call or session that failed before any conversation | None. `disposition` is required |

There is no `transcript` dialog type. A transcript is analysis.

| Field | Type | Applies to | Notes |
| --- | --- | --- | --- |
| `type` | String | all | Required. |
| `start` | Date | all | SHOULD be present; optional for `transfer` (4.3.2). |
| `duration` | UnsignedInt or UnsignedFloat | all but `transfer` | Seconds (4.3.3). |
| `parties` | index, index[], or array of either per channel | all but `transfer` | SHOULD be present for `recording`, `recording-set`, `text`. Multi-channel recordings use one array entry per channel, `null` for an unused channel (4.3.4). |
| `originator` | UnsignedInt | all but `transfer` | Only when the first listed party is not the originator (4.3.5). |
| `recordings` | UnsignedInt[] | `recording-set` | Required there, forbidden elsewhere (4.3.6). |
| `recording_set` | UnsignedInt | `recording` | Index of the owning `recording-set` (4.3.7). |
| `mediatype` | String | `recording`, `text` | MUST be present for inline content (4.3.8). |
| `filename` | String | `recording`, `text` | (4.3.9) |
| `body`, `encoding` | | `recording`, `text` | Inline content (4.3.10). |
| `url`, `content_hash` | | `recording`, `text` | External content (4.3.10). |
| `disposition` | String | `incomplete` | One of `no-answer`, `congestion`, `failed`, `busy`, `hung-up`, `voicemail-no-message` (4.3.11). |
| `session_id` | SessionId, or array matching `parties` | all but `transfer` | Object `{"local": uuid, "remote": uuid}` per RFC 7989. Use `{}` for a party with none (4.3.12). |
| `party_history` | Party_History[] | all but `transfer` | Entries with `party`, `time`, `event` (`join`, `drop`, `hold`, `unhold`, `mute`, `unmute`, `keydown`, `keyup`) and `button` for key events (4.3.13). |
| `transferee`, `transferor`, `transfer_target` | UnsignedInt | `transfer` | Party indexes (4.3.14). |
| `original`, `consultation`, `target_dialog` | UnsignedInt | `transfer` | Dialog indexes (4.3.14). |
| `application` | String | all | Platform or service the conversation ran on (4.3.15). |
| `message_id` | String | `recording`, `text` | Source system message ID, useful for de-duplicating email (4.3.16). |

Suggested `mediatype` values for dialog: `text/plain`, `audio/x-wav`, `audio/x-mp3`, `audio/x-mp4`, `audio/ogg`, `video/x-mp4`, `video/ogg`, `multipart/mixed`.

## Analysis object

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `type` | String | Yes | SHOULD be one of `report`, `sentiment`, `summary`, `transcript`, `translation`, `tts`. Extensions add others, such as `wtf_transcription` and `agent_trace` (4.5.1). |
| `dialog` | index or index[] | When derived from dialog | (4.5.2) |
| `attachment` | index or index[] | When derived from attachments | (4.5.3) |
| `mediatype` | String | SHOULD for inline content | (4.5.4) |
| `filename` | String | Optional | (4.5.5) |
| `vendor` | String | Yes | Vendor or product that produced the analysis (4.5.6). |
| `product` | String | Optional | (4.5.7) |
| `schema` | String | Optional | Token or URL naming the body's format (4.5.8). |
| `body`, `encoding` or `url`, `content_hash` | | SHOULD | Content (4.5.9). |

## Attachment object

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `purpose` | String | Optional in core | Free text describing the attachment. Extensions define specific values, such as `lawful_basis` (4.4.1). |
| `start` | Date | Yes | When the attachment was exchanged (4.4.2). |
| `party` | UnsignedInt | Yes | Party that contributed it. The organization building the vCon SHOULD itself be a party when it adds attachments (4.4.3). |
| `dialog` | UnsignedInt | Yes | Related dialog index (4.4.4). |
| `mediatype` | String | MUST for inline content | (4.4.5) |
| `filename` | String | Optional | (4.4.6) |
| `body`, `encoding` or `url`, `content_hash` | | SHOULD | Content (4.4.7). |

## Inline content and encoding

Inline content uses `body` with `encoding`. `encoding` MUST be present when `body` is a non-empty string (2.3.2). It takes one of three values:

| `encoding` | `body` holds |
| --- | --- |
| `none` | The payload as a JSON string, unchanged. Use for plain text. |
| `base64url` | The payload Base64url encoded (RFC 7515 Section 2). Use for binary. Not plain base64. |
| `json` | Any JSON value: object, array, number, string, `true`, `false` or `null`. |

With `encoding: "json"`, write the object itself:

```json
{ "encoding": "json", "body": { "summary": "Alice asked Bob to confirm a Thursday appointment." } }
```

A stringified body such as `"body": "{\"summary\": \"...\"}"` is still a valid JSON value, but it is a string, so readers have to parse it twice. Write the value form; accept both when reading.

## External content and content_hash

External content uses `url` with `content_hash` (2.4). The URL MUST be HTTPS. `content_hash` is a string, or an array of strings for several algorithms, formed as the algorithm name, a hyphen, and the Base64url SHA-512 digest of the referenced bytes:

```
sha512-GLy6IPaIUM1GqzZqfIPZlWjaDsNgNvZM0iCONNThnH0a75fhUM6cYzLZ5GynSURREvZwmOh54-2lRRieyj82UQ
```

SHA-512 MUST be supported. Hex digests, padded base64 and SHA-256-only hashes do not conform. Generating one in Python:

```python
import base64, hashlib

def content_hash(data: bytes) -> str:
    digest = hashlib.sha512(data).digest()
    return "sha512-" + base64.urlsafe_b64encode(digest).rstrip(b"=").decode()
```

## extensions and critical

An extension is either Compatible (adds data an unaware reader can ignore) or Incompatible (changes meaning). List every extension you use in `extensions`. List an extension in `critical` only when it is Incompatible, or when its own draft tells you to. A reader that finds an unsupported name in `critical` MUST reject the vCon (2.5, 4.1.4).

There is no `must_understand` field in any vCon draft. `must_support` was the 0.3.0 name for `critical`.

## Extension tokens

| Extension | Token in `extensions` | Where its data lives | `critical` |
| --- | --- | --- | --- |
| [Lawful Basis](../extensions/lawful-basis.md) | `lawful_basis` | `attachments[]` with `purpose: "lawful_basis"`, `encoding: "json"` | Not required (LB-02 Section 4.1) |
| [WTF Transcription](../extensions/wtf-transcription.md) | `wtf_transcription` | `analysis[]` with `type: "wtf_transcription"`, `encoding: "json"` | Not required (WTF-02 Section 5.1) |
| [Agent Session](../extensions/agent-session.md) | `agent_session` | `parties[]` with `role: "agent"` and `meta.agent_session`; `analysis[]` with `type: "agent_trace"`; `attachments[]` with `purpose` `agent_file_change`, `agent_artifact`, `agent_environment`, `scitt_receipt`, `agent_trace_cose_sign1` | SHOULD be listed when consumers must process the trace |
| [SIP Signaling](../extensions/sip-signaling.md) | `sip-signaling` | Party `sip_contact`, `sip_user_agent`, `sip_display_name`; dialog `sip_call_id`, `sip_from_tag`, `sip_to_tag`, `sip_cseq`; attachment purposes listed on its page | MUST NOT be listed |
| Contact Center | `CC` | Party `role`, `contact_list`; dialog `campaign`, `interaction_type`, `interaction_id`, `skill` | Not required |
| Provenance | `provenance` | A `provenance` object on an analysis or dialog entry | Not required |
| [Lifecycle](../extensions/lifecycle.md) | None | Events recorded on a SCITT transparency service, not in the vCon | Not applicable |

Spell tokens exactly as shown. `sip-signaling` uses a hyphen; the others use underscores; `CC` is uppercase.

## Legacy names

Read these on old data if you must. Never write them.

| Legacy | Current | Source |
| --- | --- | --- |
| `appended` | `amended` | Core-04 Section 7.1 (0.3.0 to 0.4.0) |
| `must_support` | `critical` | Core-04 Section 7.1 |
| `must_understand` | `critical` | Never in any draft. Earlier pages on this site used it in error |
| `session_id` as a string | `session_id` as `{"local", "remote"}` object | Core-04 Section 7.1 |
| `transfer-target`, `target-dialog` | `transfer_target`, `target_dialog` | Core-04 Section 7.2 (0.0.2 to 0.3.0) |
| `mimetype` | `mediatype` | Core-04 Section 7.3 (0.0.1 to 0.0.2) |
| `alg` plus `signature` | `content_hash` | Core-04 Section 7.3 |
| attachment `type` | attachment `purpose` | Core-04 defines only `purpose`. Older libraries wrote `type`, including for lawful basis |
| analysis `schema_version` | analysis `schema` | Core-04 defines only `schema` |
| token `wtf` | token `wtf_transcription` | WTF-02 Section 5.2 |
| `"vcon": "0.0.1"`, `"0.0.2"`, `"0.3.0"` | `"vcon": "0.4.0"` | Core-04 Section 4.1.1 |

## Signed and encrypted forms

An unsigned vCon is the plain JSON object above. A signed vCon uses the General JWS JSON Serialization whose payload is the unsigned vCon; its header MUST carry `alg` (SHOULD be `RS256`) and `x5c` or `x5u`, and SHOULD carry the vCon `uuid` (5.2). An encrypted vCon is a JWE whose plaintext is the signed vCon: sign first, then encrypt the whole object (5.3). Core-04 does not define encryption of individual analysis bodies. A vCon leaving its security domain SHOULD be signed or encrypted (Section 4); nothing signs it automatically.

The media types are `application/vcon` and `application/vcon+gzip` (6.1, 6.2).

## A complete minimal vCon

This vCon validates against the JSON schema in the core draft's repository. It has one text dialog, one analysis entry with a JSON-value body, and a lawful basis attachment. Use it as a starting point, and replace the `uuid` with one you generate.

```json
{
  "vcon": "0.4.0",
  "uuid": "01a1125f-3397-860d-832a-bc92ac6830cd",
  "created_at": "2026-10-06T14:00:00Z",
  "extensions": ["lawful_basis"],
  "parties": [
    { "tel": "+12025550100", "name": "Alice" },
    { "mailto": "bob@example.com", "name": "Bob", "type": "person" }
  ],
  "dialog": [
    {
      "type": "text",
      "start": "2026-10-06T14:00:00Z",
      "parties": [0, 1],
      "mediatype": "text/plain",
      "encoding": "none",
      "body": "Hi Bob, can you confirm my appointment for Thursday?"
    }
  ],
  "analysis": [
    {
      "type": "summary",
      "dialog": 0,
      "vendor": "example.com",
      "product": "summarizer",
      "mediatype": "application/json",
      "encoding": "json",
      "body": { "summary": "Alice asked Bob to confirm a Thursday appointment." }
    }
  ],
  "attachments": [
    {
      "purpose": "lawful_basis",
      "start": "2026-10-06T14:00:00Z",
      "party": 0,
      "dialog": 0,
      "mediatype": "application/json",
      "encoding": "json",
      "body": {
        "lawful_basis": "consent",
        "expiration": "2027-10-06T14:00:00Z",
        "purpose_grants": [
          { "purpose": "recording", "granted": true, "granted_at": "2026-10-06T14:00:00Z" },
          { "purpose": "analysis", "granted": true, "granted_at": "2026-10-06T14:00:00Z" }
        ]
      }
    }
  ]
}
```

The lawful basis here is illustrative data for an example conversation, not a record of anyone's consent.

## See also

* [Spec Compliance Checklist](../vcon-adapters/spec-compliance-checklist.md) for review rules when building adapters.
* [Extensions](../extensions/README.md) for what each extension is for.
