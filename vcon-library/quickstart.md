---
description: Install the Python vCon library and build, save, load, sign and validate a vCon with a session, a lawful basis and a transcript in a few lines.
---

# 🐰 Quickstart

## vCon Library

`vcon` 1.0.0 on PyPI. It targets [`draft-ietf-vcon-vcon-core-05`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) and writes `"vcon": "0.5.0"`. It reads 0.4.0 and 0.5.0 and migrates 0.4.0 on load. Python 3.12 or later.

```bash
pip install vcon
pip install "vcon[image]"   # adds Pillow and pypdf for image and PDF helpers
```

The package root exports `Vcon`, `Dialog`, `Session`, `SessionReference`, `Event`, `AlgorithmNotPermittedError`, `MigrationError`, `MigrationReport`, `MigrationResult`, `migrate_0_4_0_to_0_5_0`, `compute_content_hash` and `parse_content_hash_algorithm`. Import the rest from its module: `vcon.party.Party`, `vcon.civic_address.CivicAddress`, `vcon.extensions`. Field rules live in the [field reference](../vcons/field-reference.md), which describes core-04 (`"0.4.0"`); this page shows the library calls and the core-05 shape the library writes.

Importing `vcon` sets the root logger to `INFO` on stdout (through its `logger` dependency), so every call logs a line. Add `logging.getLogger().setLevel(logging.WARNING)` after the import to quiet it.

### Build a vCon

This example creates two parties, one session, one text dialog and a lawful basis attachment, then validates, saves and reloads it.

```python
from datetime import datetime, timedelta, timezone

from vcon import Dialog, Session, Vcon
from vcon.party import Party

now = datetime.now(timezone.utc)

vcon = Vcon.build_new()
vcon.add_party(Party(tel="+15551230001", name="Alice", role="caller"))
vcon.add_party(Party(tel="+15551230002", name="Bob", role="agent"))

# A session groups the dialog of one conversation. Its parties are the
# receivers; the first one is the originator.
vcon.add_session(Session(start=now, parties=[0, 1]))

vcon.add_dialog(Dialog(
    type="text",
    start=now,
    parties=[0],          # contributors only: Alice wrote this message
    body="Hello, I need help with my account.",
    encoding="none",
    mediatype="text/plain",
), session=0)             # returns the dialog index and adds it to sessions[0].dialog

# Record why this conversation may be processed. Adds the lawful_basis
# extension and a purpose "lawful_basis" attachment scoped to the session.
vcon.add_lawful_basis_attachment(
    lawful_basis="consent",
    expiration=(now + timedelta(days=365)).isoformat(),
    purpose_grants=[
        {"purpose": "recording", "granted": True, "granted_at": now.isoformat()},
        {"purpose": "transcription", "granted": True, "granted_at": now.isoformat()},
    ],
    party_index=0,
    session_index=0,
)

is_valid, errors = vcon.is_valid()
print(is_valid, errors)                  # True []
print(vcon.vcon)                         # 0.5.0
print(vcon.sessions[0]["dialog"])        # [0]

vcon.save_to_file("conversation.vcon.json")
loaded = Vcon.load("conversation.vcon.json")
print(loaded.uuid == vcon.uuid)          # True
```

A vCon with dialog or events needs at least one session; without one, `is_valid()` reports `At least one Session Object MUST exist ...`. A dialog that no session lists is a warning, returned by `vcon.get_validation_warnings()`. `add_dialog()` returns the new index; pass `session=` to add it to that session's `dialog` list.

Dialog `parties` lists who contributed to the dialog. The receivers are the `parties` of the session. Give a dialog with a `body` a `mediatype`; without one, `is_valid()` reports `invalid or missing mediatype`. Pass `mediatype=`, never `mimetype=`: the `Dialog` constructor accepts unknown keyword arguments and would write a `mimetype` property into the vCon.

Attachments need `party` and one of `dialog` or `session`. `party` defaults to 0, `start` to the vCon's `created_at`, and `mediatype` to `application/json` for JSON bodies. With neither `dialog` nor `session` given, the helpers use the only session when the vCon has exactly one and raise `ValueError` otherwise.

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
print(wtf["transcript"]["text"])         # Hello, I need help with my account.

