---
description: A compact, self-contained reference for language models that write or edit Python code against the vcon 1.0.0 library, with the field names and mistakes that matter.
---

# 📜 vCon Library Guide for LLMs

Reference for models that generate code with the `vcon` Python library, version 1.0.0, which implements `draft-ietf-vcon-vcon-core-05` (`"vcon": "0.5.0"`). Every code block below was run against `vcon==1.0.0`. For field semantics see the [field reference](../vcons/field-reference.md), which describes core-04; where it differs, the rules on this page are what the library writes. For the full method list see the [API reference](library-api-reference.md).

### Rules for generated code

1. Install with `pip install vcon` (Python 3.12 or later). Add `"vcon[image]"` for Pillow and pypdf.
2. Import `Vcon`, `Dialog`, `Session`, `SessionReference`, `Event`, `migrate_0_4_0_to_0_5_0`, `MigrationError`, `AlgorithmNotPermittedError`, `compute_content_hash` and `parse_content_hash_algorithm` from the package root. Use `from vcon.party import Party` and `from vcon.civic_address import CivicAddress`. `from vcon import Party` fails.
3. Every vCon with dialog or events needs at least one session. Add it with `vcon.add_session(Session(start=..., parties=[...]))` and add dialogs with `vcon.add_dialog(dialog, session=0)`.
4. Dialog `parties` lists the contributors only. The receivers go in the session's `parties`; the first session party is the originator.
5. Dialog `type` is `recording`, `text` or `incomplete`. There is no `transfer` or `recording-set` dialog, and no `party_history`, `originator` or transfer parameters on `Dialog`. Joins, drops, holds, key presses and transfers are `Event` objects added with `vcon.add_event(...)`.
6. Field names: `critical` (not `must_support`, not `must_understand`), `redacted_from` and `amended_from` (not `redacted`, `amended` or `appended`), `purpose` on attachments (not `type`), `mediatype` (not `mimetype`), `schema` on analysis (not `schema_version`).
7. Attachments take `purpose=` and need `party` plus `dialog` or `session`. Pass `session=` explicitly when the vCon has more than one session. Analysis takes `type=`, `vendor=` and optional `dialog=`, `sessions=`, all keyword-only.
8. The lawful basis attachment uses `purpose: "lawful_basis"`. Build it with `vcon.add_lawful_basis_attachment(..., party_index=..., session_index=...)`, never by hand, and add one to every complete vCon you create.
9. A `json`-encoded `body` is the JSON value itself (dict, list, string), not `json.dumps(...)`. Read any entry with `Vcon.decoded_body(entry)`.
10. `content_hash` is `sha512-` plus unpadded base64url. Create it with `compute_content_hash(data)` or `Dialog.calculate_content_hash()`.
11. Put WTF transcripts in `analysis[]` with `add_wtf_transcription_analysis()`, not in `attachments[]`.
12. Give every dialog with a `body` a `mediatype`; `is_valid()` fails without one.
13. `Vcon` is not a dict. There is no `vcon.get(...)` and no `vcon["parties"]`. Use properties (`vcon.parties`, `vcon.sessions`, `vcon.events`) or `vcon.to_dict()` or `vcon.vcon_dict`.
14. Dispositions are lowercase and hyphenated: `no-answer`, `congestion`, `failed`, `busy`, `hung-up`, `voicemail-no-message`. `"NO_ANSWER"` raises `ValueError`.
15. List every extension you use in `extensions` (`add_extension()`); `is_valid()` fails otherwise. The library's helpers add `lawful_basis` and `wtf_transcription` for you.
16. Never sign with `alg="none"`; it raises `AlgorithmNotPermittedError`.
17. Do not invent a lawful basis. If the source data has no consent record, say so and omit the attachment.

### Minimal complete vCon

