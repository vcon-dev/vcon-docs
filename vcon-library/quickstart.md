---
description: Install the Python vCon library and build, save, load, sign and validate a vCon with a lawful basis and a transcript in a few lines.
---

# 🐰 Quickstart

## vCon Library

`vcon` 0.10.0 on PyPI. It targets [`draft-ietf-vcon-vcon-core-04`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) and writes `"vcon": "0.4.0"`. Python 3.12 or later.

```bash
pip install vcon
pip install "vcon[image]"   # adds Pillow and pypdf for image and PDF helpers
```

The package root exports only `Vcon`, `compute_content_hash` and `parse_content_hash_algorithm`. Import everything else from its module: `vcon.party.Party`, `vcon.dialog.Dialog`, `vcon.civic_address.CivicAddress`, `vcon.extensions`. Field rules (what each property means and when it is required) live in the [field reference](../vcons/field-reference.md); this page shows the library calls.

### Build a vCon

This example creates two parties, one text dialog and a lawful basis attachment, then validates, saves and reloads it.

```python
from datetime import datetime, timedelta, timezone

from vcon import Vcon
from vcon.dialog import Dialog
from vcon.party import Party

now = datetime.now(timezone.utc)

vcon = Vcon.build_new()
vcon.add_party(Party(tel="+15551230001", name="Alice", role="caller"))
vcon.add_party(Party(tel="+15551230002", name="Bob", role="agent"))

vcon.add_dialog(Dialog(
    type="text",
    start=now,
    parties=[0, 1],
    body="Hello, I need help with my account.",
    encoding="none",
    mediatype="text/plain",
))

# Record why this conversation may be processed. Adds the lawful_basis
# extension and a purpose "lawful_basis" attachment.
vcon.add_lawful_basis_attachment(
    lawful_basis="consent",
    expiration=(now + timedelta(days=365)).isoformat(),
    purpose_grants=[
        {"purpose": "recording", "granted": True, "granted_at": now.isoformat()},
        {"purpose": "transcription", "granted": True, "granted_at": now.isoformat()},
    ],
    party_index=0,
)

is_valid, errors = vcon.is_valid()
print(is_valid, errors)

vcon.save_to_file("conversation.vcon.json")
loaded = Vcon.load("conversation.vcon.json")
print(loaded.uuid == vcon.uuid)
```

Dialogs need a `mediatype`. Without one, `is_valid()` reports `invalid or missing mediatype`. Pass `mediatype=`, never `mimetype=`: the `Dialog` constructor accepts unknown keyword arguments and would write a `mimetype` property into the vCon.

The library fills spec-required defaults for you. Every attachment gets `party: 0`, `dialog: 0`, a `start` taken from `created_at`, and `mediatype: "application/json"` for JSON bodies, unless you pass your own values.

### Check permission

```python
print(vcon.check_lawful_basis_permission("recording", party_index=0))   # True
print(vcon.check_lawful_basis_permission("marketing", party_index=0))   # False
print(len(vcon.find_lawful_basis_attachments(party_index=0)))           # 1
```

A vCon has a lawful basis when `attachments[]` holds an entry with `purpose: "lawful_basis"`. See the [Lawful Basis extension](../extensions/lawful-basis.md).

### Add a transcript

Put WTF transcripts in `analysis[]`. Use `add_wtf_transcription_analysis()` and read the result back with `Vcon.decoded_body()`.

```python
vcon.add_wtf_transcription_analysis(
    transcript={
        "text": "Hello, I need help with my account.",
        "language": "en",
        "duration": 4.2,
        "confidence": 0.92,
    },
    segments=[{
        "id": 0, "start": 0.0, "end": 4.2,
        "text": "Hello, I need help with my account.",
        "confidence": 0.92, "speaker": 0,
    }],
    metadata={
        "created_at": now.isoformat(),
        "processed_at": now.isoformat(),
        "provider": "whisper",
        "model": "whisper-1",
    },
    dialog_index=0,
)

entry = vcon.find_analysis_by_type("transcription")
wtf = Vcon.decoded_body(entry)
print(wtf["transcript"]["text"])

from vcon.extensions.wtf import WTFAttachment
print(WTFAttachment.from_dict(wtf).export_to_srt())
```