from vcon.extensions.wtf import WTFAttachment
print(WTFAttachment.from_dict(wtf).export_to_srt())
```

Known upstream gap, still present in 1.0.0: `add_wtf_transcription_analysis()` writes `type: "transcription"`, while [`draft-howe-vcon-wtf-extension`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/) defines `wtf_transcription`, and it writes the body as a JSON string rather than the JSON value. `decoded_body()` reads both forms. To write the draft values, call `add_analysis(type="wtf_transcription", dialog=0, vendor=..., body=wtf_dict, encoding="json")` and `add_extension("wtf_transcription")` yourself; `is_valid()` reports a `wtf_transcription` analysis whose extension is not listed. `metadata` must include `created_at`, `processed_at`, `provider` and `model`. See the [WTF extension](../extensions/wtf-transcription.md).

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

print(vcon.parties[-1]["name"])          # Jane Example (parties is a list of dicts)
print(len(vcon.get_party_objects()))     # 3 (get_party_objects() returns Party objects)
```

#### Recordings, events and incomplete calls

```python
from vcon import Event

# External recording: reference by URL (placeholder URL; the library does not fetch it here)
rec = vcon.add_dialog(Dialog(
    type="recording",
    start=now,
    parties=[0, 1],
    url="https://example.com/recording.wav",
    mediatype="audio/x-wav",
), session=0)

# Joins, drops, holds and key presses are events on the session, not dialog party_history
vcon.add_event(Event(type="join", time=now, sessions=0, party=0))
vcon.add_event(Event(type="join", time=now, sessions=0, party=1))
vcon.add_event(Event(type="drop", time=now, sessions=0, party=0))

# Call that never connected. Dispositions are lowercase and hyphenated:
# no-answer, congestion, failed, busy, hung-up, voicemail-no-message
vcon.add_incomplete_dialog(start=now, disposition="no-answer", parties=[0], session=0)

# Blind transfer: Bob (1) transfers Alice (0) to Carol (3). Each call is a session.
vcon.add_party(Party(name="Carol", role="supervisor"))
target = vcon.add_session(Session(start=now, parties=[0, 3]))
vcon.add_event(Event(type="blind-transfer", time=now, sessions=0, party=1,
                     transferee=0, transfer_target=3, target_session=target))

print(len(vcon.dialog), len(vcon.sessions), len(vcon.events))   # 3 2 4
print(vcon.is_valid())                                          # (True, [])
```

Dialog types are `recording`, `text` and `incomplete`. `Dialog(type="audio")` and `Dialog(type="video")` still work but warn and build a `recording`. `transfer` and `recording-set` dialogs are gone: `Dialog(party_history=...)`, `originator=`, `transferee=` and the other transfer parameters raise `ValueError`, and `add_transfer_dialog()` raises `NotImplementedError`. An `Event` checks the per-type parameter table when built, so `Event(type="join", time=now, party=0)` raises `ValueError` for the missing `sessions`. `add_incomplete_dialog()` writes no `body`; `details` goes to `metadata["details"]`.

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
    session=0,
)
print(vcon.find_attachment_by_purpose("notes")["party"])   # 0, filled by default

vcon.add_tag("department", "billing")
print(vcon.get_tag("department"))                          # billing
```

A JSON body is the value itself (an object, array or string), not a `json.dumps` string. Use `Vcon.decoded_body(entry)` to read an entry either way. Attachments take `purpose`, never `type`. `add_analysis()` takes `dialog`, `sessions`, `attachment` and `analysis` as optional keyword references. `add_tag()` scopes the tags attachment to every session and adds an empty `{}` session if there is none. Images: `vcon.add_image(path, purpose="image")`, which needs `vcon[image]`.

#### Extensions and critical

```python
vcon.add_extension("x-example")
vcon.add_critical("x-example")     # receivers that do not understand it must reject the vCon
print(vcon.get_extensions())       # ['lawful_basis', 'wtf_transcription', 'x-example']
print(vcon.get_critical())         # ['x-example']
vcon.remove_critical("x-example")
vcon.remove_extension("x-example")
```

The top-level fields are `extensions` and `critical`, managed through these methods. `is_valid()` fails when an extension in use (`lawful_basis`, `wtf_transcription`) is not listed in `extensions`, or when a `critical` entry is not. `must_support` and `must_understand` are not core names. The core-05 names for the prior-version pointers are `redacted_from` and `amended_from`; `vcon.redacted` and `vcon.amended` still work but warn.

#### Content hashes

`content_hash` is `sha512-` followed by unpadded base64url. SHA-512 is the default.

```python
from vcon import compute_content_hash, parse_content_hash_algorithm

