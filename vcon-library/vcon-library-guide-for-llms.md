---
description: A compact, self-contained reference for language models that write or edit Python code against the vcon 0.10.0 library, with the field names and mistakes that matter.
---

# 📜 vCon Library Guide for LLMs

Reference for models that generate code with the `vcon` Python library, version 0.10.0, which implements `draft-ietf-vcon-vcon-core-04` (`"vcon": "0.4.0"`). Every code block below was run against `vcon==0.10.0`. For field semantics see the [field reference](../vcons/field-reference.md). For the full method list see the [API reference](library-api-reference.md).

### Rules for generated code

1. Install with `pip install vcon` (Python 3.12 or later). Add `"vcon[image]"` for Pillow and pypdf.
2. Import only `Vcon`, `compute_content_hash` and `parse_content_hash_algorithm` from the package root. Use `from vcon.party import Party, PartyHistory`, `from vcon.dialog import Dialog`, `from vcon.civic_address import CivicAddress`. `from vcon import Party` fails.
3. Field names: `critical` (not `must_support`, not `must_understand`), `amended` (not `appended`), `purpose` on attachments (not `type`), `mediatype` (not `mimetype`), `schema` on analysis (not `schema_version`).
4. Attachments take `purpose=`. Analysis takes `type=`, `vendor=` and `dialog=`, all keyword-only.
5. The lawful basis attachment uses `purpose: "lawful_basis"`. Build it with `vcon.add_lawful_basis_attachment(...)`, never by hand, and add one to every complete vCon you create.
6. A `json`-encoded `body` is the JSON value itself (dict, list, string), not `json.dumps(...)`. Read any entry with `Vcon.decoded_body(entry)`.
7. `content_hash` is `sha512-` plus unpadded base64url. Create it with `compute_content_hash(data)` or `Dialog.calculate_content_hash()`.
8. Put WTF transcripts in `analysis[]` with `add_wtf_transcription_analysis()`, not in `attachments[]`.
9. Give every dialog a `mediatype`; `is_valid()` fails without one.
10. `Vcon` is not a dict. There is no `vcon.get(...)` and no `vcon["parties"]`. Use properties (`vcon.parties`) or `vcon.to_dict()` or `vcon.vcon_dict`.
11. Dispositions are lowercase and hyphenated: `no-answer`, `congestion`, `failed`, `busy`, `hung-up`, `voicemail-no-message`. `"NO_ANSWER"` raises `ValueError`.
12. Do not invent a lawful basis. If the source data has no consent record, say so and omit the attachment.

### Minimal complete vCon

```python
from datetime import datetime, timedelta, timezone

from vcon import Vcon
from vcon.dialog import Dialog
from vcon.party import Party

now = datetime.now(timezone.utc)

vcon = Vcon.build_new()
vcon.add_party(Party(tel="+15551230001", name="Alice", role="caller"))
vcon.add_party(Party(tel="+15551230002", name="Bob", role="agent"))
vcon.add_dialog(Dialog(type="text", start=now, parties=[0, 1],
                       body="Hello, I need help.", encoding="none",
                       mediatype="text/plain"))
vcon.add_lawful_basis_attachment(
    lawful_basis="consent",
    expiration=(now + timedelta(days=365)).isoformat(),
    purpose_grants=[{"purpose": "recording", "granted": True,
                     "granted_at": now.isoformat()}],
    party_index=0,
)
ok, errors = vcon.is_valid()
assert ok, errors
vcon.save_to_file("conversation.vcon.json")
```

### Vcon

| Task | Call |
|---|---|
| New | `Vcon.build_new()` |
| Load | `Vcon.load(path_or_url)`, `Vcon.load_from_file(path)`, `Vcon.load_from_url(url)`, `Vcon.build_from_json(str)` |
| Save, serialize | `save_to_file(path)`, `to_json()`, `to_dict()`, `post_to_url(url, headers=None)` |
| Validate | `is_valid() -> (bool, list)`, `Vcon.validate_file(path)`, `Vcon.validate_json(str)` |
| Sign | `Vcon.generate_key_pair()`, `sign(private_key)`, `verify(public_key) -> bool` |
| Read | properties `uuid`, `vcon`, `subject`, `created_at`, `updated_at`, `parties` (list of dicts), `dialog`, `attachments`, `analysis`, `redacted`, `amended`, `group`, `meta`, `tags` |
| Party objects | `get_party_objects()` returns `Party` instances; `find_party_index(by, val)` |

`subject` is read-only: set it with `vcon.vcon_dict["subject"] = "..."`. `Vcon.load(..., property_handling="strict" | "meta" | "default")` removes, relocates or keeps unknown properties.

