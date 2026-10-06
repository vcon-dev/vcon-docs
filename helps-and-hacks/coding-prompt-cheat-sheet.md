---
description: >-
  A block of vCon context you can paste into Cursor, Claude Code or Replit so
  the assistant writes vCons that match draft-ietf-vcon-vcon-core-04.
---

# Coding Prompt Cheat Sheet

Paste everything below the line into your coding assistant's context. It is a condensed copy of the [field reference](../vcons/field-reference.md), which is the page to check when the two differ. If you are building an adapter, also give the assistant the [Spec Compliance Checklist](../vcon-adapters/spec-compliance-checklist.md).

***

## vCon context for code generation

**Spec:** IETF `draft-ietf-vcon-vcon-core-04` (https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/). Write `"vcon": "0.4.0"`.

**Never write these legacy names:**

* `appended` (use `amended`)
* `must_support` or `must_understand` (use `critical`; `must_understand` never existed in any draft)
* `type` on an attachment (use `purpose`, including `"purpose": "lawful_basis"`)
* `mimetype` (use `mediatype`)
* `schema_version` (use `schema`)
* a `session_id` string (use an object `{"local": "<uuid>", "remote": "<uuid>"}`)
* `group` (reserved; do not emit)

### What a vCon is

A JSON object holding one conversation (call, video meeting, SMS, chat, email thread). Three forms: unsigned (plain JSON), signed (JWS wrapping the unsigned form), encrypted (JWE wrapping the signed form). Indexes into `parties`, `dialog` and `attachments` are zero-based array positions.

### Top-level object

```json
{
  "vcon": "0.4.0",          // write this; deprecated once the draft is an RFC
  "uuid": "string",         // REQUIRED; SHOULD be a version 8 UUID (see Python notes)
  "created_at": "Date",     // REQUIRED; RFC 3339 with timezone
  "updated_at": "Date",     // optional
  "subject": "string",      // optional
  "extensions": ["string"], // SHOULD list every extension used, e.g. "lawful_basis", "wtf_transcription"
  "critical": ["string"],   // optional; extensions a reader MUST support or else reject the vCon
  "redacted": {},           // optional; mutually exclusive with amended
  "amended": {},            // optional; mutually exclusive with redacted
  "parties": [],            // optional
  "dialog": [],             // optional
  "analysis": [],           // optional
  "attachments": []         // optional
}
```

Include at least one of `parties`, `dialog`, `analysis`, `attachments`.

### Party object (all fields optional)

```json
{
  "tel": "string",           // tel URL; "tel:" prefix optional
  "sip": "string",           // SIP addr-spec, e.g. "sip:alice@example.com"
  "stir": "string",          // STIR PASSporT, JWS compact form
  "mailto": "string",        // email address, bare or mailto: URL
  "name": "string",          // "anonymous" for a deliberately unidentified party
  "did": "string",           // Decentralized Identifier URI
  "validation": "string",    // label for how identity was checked, e.g. "DOB"; SHOULD be present with name
  "gmlpos": "string",        // PIDF-LO gml:pos
  "civicaddress": {},        // keys: country, a1-a6, prd, pod, sts, hno, hns, lmk, loc, flr, nam, pc
  "uuid": "string",          // stable participant ID, any unique string
  "type": "string",          // "person", "bot" or "organization"
  "org": "string",
  "dept": "string"
}
```

`role` and `contact_list` belong to the contact center extension (token `CC`), not core.

### Dialog object

```json
{
  "type": "string",          // REQUIRED: "recording", "text", "recording-set", "transfer" or "incomplete"
  "start": "Date",           // SHOULD
  "duration": 12.5,          // optional, seconds
  "parties": [0, 1],         // SHOULD for recording, recording-set, text; forbidden for transfer
  "originator": 0,           // only when the first listed party is not the originator
  "mediatype": "string",     // MUST for inline content (recording, text)
  "filename": "string",
  "body": "*",               // inline content, with encoding
  "encoding": "string",      // "none", "base64url" or "json"
  "url": "string",           // external content (HTTPS), with content_hash
  "content_hash": "string",  // "sha512-<base64url digest>", or an array of such strings
  "disposition": "string",   // REQUIRED for incomplete: no-answer, congestion, failed, busy, hung-up, voicemail-no-message
  "session_id": {},          // {"local": uuid, "remote": uuid}
  "party_history": [],       // {party, time, event}; events join, drop, hold, unhold, mute, unmute, keydown, keyup (+ button)
  "recordings": [0],         // recording-set only: indexes of its recording dialogs
  "recording_set": 0,        // recording only: index of its recording-set
  "application": "string",
  "message_id": "string",    // recording and text only
  "transferee": 0, "transferor": 0, "transfer_target": 0,  // transfer only: party indexes
  "original": 0, "consultation": 0, "target_dialog": 0     // transfer only: dialog indexes
}
```

`recording-set`, `transfer` and `incomplete` dialogs carry no content: no `body`, `url`, `mediatype` or `filename`. There is no `transcript` dialog type; transcripts are analysis.

### Analysis object