h = compute_content_hash(b"hello")
print(h)                                 # sha512-m3HSJL1i83hdltRq0-o9czGb-8KJDKra4t_3JRlnPKcjI8PZm6XBHXx6zG4UuMXaDEZjR1wuXDre9G9zvN7AQw
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
signed.sign(private_key)                 # RS256 by default; alg="none" raises AlgorithmNotPermittedError
print(signed.verify(public_key))         # True
```

Signing adds `payload` and `signatures` to the vCon, so sign last. `sign()` takes `alg=` (default `RS256`) and refuses `none`; `verify()` raises `AlgorithmNotPermittedError` for a signature that declares `alg: "none"`.

#### Validate and load

```python
ok, errors = Vcon.validate_file("conversation.vcon.json")
print(ok, errors)                        # True []

ok, errors = Vcon.validate_json(vcon.to_json())
print(ok)                                # True

# placeholder URL, not run: Vcon.load("https://example.com/conversation.vcon.json")
# placeholder URL, not run: vcon.post_to_url("https://example.com/vcons", headers={"x-api-token": "..."})
strict = Vcon.load("conversation.vcon.json", property_handling="strict")
print(strict.uuid == loaded.uuid)        # True
```

`property_handling` is `"default"` (keep unknown properties), `"strict"` (drop them) or `"meta"` (move them into `meta`).

`subject` has no setter. Set it with `vcon.vcon_dict["subject"] = "Refund request"`.

#### Validate extensions

```python
results = vcon.validate_extensions()
print(results["lawful_basis"]["is_valid"])        # True
print(results["wtf_transcription"]["is_valid"])   # True
```

### Load a 0.4.0 vCon

`Vcon.load()`, `Vcon.load_from_file()`, `Vcon.load_from_url()`, `Vcon.build_from_json()` and `Vcon(dict)` migrate a vCon that declares `"vcon": "0.4.0"` to 0.5.0 and keep a report of what the migration did.

```python
import json

old = {
    "vcon": "0.4.0",
    "uuid": "01928e10-193e-8231-b9a2-279e0d16bc46",
    "created_at": "2026-10-06T14:00:00+00:00",
    "parties": [{"tel": "+15551230001", "name": "Alice"},
                {"tel": "+15551230002", "name": "Bob"}],
    "dialog": [{
        "type": "recording", "start": "2026-10-06T14:00:00+00:00", "duration": 62.5,
        "parties": [0, 1], "originator": 0, "mediatype": "audio/x-wav",
        "url": "https://example.com/recording.wav", "content_hash": "sha512-GLy6IPa",
        "party_history": [
            {"party": 0, "event": "join", "time": "2026-10-06T14:00:00+00:00"},
            {"party": 1, "event": "join", "time": "2026-10-06T14:00:02+00:00"},
        ],
    }],
}
with open("old.vcon.json", "w") as f:
    json.dump(old, f)

migrated = Vcon.load("old.vcon.json")
print(migrated.vcon)                                # 0.5.0
print(migrated.migration_report.summary())          # 2 inferred, 0 fallback, 0 unresolved
print(migrated.sessions[0]["dialog"])               # [0]
print([e["type"] for e in migrated.events])         # ['join', 'join']
print(migrated.is_valid())                          # (True, [])

from vcon import migrate_0_4_0_to_0_5_0
new_dict, report = migrate_0_4_0_to_0_5_0(old, strict=False)