```python
from datetime import datetime, timedelta, timezone

from vcon import Dialog, Session, Vcon
from vcon.party import Party

now = datetime.now(timezone.utc)

vcon = Vcon.build_new()
vcon.add_party(Party(tel="+15551230001", name="Alice", role="caller"))
vcon.add_party(Party(tel="+15551230002", name="Bob", role="agent"))
vcon.add_session(Session(start=now, parties=[0, 1]))
vcon.add_dialog(Dialog(type="text", start=now, parties=[0],
                       body="Hello, I need help.", encoding="none",
                       mediatype="text/plain"), session=0)
vcon.add_lawful_basis_attachment(
    lawful_basis="consent",
    expiration=(now + timedelta(days=365)).isoformat(),
    purpose_grants=[{"purpose": "recording", "granted": True,
                     "granted_at": now.isoformat()}],
    party_index=0,
    session_index=0,
)
ok, errors = vcon.is_valid()
assert ok, errors
vcon.save_to_file("conversation.vcon.json")
```

### Vcon

| Task | Call |
|---|---|
| New | `Vcon.build_new()` (writes `"vcon": "0.5.0"`) |
| Load | `Vcon.load(path_or_url)`, `Vcon.load_from_file(path)`, `Vcon.load_from_url(url)`, `Vcon.build_from_json(str)`, `Vcon(dict)`. A 0.4.0 vCon is migrated. |
| Save, serialize | `save_to_file(path)`, `to_json()`, `to_dict()`, `post_to_url(url, headers=None)` |
| Validate | `is_valid() -> (bool, list)`, `get_validation_warnings() -> list`, `Vcon.validate_file(path)`, `Vcon.validate_json(str)` |
| Sign | `Vcon.generate_key_pair()`, `sign(private_key, alg="RS256")`, `verify(public_key) -> bool` |
| Read | properties `uuid`, `vcon`, `subject`, `created_at`, `updated_at`, `parties` (list of dicts), `sessions`, `dialog`, `events`, `attachments`, `analysis`, `redacted_from`, `amended_from`, `redacted_strings`, `group`, `meta`, `tags`, `migration_report` |
| Party objects | `get_party_objects()` returns `Party` instances; `find_party_index(by, val)` |

`subject` is read-only: set it with `vcon.vcon_dict["subject"] = "..."`. `Vcon.load(..., property_handling="strict" | "meta" | "default")` removes, relocates or keeps unknown properties. `vcon.redacted` and `vcon.amended` still work but emit `DeprecationWarning`.

### Sessions, dialogs and events

```python
from vcon import Event, SessionReference

vcon.add_dialog(Dialog(
    type="recording", start=now, parties=[0, 1],
    url="https://example.com/recording.wav",   # placeholder URL
    mediatype="audio/x-wav", duration=62.5,
), session=0)
vcon.add_event(Event(type="join", time=now, sessions=0, party=0))
vcon.add_event(Event(type="join", time=now, sessions=0, party=1))
vcon.add_incomplete_dialog(start=now, disposition="no-answer", parties=[0], session=0)

# Bob (1) transfers Alice (0) to Carol (2) without consulting: one session per call
vcon.add_party(Party(name="Carol", role="supervisor"))
target = vcon.add_session(Session(start=now, parties=[0, 2],
                                  sessions=[SessionReference(0, "precursor")]))
vcon.add_event(Event(type="blind-transfer", time=now, sessions=0, party=1,
                     transferee=0, transfer_target=2, target_session=target))
print(len(vcon.dialog), len(vcon.sessions), len(vcon.events))   # 3 2 3
print(vcon.is_valid())                                          # (True, [])
```