```json
{
  "type": "string",          // REQUIRED; SHOULD be report, sentiment, summary, transcript, translation or tts; extensions add e.g. wtf_transcription
  "dialog": 0,               // index or array; required when derived from dialog
  "attachment": 0,           // index or array; required when derived from attachments
  "vendor": "string",        // REQUIRED
  "product": "string",       // optional
  "schema": "string",        // optional; names the body format
  "mediatype": "string",     // SHOULD for inline content
  "filename": "string",
  "encoding": "json",
  "body": {}                 // or url + content_hash
}
```

### Attachment object

```json
{
  "purpose": "string",       // what the attachment is; extensions define values such as "lawful_basis"
  "start": "Date",           // REQUIRED
  "party": 0,                // REQUIRED; contributing party (use 0 when none applies)
  "dialog": 0,               // REQUIRED; related dialog (use 0 when none applies)
  "mediatype": "string",     // MUST for inline content
  "filename": "string",
  "encoding": "string",
  "body": "*"                // or url + content_hash
}
```

### Redacted and amended objects

```json
{ "uuid": "string", "type": "string", "url": "string", "content_hash": "string" }  // redacted: uuid SHOULD, type = kind of redaction
{ "uuid": "string", "url": "string", "content_hash": "string" }                   // amended: uuid optional only if url given
```

`content_hash` is required whenever `url` is present. Neither object carries a `body`.

### Content encoding

* `none`: body is a plain JSON string (text).
* `base64url`: body is Base64url (RFC 7515), for binary. Not plain base64.
* `json`: body is any JSON value. Write the object or array itself, not `json.dumps(...)`. Readers should still accept a stringified body.

`encoding` MUST accompany a non-empty `body`. External `url` MUST be HTTPS.

### Common extensions

* Lawful basis: `extensions: ["lawful_basis"]`; attachment `{"purpose": "lawful_basis", "encoding": "json", "body": {"lawful_basis": "consent", "expiration": "<Date or null>", "purpose_grants": [{"purpose": "recording", "granted": true, "granted_at": "<Date>"}]}}`. Optional `proof_mechanisms[]` entries have `proof_type`, `timestamp`, `proof_data`. Not required in `critical`.
* WTF transcript: `extensions: ["wtf_transcription"]`; analysis `{"type": "wtf_transcription", "vendor": "...", "encoding": "json", "body": {"transcript": {...}, "segments": [...], "metadata": {...}}}`.
* SIP signaling: token `sip-signaling` (hyphen). MUST NOT be in `critical`.

### Signed form (JWS)

```json
{
  "payload": "base64url of the unsigned vCon",
  "signatures": [{
    "header": { "alg": "RS256", "x5c": ["..."], "uuid": "vCon uuid" },  // alg SHOULD be RS256; x5c or x5u MUST
    "protected": "base64url",
    "signature": "base64url"
  }]
}
```

### Encrypted form (JWE)

Sign first, then encrypt the whole signed vCon.

```json
{
  "unprotected": { "cty": "application/vcon", "enc": "A256CBC-HS512", "uuid": "vCon uuid" },
  "recipients": [{ "header": { "alg": "RSA-OAEP" }, "encrypted_key": "base64url" }],
  "iv": "base64url",
  "ciphertext": "base64url of the signed vCon",
  "tag": "base64url"
}
```

Nothing is signed automatically. A vCon SHOULD be signed or encrypted before it leaves the security domain that built it.

### Python notes

Timestamps must carry a timezone: `datetime.now(timezone.utc).isoformat()`, never `datetime.utcnow()`.

The core draft recommends a version 8 UUID laid out like version 7 (millisecond timestamp first) with its final 62 bits set to the high 62 bits of the SHA-1 hash of a host name you control:

```python
import hashlib, os, time, uuid

def vcon_uuid(fqhn: str) -> str:
    """Version 8 UUID per draft-ietf-vcon-vcon-core-04 Section 4.1.2."""
    unix_ts_ms = time.time_ns() // 1_000_000
    rand_a = int.from_bytes(os.urandom(2), "big") & 0xFFF
    custom_c = int.from_bytes(hashlib.sha1(fqhn.encode()).digest()[:8], "big") >> 2
    n = (unix_ts_ms & (2**48 - 1)) << 80 | 0x8 << 76 | rand_a << 64 | 0b10 << 62 | custom_c
    return str(uuid.UUID(int=n))

u = uuid.UUID(vcon_uuid("example.com"))
assert u.version == 8 and u.variant == uuid.RFC_4122
```

Content hash for external files:

```python
import base64, hashlib

def content_hash(data: bytes) -> str:
    return "sha512-" + base64.urlsafe_b64encode(hashlib.sha512(data).digest()).rstrip(b"=").decode()
```

The [`vcon`](../vcon-library/README.md) library (0.10.0 or later) does most of this: `Vcon.build_new()` sets `"vcon": "0.4.0"` and a version 8 `uuid`, and `add_analysis()` takes keyword arguments and requires `vendor`.

### Validation rules

* `redacted` and `amended` are mutually exclusive.
* Every index must point at an existing array element.
* `url` always comes with `content_hash`.
* `incomplete` dialogs need `disposition`; `recording-set` dialogs need `recordings`.
* Every attachment has `start`, `party` and `dialog`; every analysis has `vendor`.
* A reader that finds an unsupported name in `critical` rejects the vCon.