raw = Vcon.load("old.vcon.json", migrate_legacy=False)
print(raw.vcon)                                     # 0.4.0, kept as is; is_valid() fails
```

The migration follows Appendix A.1 of core-05. It adds a session for the dialogs, turns `party_history` into events, moves `originator` to the session, turns `transfer` dialogs into transfer events and `recording-set` dialogs into sessions, and renames `redacted` and `amended` to `redacted_from` and `amended_from`. A removed dialog becomes `{}` in place, so every index stays valid. `report.inferred`, `report.fallbacks` and `report.unresolved` list what was not dictated by the draft, and `report.removed_dialogs` holds the original content of each removed dialog. Store `report.to_dict()` with the migrated record. `strict=True` raises `MigrationError` instead of using a fallback for a required field. A vCon that is already 0.5.0 raises `MigrationError`.

A signed 0.4.0 vCon is not migrated on load, because migrating drops the signature. Verify it, call `Vcon.migrate_0_4_0_to_0_5_0(signed)`, and sign the result again (`report.resign_required` is `True`).

### Release notes

**1.0.0** retargets the library to core-05 (syntax `"0.5.0"`). It is a breaking release.

* New `Session` and `SessionReference` (top-level `sessions`, `add_session()`), and `Event` (top-level `events`, `add_event()`). Events enforce the per-type parameter table in `Event(...)` and `is_valid()`.
* `build_new()` writes `"vcon": "0.5.0"`. A 0.4.0 vCon is migrated on load; `migrate_legacy=False` keeps it as is. New `migrate_0_4_0_to_0_5_0()`, `Vcon.migrate_0_4_0_to_0_5_0()`, `Vcon.migration_report`, `MigrationReport` and `MigrationError`.
* Dialog types are `recording`, `text` and `incomplete`. `transfer`, `recording-set`, `party_history`, `originator` and the transfer parameters are gone. `audio` and `video` warn and build a `recording`. `PartyHistory` is deprecated and converts with `to_event(sessions)`. `add_transfer_dialog()` raises `NotImplementedError`.
* Dialog `parties` lists the contributors; the receivers are the session's `parties`.
* Attachments need `party` plus `dialog` or `session`. The `dialog` default of 0 is gone. `add_lawful_basis_attachment()` and `add_wtf_transcription_attachment()` take `session_index`.
* `add_dialog()` and `add_incomplete_dialog()` return the new index and take `session=`. `add_incomplete_dialog()` writes no `body`. `add_analysis()` takes `dialog` as an optional keyword, plus `sessions`, `attachment` and `analysis`.
* `redacted_from`, `amended_from` and `redacted_strings`; `vcon.redacted` and `vcon.amended` are deprecated aliases.
* `is_valid()` checks sessions, events and every index reference, that `extensions` lists every extension in use and every `critical` entry, and that `vcon` is `"0.5.0"` when present. New `get_validation_warnings()` returns SHOULD-level findings.
* `sign(key, alg="RS256")`. `sign()` and `verify()` reject `alg: "none"` with `AlgorithmNotPermittedError`.

**0.10.0** retargeted the library to core-04: a `json`-encoded `body` is the JSON value itself, attachments got `party`, `dialog`, `start` and `mediatype` defaults, `Dialog.to_dict()` stopped writing empty `meta` and `metadata`, inline base64url bodies became unpadded (`b64url_encode()`, `b64url_decode()`), and `decode_body()` / `Vcon.decoded_body()` read either body shape.

**0.9.5** content hashes use the spec form `sha512-<unpadded base64url>`, and `compute_content_hash` and `parse_content_hash_algorithm` were added. Old unprefixed SHA-256 hashes still verify. **0.9.6** security: `requests` bump. **0.9.3** fixed `add_tag()` defaults. **0.9.2** `build_new()` emitted `"vcon": "0.4.0"` and `add_wtf_transcription_analysis()` was added.

### Known limits in 1.0.0

* `subject` has no setter.
* `add_wtf_transcription_analysis()` uses `type: "transcription"` and a string body (see above).
* `add_image()` writes an attachment with no `dialog` or `session`, so `is_valid()` then fails. Set the scope after the call, for example `vcon.attachments[-1]["session"] = 0`.
* `add_inline_data()` does not encode its body, and `add_image_data()` hashes the raw image bytes while `verify_content_hash()` recomputes over the encoded body string, so verification of an image dialog returns `False`.
* `Dialog.VALID_TYPES` still lists `audio` and `video`, which are accepted for backward compatibility and written as `recording`.

### Next

* [Library API reference](library-api-reference.md)
* [Guide for LLMs](vcon-library-guide-for-llms.md)
* [Adapter quick start](../vcon-adapters/quick-start-from-template.md)

MIT License.