* `Session(start=None, duration=None, parties=None, originator=None, session_id=None, sessions=None, dialog=None)`. Set `originator` only when it is not the first party. `Session()` is an allowed empty `{}`.
* `SessionReference(session, type, description=None)`; `type` is `child`, `peer`, `precursor` or `breakout`.
* `add_session()`, `add_dialog(dialog, session=None)`, `add_event()` and `add_incomplete_dialog(..., session=None)` return the new index.
* `Event(type, time, sessions=None, party=None, dialog=None, channel=None, contributors=None, button=None, transferee=None, transfer_target=None, consultation=None, target_session=None)`. The constructor raises `ValueError` when a required parameter for the type is missing or a forbidden one is present.
* Event types: `join`, `drop`, `hold`, `unhold`, `mute`, `unmute` (need `sessions`, `party`); `keydown`, `keyup` (also `button`); `consultation-start`, `blind-transfer`, `consultative-transfer`, `transfer-abandoned` (need `sessions` as one index, `party`, `transferee`, and per type `transfer_target`, `consultation`, `target_session`); `channel-contributors` (needs `dialog` and `channel`, no `sessions` or `party`).
* `Dialog.type` is `recording`, `text` or `incomplete`. `audio` and `video` warn and become `recording`. `transfer` raises `ValueError`.
* `Dialog(party_history=...)`, `originator=`, `transferee=`, `transferor=`, `transfer_target=`, `original=`, `consultation=`, `target_dialog=`, `transfer=`, `recordings=` and `recording_set=` raise `ValueError`. `add_transfer_dialog()` raises `NotImplementedError`. `PartyHistory` is deprecated; `PartyHistory(...).to_event(sessions)` converts one.
* Other extra keyword arguments are accepted silently, so a wrong name such as `mimetype=` ends up as a stray property. Check spelling.
* `add_incomplete_dialog(start, disposition, details=None, parties=None, metadata=None, session=None)` writes no `body`; `details` goes to `metadata["details"]`.
* `Dialog.placeholder()` is an empty `{}` dialog that keeps indices stable.
* `add_external_data(url, filename, mediatype)` downloads the URL and hashes it. `add_inline_data(body, filename, mediatype)` does not encode: pass a body that is already base64url. `add_image_data(path, mediatype=None)` needs `vcon[image]`. `add_video_data(...)` needs `ffmpeg`.
* Type tests: `is_text()`, `is_recording()`, `is_incomplete()`, `is_audio()`, `is_video()` (by `mediatype`), `is_image()`, `is_pdf()`, `is_email()`, `is_external_data()`, `is_inline_data()`, `is_placeholder()`.
* Search: `vcon.find_dialogs_by_type("text")`, `vcon.find_dialog("type", "text")`.

### Attachments, analysis, tags

```python
vcon.add_attachment(purpose="notes", body={"summary": "Account verified."},
                    encoding="json", session=0)
vcon.add_analysis(type="sentiment", dialog=0, vendor="example-vendor",
                  body={"sentiment": "positive", "confidence": 0.9}, encoding="json")
vcon.add_tag("department", "billing")

print(vcon.find_attachment_by_purpose("notes")["party"])         # 0, default
print(Vcon.decoded_body(vcon.find_analysis_by_type("sentiment")))  # {'sentiment': 'positive', 'confidence': 0.9}
print(vcon.get_tag("department"))                                # billing
```

```text
add_attachment(purpose, body=None, encoding="none", mediatype=None, filename=None,
               url=None, content_hash=None, start=None, party=None, dialog=None, session=None)
add_image(image_path, purpose="image")        Attachment.from_image(path, purpose="image")
add_analysis(*, type, vendor, body, dialog=None, sessions=None, attachment=None, analysis=None,
             encoding="none", mediatype=None, filename=None, product=None, url=None,
             content_hash=None, schema=None, meta=None, **extra)
```

* `encoding` is `json`, `base64url` or `none`.
* Defaults on every attachment: `party` 0, `start` from `created_at`, `mediatype` `application/json` for `json` bodies. There is no `dialog` default. With neither `dialog` nor `session`, the only session is used if there is exactly one; otherwise `ValueError`.
* `add_tag()` scopes the tags attachment to every session and adds an empty session if there is none.
* `add_image()` sets no `dialog` or `session`; set one on `vcon.attachments[-1]` afterwards or `is_valid()` fails.
* Find helpers: `find_attachment_by_purpose(purpose)` (not `find_attachment_by_type`), `find_analysis_by_type(type)`.

### Extensions and critical