### Dialogs

```python
from vcon.party import PartyHistory

vcon.add_dialog(Dialog(
    type="recording", start=now, parties=[0, 1],
    url="https://example.com/recording.wav",   # placeholder URL
    mediatype="audio/x-wav", duration=62.5,
    party_history=[PartyHistory(0, "join", now), PartyHistory(1, "join", now)],
))
vcon.add_incomplete_dialog(start=now, disposition="no-answer", parties=[0])
vcon.add_transfer_dialog(start=now, parties=[0, 1],
                         transfer_data={"transferee": 0, "transferor": 1, "transfer_target": 1},
                         metadata={"reason": "escalation"})
print(len(vcon.dialog))
```

* `Dialog.type` is one of `recording`, `text`, `transfer`, `incomplete`, `audio`, `video`; other values raise `ValueError`.
* Extra keyword arguments are accepted silently, so a wrong name such as `mimetype=` ends up as a stray property. Check spelling.
* `add_incomplete_dialog(start, disposition, details=None, parties=None, metadata=None)`.
* `add_transfer_dialog(start, transfer_data, parties, metadata=None)`.
* `add_external_data(url, filename, mediatype)` downloads the URL and hashes it. `add_inline_data(body, filename, mediatype)` does not encode: pass a body that is already base64url. `add_image_data(path, mediatype=None)` needs `vcon[image]`. `add_video_data(...)` needs `ffmpeg`.
* Type tests: `is_text()`, `is_recording()`, `is_audio()`, `is_video()`, `is_transfer()`, `is_incomplete()`, `is_image()`, `is_pdf()`, `is_email()`, `is_external_data()`, `is_inline_data()`.
* Search: `vcon.find_dialogs_by_type("text")`, `vcon.find_dialog("type", "text")`.

### Attachments, analysis, tags

```python
vcon.add_attachment(purpose="notes", body={"summary": "Account verified."}, encoding="json")
vcon.add_analysis(type="sentiment", dialog=0, vendor="example-vendor",
                  body={"sentiment": "positive", "confidence": 0.9}, encoding="json")
vcon.add_tag("department", "billing")

print(vcon.find_attachment_by_purpose("notes")["party"])         # 0, default
print(Vcon.decoded_body(vcon.find_analysis_by_type("sentiment")))
print(vcon.get_tag("department"))
```

```text
add_attachment(purpose, body=None, encoding="none", mediatype=None, filename=None,
               url=None, content_hash=None, start=None, party=None, dialog=None)
add_image(image_path, purpose="image")        Attachment.from_image(path, purpose="image")
add_analysis(*, type, dialog, vendor, body, encoding="none", mediatype=None, filename=None,
             product=None, url=None, content_hash=None, schema=None, meta=None, **extra)
```

* `encoding` is `json`, `base64url` or `none`.
* Defaults on every attachment: `party` 0, `dialog` 0, `start` from `created_at`, `mediatype` `application/json` for `json` bodies.
* Find helpers: `find_attachment_by_purpose(purpose)` (not `find_attachment_by_type`), `find_analysis_by_type(type)`.

### Extensions and critical

```python
vcon.add_extension("x-example")
vcon.add_critical("x-example")
print(vcon.get_extensions(), vcon.get_critical())
vcon.remove_critical("x-example")
vcon.remove_extension("x-example")
```

There are no `must_support` methods, no `vcon.extensions` or `vcon.must_support` properties and no `appended` property. Use `get_extensions()`, `get_critical()` and `amended`.

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
    proof_mechanisms=[ProofMechanism(ProofType.VERBAL_CONFIRMATION, now,
                                     {"recording_ref": "call-0001"})],
)
print(vcon.check_lawful_basis_permission("analysis", party_index=1))
print(len(vcon.find_lawful_basis_attachments(party_index=1)))
```

* Body shape: `lawful_basis`, `expiration` (may be null), `purpose_grants[]` (`purpose`, `granted`, `granted_at`, optional `conditions`), optional `proof_mechanisms[]` (`proof_type`, `timestamp`, `proof_data`).
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
* `add_wtf_transcription_analysis()` writes `type: "transcription"` and a JSON-string body. The extension draft defines `wtf_transcription`. This is an upstream gap in 0.10.0. For draft-conformant output call `vcon.add_analysis(type="wtf_transcription", dialog=0, vendor=..., product=..., body=wtf_dict, encoding="json")`.
* `add_wtf_transcription_attachment(...)` (same arguments with `party_index`, `dialog_index`) writes to `attachments[]`; `find_wtf_attachments()` searches only there.
* Provider conversion: `WhisperAdapter().convert(data)`, `DeepgramAdapter`, `AssemblyAIAdapter` return a `WTFAttachment` with `.transcript`, `.segments`, `.metadata`, `export_to_srt()`, `export_to_vtt()`, `get_speaking_time()`, `find_low_confidence_segments(threshold=0.5)`, `extract_keywords(min_confidence=0.8)`.
* `WTFExtension` (dict in, dict out): `create_wtf_attachment`, `convert_from_provider(data, provider)`, `validate_wtf_attachment`, `export_transcription(att, format)`, `analyze_transcription`, `compare_transcriptions`. `WTFProcessor` has the same methods over `WTFAttachment` objects.

### Content hashes and encoding

```python
from vcon import compute_content_hash, parse_content_hash_algorithm
from vcon.dialog import b64url_decode, b64url_encode