Known upstream gap in 0.10.0: `add_wtf_transcription_analysis()` writes `type: "transcription"`, while [`draft-howe-vcon-wtf-extension`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/) defines `wtf_transcription`, and it still writes the body as a JSON string rather than the JSON value. `decoded_body()` reads both forms. If you need the draft values today, call `add_analysis(type="wtf_transcription", dialog=0, vendor=..., body=wtf_dict, encoding="json")` yourself. `metadata` must include `created_at`, `processed_at`, `provider` and `model`. See the [WTF extension](../extensions/wtf-transcription.md).

`add_wtf_transcription_attachment()` also exists and writes to `attachments[]`. Prefer the analysis form.

### Common operations

#### Parties and civic address

```python
from vcon.civic_address import CivicAddress
from vcon.party import Party

party = Party(
    tel="+15551230003",
    name="Jane Example",
    sip="sip:jane@example.com",
    did="did:example:123456789abcdef",
    jCard={"fn": "Jane Example", "email": "jane@example.com"},
    timezone="America/New_York",
    civicaddress=CivicAddress(
        country="US", a1="CA", a3="San Francisco",
        sts="Market Street", hno="123", pc="94102",
    ),
)
vcon.add_party(party)

print(vcon.parties[-1]["name"])          # parties is a list of dicts
print(len(vcon.get_party_objects()))     # get_party_objects() returns Party objects
```

#### Dialog variants

```python
from vcon.party import PartyHistory

# External recording: reference by URL (placeholder URL; the library does not fetch it here)
vcon.add_dialog(Dialog(
    type="recording",
    start=now,
    parties=[0, 1],
    url="https://example.com/recording.wav",
    mediatype="audio/x-wav",
    party_history=[
        PartyHistory(0, "join", now),
        PartyHistory(1, "join", now),
        PartyHistory(0, "drop", now),
    ],
))

# Call that never connected. Dispositions are lowercase and hyphenated:
# no-answer, congestion, failed, busy, hung-up, voicemail-no-message
vcon.add_incomplete_dialog(start=now, disposition="no-answer", parties=[0])

# Transfer between parties 0, 1 and 2
vcon.add_party(Party(name="Carol", role="supervisor"))
vcon.add_transfer_dialog(
    start=now,
    transfer_data={"transferee": 0, "transferor": 1, "transfer_target": 2},
    parties=[0, 1, 2],
)
print(len(vcon.dialog))
```

`add_external_data(url, filename, mediatype)` downloads the URL and hashes it, so it needs a reachable file. `add_inline_data(body, filename, mediatype)` labels the body `base64url` but does not encode it, so pass a body that is already base64url.

#### Analysis, attachments and tags

```python
vcon.add_analysis(
    type="sentiment",
    dialog=0,
    vendor="example-vendor",
    body={"sentiment": "positive", "confidence": 0.95},
    encoding="json",
)

vcon.add_attachment(
    purpose="notes",
    body={"agent_note": "Customer verified."},
    encoding="json",
)
print(vcon.find_attachment_by_purpose("notes")["party"])   # 0, filled by default

vcon.add_tag("department", "billing")
print(vcon.get_tag("department"))                          # billing
```

A JSON body is the value itself (an object, array or string), not a `json.dumps` string. Use `Vcon.decoded_body(entry)` to read an entry either way. Attachments take `purpose`, never `type`. Images: `vcon.add_image(path, purpose="image")`, which needs `vcon[image]`.

#### Extensions and critical

```python
vcon.add_extension("x-example")
vcon.add_critical("x-example")     # receivers that do not understand it must reject the vCon
print(vcon.get_extensions())       # ['lawful_basis', 'wtf_transcription', 'x-example']
print(vcon.get_critical())         # ['x-example']
vcon.remove_critical("x-example")
vcon.remove_extension("x-example")
```

The top-level fields are `extensions` and `critical`, managed through these methods. `must_support` and `must_understand` are not core-04 names, and the amendment field is `amended`, not `appended`.