```python
vcon.add_extension("x-example")
vcon.add_critical("x-example")
print(vcon.get_extensions(), vcon.get_critical())   # ['lawful_basis', 'x-example'] ['x-example']
vcon.remove_critical("x-example")
vcon.remove_extension("x-example")
```

There are no `must_support` methods, no `vcon.extensions` or `vcon.must_support` properties and no `appended` property. Use `get_extensions()`, `get_critical()` and `amended_from`. Every `critical` entry must also be in `extensions`.

### Lawful basis

```python
from vcon.extensions.lawful_basis import ProofMechanism, ProofType

vcon.add_lawful_basis_attachment(
    lawful_basis="consent",            # consent, contract, legal_obligation, vital_interests,
                                       # public_task, legitimate_interests
    expiration=None,                   # or an ISO 8601 string
    purpose_grants=[
        {"purpose": "analysis", "granted": True, "granted_at": now.isoformat(),
         "conditions": ["anonymized_data_only"]},
    ],
    party_index=1,
    session_index=0,
    proof_mechanisms=[ProofMechanism(ProofType.VERBAL_CONFIRMATION, now,
                                     {"recording_ref": "call-0001"})],
)
print(vcon.check_lawful_basis_permission("analysis", party_index=1))   # True
print(len(vcon.find_lawful_basis_attachments(party_index=1)))         # 1
```

* Body shape: `lawful_basis`, `expiration` (may be null), `purpose_grants[]` (`purpose`, `granted`, `granted_at`, optional `conditions`), optional `proof_mechanisms[]` (`proof_type`, `timestamp`, `proof_data`).
* Scope with `session_index` (an index or list). `dialog_index` alone logs a warning; with neither, the only session is used or `ValueError` is raised. A missing `party_index` defaults to 0 with a warning.
* Extra keywords reach `LawfulBasisAttachment`: `terms_of_service`, `status_interval`, `content_hash`, `registry`, `proof_mechanisms`, `metadata`. `proof_mechanisms` takes `ProofMechanism` objects, not dicts.
* `ProofType`: `verbal_confirmation`, `signed_document`, `cryptographic_signature`, `external_system`.
* `HashAlgorithm` (for the lawful basis `ContentHash` only): `SHA_256`, `SHA_3_256`, `BLAKE2B_256`. This is not the dialog `content_hash`.
* Attestation registries: `SCITTRegistryClient(registry_url, auth_token=None)` and `RegistryManager()` in `vcon.extensions.lawful_basis.registry`; `RegistryValidator` has static `validate_scitt_receipt`, `validate_attestation_status`, `validate_registry_metadata`.

### WTF transcription

```python
vcon.add_wtf_transcription_analysis(
    transcript={"text": "Hello, I need help.", "language": "en", "duration": 3.0, "confidence": 0.93},
    segments=[{"id": 0, "start": 0.0, "end": 3.0, "text": "Hello, I need help.",
               "confidence": 0.93, "speaker": 0}],
    metadata={"created_at": now.isoformat(), "processed_at": now.isoformat(),
              "provider": "whisper", "model": "whisper-1"},
    dialog_index=0,
)
entry = vcon.find_analysis_by_type("transcription")
wtf = Vcon.decoded_body(entry)

from vcon.extensions.wtf import WTFAttachment
print(WTFAttachment.from_dict(wtf).export_to_srt())
```

* `metadata` needs `created_at`, `processed_at`, `provider`, `model`. `Metadata(audio_quality=...)` raises `TypeError`: `audio_quality` belongs to `Quality`.
* `add_wtf_transcription_analysis()` writes `type: "transcription"` and a JSON-string body. The extension draft defines `wtf_transcription`. This upstream gap is still present in 1.0.0. For draft-conformant output call `vcon.add_analysis(type="wtf_transcription", dialog=0, vendor=..., product=..., body=wtf_dict, encoding="json")` and `vcon.add_extension("wtf_transcription")`.
* `add_wtf_transcription_attachment(...)` (same arguments with `party_index`, `dialog_index`, `session_index`) writes to `attachments[]`; `find_wtf_attachments()` searches only there.
* Provider conversion: `WhisperAdapter().convert(data)`, `DeepgramAdapter`, `AssemblyAIAdapter` return a `WTFAttachment` with `.transcript`, `.segments`, `.metadata`, `export_to_srt()`, `export_to_vtt()`, `get_speaking_time()`, `find_low_confidence_segments(threshold=0.5)`, `extract_keywords(min_confidence=0.8)`.
* `WTFExtension` (dict in, dict out): `create_wtf_attachment`, `convert_from_provider(data, provider)`, `validate_wtf_attachment`, `export_transcription(att, format)`, `analyze_transcription`, `compare_transcriptions`. `WTFProcessor` has the same methods over `WTFAttachment` objects.