h = compute_content_hash(b"hello")               # sha512-... unpadded base64url
print(parse_content_hash_algorithm(h))           # sha512
print(compute_content_hash(b"hello", "sha256")[:12])
print(b64url_encode(b"ab"), b64url_decode("YWI"))
d = Dialog(type="text", start=now, parties=[0], body="hello",
           encoding="none", mediatype="text/plain")
print(d.verify_content_hash(d.calculate_content_hash()))
```

`Dialog.verify_content_hash()` recomputes over the stored `body` string, so for base64url image dialogs from `add_image_data()` (hash taken over raw bytes) it returns `False`. That is an upstream gap in 0.10.0.

### Other public API

```text
Vcon.decoded_body(entry) -> Any                       also vcon.body.decode_body(entry)
vcon.get_party_objects() -> list[Party]
vcon.get_critical() / add_critical(name) / remove_critical(name)
vcon.validate_extensions() -> dict      vcon.process_extensions() -> dict
vcon.compute_content_hash(data: bytes, algorithm="sha512") -> str
vcon.parse_content_hash_algorithm(content_hash) -> str | None
vcon.dialog.b64url_encode(data: bytes) -> str          b64url_decode(value) -> bytes
WTFProcessor.process(vcon_dict) / can_process(name) / convert_from_provider(data, provider)
WTFExtension.create_wtf_attachment(transcript, segments, metadata, **kwargs)
RegistryManager.submit_to_registry(lawful_basis, registry_type) -> str
RegistryValidator.validate_scitt_receipt(receipt)
```

### Errors

```python
for call in (
    lambda: Dialog(type="bogus", start=now, parties=[0]),
    lambda: vcon.add_incomplete_dialog(now, "NO_ANSWER", parties=[0]),
    lambda: vcon.add_attachment("x", body="a", encoding="bogus"),
    lambda: Vcon.load_from_file("/nonexistent.vcon.json"),
):
    try:
        call()
    except (ValueError, FileNotFoundError) as e:
        print(type(e).__name__)
```

* Invalid `type`, `disposition`, `encoding`, party history event or lawful basis type raises `ValueError`.
* A missing file raises `FileNotFoundError`; malformed JSON raises `json.JSONDecodeError`.
* `is_valid()` returns problems as strings and does not raise.
* `add_party()` needs a `Party`, not a dict (`AttributeError`).

### Common mistakes

| Wrong | Right |
|---|---|
| `from vcon import Party, Dialog` | `from vcon.party import Party`, `from vcon.dialog import Dialog` |
| `add_attachment(type="notes", ...)` | `add_attachment(purpose="notes", ...)` |
| `find_attachment_by_type(...)` | `find_attachment_by_purpose(...)` |
| `Dialog(..., mimetype="audio/wav")` | `Dialog(..., mediatype="audio/wav")` |
| `vcon.add_must_support("x")` | `vcon.add_critical("x")` |
| `vcon.must_support`, `vcon.extensions` | `vcon.get_critical()`, `vcon.get_extensions()` |
| `vcon.appended` | `vcon.amended` |
| `vcon.get("parties")` | `vcon.parties` or `vcon.to_dict().get("parties")` |
| `body=json.dumps(obj), encoding="json"` | `body=obj, encoding="json"` |
| `disposition="NO_ANSWER"` | `disposition="no-answer"` |
| `{"type": "lawful_basis", ...}` attachment | `purpose: "lawful_basis"` (not `type`) via `add_lawful_basis_attachment()` |
| WTF in `attachments[]` | `add_wtf_transcription_analysis()` |
| `Metadata(audio_quality=...)` | `Quality(audio_quality=...)` |
| `sha256` hash without prefix | `sha512-<base64url>` from `compute_content_hash()` |