#### Content hashes

`content_hash` is `sha512-` followed by unpadded base64url. SHA-512 is the default.

```python
from vcon import compute_content_hash, parse_content_hash_algorithm

h = compute_content_hash(b"hello")
print(h)
print(parse_content_hash_algorithm(h))   # sha512

d = Dialog(type="text", start=now, parties=[0], body="hello",
           encoding="none", mediatype="text/plain")
h = d.calculate_content_hash()           # sha512; pass "sha256" for sha256-
print(d.verify_content_hash(h))          # True
```

#### Sign and verify

```python
private_key, public_key = Vcon.generate_key_pair()
signed = Vcon.build_new()
signed.add_party(Party(name="Alice"))
signed.sign(private_key)
print(signed.verify(public_key))         # True
```

Signing turns the object into a JWS wrapper with `payload` and `signatures`, so sign last.

#### Validate and load

```python
ok, errors = Vcon.validate_file("conversation.vcon.json")
print(ok, errors)

ok, errors = Vcon.validate_json(vcon.to_json())
print(ok)

# placeholder URL, not run: Vcon.load("https://example.com/conversation.vcon.json")
# placeholder URL, not run: vcon.post_to_url("https://example.com/vcons", headers={"x-api-token": "..."})
strict = Vcon.load("conversation.vcon.json", property_handling="strict")
print(strict.uuid == loaded.uuid)
```

`property_handling` is `"default"` (keep unknown properties), `"strict"` (drop them) or `"meta"` (move them into `meta`).

`subject` has no setter. Set it with `vcon.vcon_dict["subject"] = "Refund request"`.

#### Validate extensions

```python
results = vcon.validate_extensions()
print(results["lawful_basis"]["is_valid"])
print(results["wtf_transcription"]["is_valid"])
```

### Release notes

**0.10.0** retargets the library to core-04.

* A `json`-encoded `body` is the JSON value itself. `add_tag()`, `add_lawful_basis_attachment()` and `add_attachment()` write values. Readers accept the older stringified form.
* Attachments created by `add_attachment()`, `add_lawful_basis_attachment()` and `add_wtf_transcription_attachment()` default `party` and `dialog` to `0`, `start` to the vCon's `created_at`, and `mediatype` to `application/json` for JSON bodies. `Attachment(...)` applies the same defaults. Code that tested for the absence of these keys will see a change.
* `Dialog.to_dict()` no longer writes empty `meta` or `metadata`.
* Inline base64url bodies are unpadded. New `vcon.dialog.b64url_encode()` and `b64url_decode()` helpers; decoding accepts padded and unpadded input.
* New `vcon.body.decode_body()`, also `Vcon.decoded_body()`, reads a body in either shape.
* Release on tag push: CI publishes to PyPI on `v*` tags.

**0.9.5** content hashes use the spec form. `Dialog` writes `sha512-<unpadded base64url>` for `add_external_data`, `add_inline_data`, `add_image_data` and `to_inline_data`. `calculate_content_hash()` defaults to `sha512`, and `verify_content_hash()` reads the prefix of the stored hash. Old unprefixed SHA-256 hashes still verify. New exports `compute_content_hash` and `parse_content_hash_algorithm`.

**0.9.6** security: `requests` bump and removal of a stale `requirements.txt`.

**0.9.3** fixed `add_tag()` to include `party: 0` and `dialog: 0`. **0.9.2** `build_new()` emits `"vcon": "0.4.0"` and no empty `group` or `redacted`; added `add_wtf_transcription_analysis()`.

### Known limits in 0.10.0

* `subject` has no setter.
* `add_wtf_transcription_analysis()` uses `type: "transcription"` and a string body (see above).
* `add_inline_data()` does not encode its body, and `add_image_data()` hashes the raw image bytes while `verify_content_hash()` recomputes over the encoded body string, so verification of an image dialog returns `False`.

### Next

* [Library API reference](library-api-reference.md)
* [Guide for LLMs](vcon-library-guide-for-llms.md)
* [Adapter quick start](../vcon-adapters/quick-start-from-template.md)

MIT License.