### Reading 0.4.0 vCons

```python
from vcon import MigrationError, migrate_0_4_0_to_0_5_0

old = {"vcon": "0.4.0", "uuid": "01928e10-193e-8231-b9a2-279e0d16bc46",
       "created_at": "2026-10-06T14:00:00+00:00",
       "parties": [{"name": "Alice"}, {"name": "Bob"}],
       "dialog": [{"type": "text", "start": "2026-10-06T14:00:00+00:00", "parties": [0, 1],
                   "body": "Hi", "encoding": "none", "mediatype": "text/plain"}]}
v = Vcon(old)                                   # also Vcon.load(...), build_from_json(...)
print(v.vcon, v.migration_report.summary())     # 0.5.0 1 inferred, 0 fallback, 0 unresolved
new_dict, report = migrate_0_4_0_to_0_5_0(old)  # the function form; strict=True raises on fallbacks
print(report.to_dict()["target_version"])       # 0.5.0
```

* Loading migrates any vCon that declares `"vcon": "0.4.0"`. Use `migrate_legacy=False` to keep it unchanged (it then fails `is_valid()`).
* The report lists every inferred value (`report.inferred`), fallback (`report.fallbacks`) and unresolved item (`report.unresolved`), and keeps removed dialogs in `report.removed_dialogs`. Store `report.to_dict()`.
* Removed `transfer` and `recording-set` dialogs become `{}` in place, so indices stay valid.
* A signed 0.4.0 vCon is not migrated on load. Verify it, call `Vcon.migrate_0_4_0_to_0_5_0(signed)`, then sign again.
* Migrating a 0.5.0 vCon raises `MigrationError`.

### Content hashes and encoding

```python
from vcon import compute_content_hash, parse_content_hash_algorithm
from vcon.dialog import b64url_decode, b64url_encode

h = compute_content_hash(b"hello")               # sha512-... unpadded base64url
print(parse_content_hash_algorithm(h))           # sha512
print(compute_content_hash(b"hello", "sha256")[:12])   # sha256-LPJNu
print(b64url_encode(b"ab"), b64url_decode("YWI"))      # YWI b'ab'
d = Dialog(type="text", start=now, parties=[0], body="hello",
           encoding="none", mediatype="text/plain")
print(d.verify_content_hash(d.calculate_content_hash()))   # True
```

`Dialog.verify_content_hash()` recomputes over the stored `body` string, so for base64url image dialogs from `add_image_data()` (hash taken over raw bytes) it returns `False`. That upstream gap is still present in 1.0.0.

### Other public API

```text
Vcon.decoded_body(entry) -> Any                       also vcon.body.decode_body(entry)
vcon.get_party_objects() -> list[Party]
vcon.get_critical() / add_critical(name) / remove_critical(name)
vcon.get_validation_warnings() -> list[str]
vcon.validate_extensions() -> dict      vcon.process_extensions() -> dict
vcon.compute_content_hash(data: bytes, algorithm="sha512") -> str
vcon.parse_content_hash_algorithm(content_hash) -> str | None
vcon.dialog.b64url_encode(data: bytes) -> str          b64url_decode(value) -> bytes
vcon.event.validate_event(event_dict) -> (errors, warnings)
vcon.session.validate_session(session_dict) -> (errors, warnings)
WTFProcessor.process(vcon_dict) / can_process(name) / convert_from_provider(data, provider)
WTFExtension.create_wtf_attachment(transcript, segments, metadata, **kwargs)
RegistryManager.submit_to_registry(lawful_basis, registry_type) -> str
RegistryValidator.validate_scitt_receipt(receipt)
```

### Errors

```python
from vcon import AlgorithmNotPermittedError

private_key, _ = Vcon.generate_key_pair()
for call in (
    lambda: Dialog(type="transfer", start=now, parties=[0]),
    lambda: Dialog(type="recording", start=now, parties=[0], originator=0),
    lambda: vcon.add_incomplete_dialog(now, "NO_ANSWER", parties=[0]),
    lambda: vcon.add_attachment("x", body="a", encoding="bogus", session=0),
    lambda: vcon.add_attachment("x", body="a"),            # two sessions, no scope
    lambda: Event(type="join", time=now, party=0),         # no sessions
    lambda: Vcon.build_new().sign(private_key, alg="none"),
    lambda: Vcon.load_from_file("/nonexistent.vcon.json"),
):
    try:
        call()
    except (ValueError, FileNotFoundError) as e:
        print(type(e).__name__)
```

* Invalid dialog `type`, removed dialog parameter, `disposition`, `encoding`, event parameters, attachment scope or lawful basis type raises `ValueError`. `AlgorithmNotPermittedError` and `MigrationError` are `ValueError` subclasses.
* `add_transfer_dialog()` raises `NotImplementedError`.
* A missing file raises `FileNotFoundError`; malformed JSON raises `json.JSONDecodeError`.
* `is_valid()` returns problems as strings and does not raise. `get_validation_warnings()` returns SHOULD-level findings.
* `add_party()` needs a `Party`, not a dict (`AttributeError`).

### Common mistakes

| Wrong | Right |
|---|---|
| `from vcon import Party` | `from vcon.party import Party` (`Dialog`, `Session` and `Event` do come from `vcon`) |
| Dialogs with no session | `vcon.add_session(Session(...))`, then `add_dialog(d, session=0)` |
| `Dialog(parties=[sender, receiver])` for a text message | `Dialog(parties=[sender])`; receivers in `Session(parties=[...])` |
| `Dialog(type="transfer", ...)`, `add_transfer_dialog()` | `Event(type="blind-transfer", ...)` and a session per call |
| `Dialog(party_history=[PartyHistory(...)])` | `vcon.add_event(Event(type="join", time=..., sessions=0, party=0))` |
| `Dialog(originator=1)` | `Session(originator=1)` (only when it is not the first party) |
| `Dialog(type="audio")` or `"video"` | `Dialog(type="recording", mediatype="audio/...")` |
| `add_attachment(type="notes", ...)` | `add_attachment(purpose="notes", ..., session=0)` |
| `find_attachment_by_type(...)` | `find_attachment_by_purpose(...)` |
| `Dialog(..., mimetype="audio/wav")` | `Dialog(..., mediatype="audio/wav")` |
| `vcon.add_must_support("x")` | `vcon.add_critical("x")` |
| `vcon.must_support`, `vcon.extensions` | `vcon.get_critical()`, `vcon.get_extensions()` |
| `vcon.appended`, `vcon.amended`, `vcon.redacted` | `vcon.amended_from`, `vcon.redacted_from` |
| `vcon.get("parties")` | `vcon.parties` or `vcon.to_dict().get("parties")` |
| `body=json.dumps(obj), encoding="json"` | `body=obj, encoding="json"` |
| `disposition="NO_ANSWER"` | `disposition="no-answer"` |
| `{"type": "lawful_basis", ...}` attachment | `purpose: "lawful_basis"` (not `type`) via `add_lawful_basis_attachment()` |
| WTF in `attachments[]` | `add_wtf_transcription_analysis()` |
| `Metadata(audio_quality=...)` | `Quality(audio_quality=...)` |
| `sha256` hash without prefix | `sha512-<base64url>` from `compute_content_hash()` |
| `sign(key, alg="none")` | `sign(key)` (RS256) |
