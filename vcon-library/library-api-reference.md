---
description: Signatures, return types and working examples for every public class and function in the vcon 0.10.0 Python library.
---

# 🔌 Library API Reference

## vCon Library API Reference

Reference for `vcon` **0.10.0**, which implements [`draft-ietf-vcon-vcon-core-04`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) (`"vcon": "0.4.0"`). The [quickstart](quickstart.md) has release notes and known limits. What each field means, and when it is required, is in the [field reference](../vcons/field-reference.md).

### Installation

```bash
pip install vcon
pip install "vcon[image]"    # Pillow and pypdf for image and PDF helpers
```

Python 3.12 or later. Video helpers need the `ffmpeg` binary.

### Imports

The package root exports three names. Everything else comes from its module.

```python
from vcon import Vcon, compute_content_hash, parse_content_hash_algorithm
from vcon.party import Party, PartyHistory
from vcon.dialog import Dialog, b64url_encode, b64url_decode
from vcon.civic_address import CivicAddress
from vcon.vcon import Attachment
from vcon.body import decode_body
from vcon.extensions import (
    ExtensionType, ExtensionRegistry, get_extension_registry,
    LawfulBasisExtension, WTFExtension,
)
from vcon.extensions.lawful_basis import (
    LawfulBasisAttachment, PurposeGrant, ProofMechanism, ContentHash, RegistryInfo,
    LawfulBasisType, ProofType, HashAlgorithm, CanonicalizationMethod,
    LawfulBasisValidator, LawfulBasisProcessor, SCITTRegistryClient,
)
from vcon.extensions.lawful_basis.registry import RegistryManager
from vcon.extensions.wtf import (
    WTFAttachment, Transcript, Segment, Word, Speaker, Quality, Metadata,
    WTFProvider, WTFValidator, WTFProcessor,
    WhisperAdapter, DeepgramAdapter, AssemblyAIAdapter, ProviderAdapter,
)
```

`from vcon import Party` and `from vcon import Dialog` raise `ImportError`.

### Vcon

#### Construction and I/O

| Method | Returns | Notes |
|---|---|---|
| `Vcon(vcon_dict=None, property_handling="default")` | `Vcon` | Wraps an existing dict. |
| `Vcon.build_new(created_at=None, property_handling="default")` | `Vcon` | New vCon with `vcon: "0.4.0"`, a UUID8 and `created_at`. No empty `group` or `redacted`. |
| `Vcon.build_from_json(json_string, property_handling="default")` | `Vcon` | |
| `Vcon.load(source, property_handling="default")` | `Vcon` | Path or URL. |
| `Vcon.load_from_file(file_path, ...)` / `Vcon.load_from_url(url, ...)` | `Vcon` | `FileNotFoundError` for a missing file. |
| `save_to_file(file_path)` | `None` | |
| `post_to_url(url, headers=None)` | `requests.Response` | |
| `to_json()` / `dumps()` | `str` | |
| `to_dict()` | `dict` | |
| `is_valid()` | `(bool, list[str])` | |
| `Vcon.validate_file(file_path)` / `Vcon.validate_json(json_str)` | `(bool, list[str])` | |
| `sign(private_key)` / `verify(public_key)` | `None` / `bool` | JWS. After signing, `to_dict()` holds `payload` and `signatures`. |
| `Vcon.generate_key_pair()` | `(private_key, public_key)` | RSA. |
| `Vcon.uuid8_domain_name(domain)` / `Vcon.uuid8_time(custom_c_62_bits)` | `str` | UUID8 helpers. |
| `set_created_at(ts)` / `set_updated_at(ts)` | `None` | `str` or `datetime`. |

`property_handling` is `"default"` (keep unknown properties), `"strict"` (remove them) or `"meta"` (move them into `meta`). The constants are `vcon.vcon.PROPERTY_HANDLING_DEFAULT`, `_STRICT` and `_META`.

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
                       body="Hello", encoding="none", mediatype="text/plain"))
print(vcon.vcon, vcon.uuid, vcon.is_valid())
```

#### Properties

Read-only: `vcon`, `uuid`, `subject`, `created_at`, `updated_at`, `parties` (a `list[dict]`), `dialog`, `attachments`, `analysis`, `redacted`, `amended`, `group`, `meta`, `tags`. There is no setter for `subject`; write `vcon.vcon_dict["subject"] = "..."`. The instance holds its data in `vcon.vcon_dict`.

There is no `appended` (use `amended`), no `must_support` or `must_understand`, and no `extensions` or `critical` property. Use the methods below.

#### Parties and dialogs

| Method | Notes |
|---|---|
| `add_party(party: Party)` | |
| `get_party_objects() -> list[Party]` | `parties` returns dicts, this returns `Party` objects. |
| `find_party_index(by, val) -> int or None` | `by` is a party attribute such as `"name"` or `"tel"`. |
| `add_dialog(dialog: Dialog)` | |
| `find_dialog(by, val) -> Dialog or None` | |
| `find_dialogs_by_type(type) -> list[dict]` | |
| `add_incomplete_dialog(start, disposition, details=None, parties=None, metadata=None)` | `disposition` is lowercase and hyphenated, see constants. Raises `ValueError` otherwise. |
| `add_transfer_dialog(start, transfer_data, parties, metadata=None)` | `transfer_data` holds `transferee`, `transferor`, `transfer_target`, and optionally `original`, `consultation`, `target_dialog`. |

#### Attachments and analysis

```text
add_attachment(purpose, body=None, encoding="none", mediatype=None, filename=None,
               url=None, content_hash=None, start=None, party=None, dialog=None) -> Attachment
add_image(image_path, purpose="image") -> Attachment
find_attachment_by_purpose(purpose) -> dict | None
add_analysis(*, type, dialog, vendor, body, encoding="none", mediatype=None, filename=None,
             product=None, url=None, content_hash=None, schema=None, meta=None, **extra) -> None
find_analysis_by_type(type) -> dict | None
```

`encoding` is `"json"`, `"base64url"` or `"none"`; anything else raises `ValueError`. With `encoding="json"`, `body` is the JSON value itself. `party` and `dialog` default to `0`, `start` to the vCon's `created_at`, and `mediatype` to `application/json` for JSON bodies. `add_analysis` requires `vendor` and keyword arguments only. `type=` is analysis only; attachments use `purpose=`.

```python
vcon.add_attachment(purpose="notes", body={"agent_note": "Verified."}, encoding="json")
vcon.add_analysis(type="sentiment", dialog=0, vendor="example-vendor",
                  body={"sentiment": "positive", "confidence": 0.95}, encoding="json")
print(vcon.find_attachment_by_purpose("notes"))
print(Vcon.decoded_body(vcon.find_analysis_by_type("sentiment")))
```

`Vcon.decoded_body(entry)` is a static method, also available as `vcon.body.decode_body(entry)`. It returns the decoded value of an attachment, analysis or dialog entry and accepts both the core-04 form and a stringified JSON body written by older code.

#### Tags

`add_tag(tag_name, tag_value)`, `get_tag(tag_name) -> str | None`, and the `tags` property (the tags attachment). Tags are stored as a list of `"name:value"` strings in a `purpose: "tags"` attachment.

#### Extensions and critical

| Method | Returns |
|---|---|
| `add_extension(name)` / `remove_extension(name)` | `None` |
| `get_extensions()` | `list[str]` |
| `add_critical(name)` / `remove_critical(name)` | `None` |
| `get_critical()` | `list[str]` |

`critical` lists extensions a receiver must understand to process the vCon. See the [field reference](../vcons/field-reference.md).

```python
vcon.add_extension("x-example")
vcon.add_critical("x-example")
print(vcon.get_extensions(), vcon.get_critical())
vcon.remove_critical("x-example")
vcon.remove_extension("x-example")
```

#### Extension helpers on Vcon

| Method | Notes |
|---|---|
| `add_lawful_basis_attachment(lawful_basis, expiration, purpose_grants, party_index=None, dialog_index=None, **kwargs)` | `purpose_grants` is a list of dicts. Extra keywords go to `LawfulBasisAttachment` (`terms_of_service`, `status_interval`, `content_hash`, `registry`, `proof_mechanisms`, `metadata`). Writes `purpose: "lawful_basis"` and adds the `lawful_basis` extension. `expiration` may be `None`. |
| `find_lawful_basis_attachments(party_index=None)` | |
| `check_lawful_basis_permission(purpose, party_index=None) -> bool` | |
| `add_wtf_transcription_analysis(transcript, segments, metadata, dialog_index=None, **kwargs)` | Writes to `analysis[]` and adds the `wtf_transcription` extension. Preferred. |
| `add_wtf_transcription_attachment(transcript, segments, metadata, party_index=None, dialog_index=None, **kwargs)` | Writes to `attachments[]`. |
| `find_wtf_attachments(party_index=None)` | Searches `attachments[]` only. |
| `validate_extensions() -> dict` | Keyed by extension name, plus `attachments`. |
| `process_extensions() -> dict` | Name to `ProcessingResult`. |

`add_wtf_transcription_analysis()` writes `type: "transcription"`, while [`draft-howe-vcon-wtf-extension`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/) defines `wtf_transcription`, and it stores the body as a JSON string. Both are upstream gaps in 0.10.0. To write the draft values, call `add_analysis(type="wtf_transcription", ..., body=wtf_dict, encoding="json")`.

### Party

```text
Party(tel=None, stir=None, mailto=None, name=None, validation=None, gmlpos=None,
      civicaddress=None, uuid=None, role=None, contact_list=None, meta=None,
      sip=None, did=None, jCard=None, timezone=None, **kwargs)
```

`to_dict()` omits unset fields. `PartyHistory(party: int, event: str, time: datetime)` records `join`, `drop`, `hold`, `unhold`, `mute` or `unmute` and raises `ValueError` for other events.

`CivicAddress(country, a1..a6, prd, pod, sts, hno, hns, lmk, loc, flr, nam, pc)` holds GEOPRIV fields; pass it as `civicaddress=`.

```python
from datetime import datetime, timezone
from vcon.civic_address import CivicAddress
from vcon.party import Party, PartyHistory

p = Party(name="Jane", sip="sip:jane@example.com", timezone="America/New_York",
          civicaddress=CivicAddress(country="US", a1="CA", a3="San Francisco", pc="94102"))
print(p.to_dict())
print(PartyHistory(0, "join", datetime.now(timezone.utc)).to_dict())
```

### Dialog

```text
Dialog(type, start, parties, originator=None, mediatype=None, filename=None, body=None,
       encoding=None, url=None, disposition=None, party_history=None, transferee=None,
       transferor=None, transfer_target=None, original=None, consultation=None,
       target_dialog=None, campaign=None, interaction=None, skill=None, duration=None,
       meta=None, metadata=None, transfer=None, signaling=None, resolution=None,
       frame_rate=None, codec=None, bitrate=None, thumbnail=None, session_id=None,
       content_hash=None, application=None, message_id=None, **kwargs)
```

`type` must be `recording`, `text`, `transfer`, `incomplete`, `audio` or `video`; other values raise `ValueError`. Use `mediatype`. A `mimetype=` keyword is not an error: it falls into `**kwargs` and is written as a stray `mimetype` property. Dialogs without a `mediatype` fail `is_valid()`.

| Method | Notes |
|---|---|
| `to_dict()` | Omits empty `meta` and `metadata`. |
| `add_external_data(url, filename, mediatype)` | Downloads the URL, sets `url` and a `sha512-` hash. Raises if the fetch fails. |
| `add_inline_data(body, filename, mediatype)` | Sets `encoding="base64url"` and the hash. Does not encode: pass an already base64url body. |
| `add_image_data(image_path, mediatype=None)`, `extract_image_metadata(image_data, mediatype)` | Needs `vcon[image]`. Body is unpadded base64url. |
| `add_video_data(video_data, filename=None, mediatype=None, inline=True, metadata=None)` | Needs `ffmpeg` for metadata. |
| `add_video_with_optimal_storage(video_data, filename, mediatype=None, size_threshold_mb=10)` | |
| `add_streaming_video_reference(reference_id, mediatype, metadata=None)` | |
| `extract_video_metadata(video_path=None)`, `generate_thumbnail(max_size=(200, 200))`, `transcode_video(target_format, codec=None, bit_rate=None, width=None, height=None)` | |
| `to_inline_data()` | Fetches an external body and inlines it as unpadded base64url. |
| `is_external_data()`, `is_inline_data()`, `is_text()`, `is_recording()`, `is_transfer()`, `is_incomplete()`, `is_audio()`, `is_video()`, `is_email()`, `is_image()`, `is_pdf()`, `has_thumbnail()` | `bool` |
| `set_session_id(session_id)` / `get_session_id()` | Dict or list of dicts. |
| `set_content_hash(h)` / `get_content_hash()` | |
| `calculate_content_hash(algorithm="sha512") -> str` | Hashes the stored `body` string. Raises `ValueError` with no body. |
| `verify_content_hash(expected_hash, algorithm=None) -> bool` | Reads the algorithm from the prefix of `expected_hash`. A legacy unprefixed SHA-256 value still verifies. |
| `is_external_data_changed() -> bool` | |

Known gap: `add_image_data()` hashes the raw image bytes, but `verify_content_hash()` recomputes over the encoded body, so it returns `False` for image dialogs.

```python
from vcon import compute_content_hash, parse_content_hash_algorithm
from vcon.dialog import Dialog, b64url_decode, b64url_encode

d = Dialog(type="text", start=now, parties=[0], body="hello",
           encoding="none", mediatype="text/plain")
h = d.calculate_content_hash()
print(h, parse_content_hash_algorithm(h), d.verify_content_hash(h))
print(compute_content_hash(b"hello", "sha256"))
print(b64url_encode(b"ab"), b64url_decode("YWI"), b64url_decode("YWI="))
```

### Hash and encoding helpers

| Function | Notes |
|---|---|
| `vcon.compute_content_hash(data: bytes, algorithm="sha512") -> str` | Returns `"<algorithm>-<unpadded base64url>"`. `sha512` and `sha256` only. |
| `vcon.parse_content_hash_algorithm(content_hash) -> str \| None` | `None` for a legacy unprefixed hash. |
| `vcon.dialog.b64url_encode(data: bytes) -> str` | No padding. |
| `vcon.dialog.b64url_decode(value: str \| bytes) -> bytes` | Accepts padded and unpadded input. |
| `vcon.body.decode_body(entry) -> Any` | Same as `Vcon.decoded_body`. |

### Attachment

`Attachment(purpose, body=None, encoding="none", mediatype=None, filename=None, url=None, content_hash=None, start=None, party=None, dialog=None)` applies the same defaults as `Vcon.add_attachment()` (`party` and `dialog` 0, JSON `mediatype`). `Attachment.from_image(image_path, purpose="image")` builds one from a file. `to_dict()` serializes. `Attachment.VALID_ENCODINGS` is `["base64url", "json", "none"]`.

### Extension framework

`vcon.extensions` holds the registry and base types.

| Name | Purpose |
|---|---|
| `get_extension_registry() -> ExtensionRegistry` | Global registry with `lawful_basis` and `wtf_transcription` registered. |
| `ExtensionRegistry` | `register_extension(info)`, `get_extension(name)`, `list_extensions()`, `is_extension_registered(name)`, `validate_extension(name, vcon_dict)`, `validate_attachment(attachment)`, `process_extensions(vcon_dict)`, `get_required_extensions(vcon_dict)`, `check_compatibility(vcon_dict)`. |
| `ExtensionType` | `COMPATIBLE`, `INCOMPATIBLE`, `EXPERIMENTAL`. |
| `ExtensionInfo`, `ExtensionValidator`, `ExtensionProcessor`, `ExtensionAttachment` | In `vcon.extensions.base`. |
| `ValidationResult(is_valid, errors, warnings)`, `ProcessingResult(success, data, errors)` | Returned by validators and processors. |

```python
from vcon.extensions import get_extension_registry

print(get_extension_registry().list_extensions())
print(vcon.validate_extensions())
```

### Lawful Basis extension

```text
LawfulBasisAttachment(lawful_basis: LawfulBasisType, expiration, purpose_grants: list[PurposeGrant],
                      terms_of_service=None, status_interval=None, content_hash=None,
                      registry=None, proof_mechanisms=None, metadata=None)
```

Methods: `is_valid()`, `has_permission(purpose)`, `get_conditions(purpose)`, `validate_content_hash()`, `to_dict()`, `LawfulBasisAttachment.from_dict(d)`.

`PurposeGrant(purpose, granted, granted_at, conditions=None)` has `is_expired(status_interval=None)`. `ProofMechanism(proof_type, timestamp, proof_data)`. `ContentHash(algorithm, canonicalization, value)` has `validate(content)`. `RegistryInfo(registry_type, url)`.

`LawfulBasisExtension` offers `create_lawful_basis_attachment(lawful_basis, expiration, purpose_grants, **kwargs) -> dict` (the attachment dict, `purpose: "lawful_basis"`), `validate_lawful_basis_attachment(attachment) -> bool` and `check_permission(vcon_dict, purpose, party_index=None) -> bool`. `LawfulBasisValidator` and `LawfulBasisProcessor` expose the `validate_*` and `check_permission` methods with `ValidationResult` and `ProcessingResult` returns.

Registry (attestation) classes in `vcon.extensions.lawful_basis.registry`:

* `RegistryClient(registry_url, auth_token=None)`: base with `submit_attestation(lawful_basis) -> str`, `verify_receipt(receipt_id)`, `query_status(attestation_id)`.
* `SCITTRegistryClient(registry_url, auth_token=None)`: SCITT client, adds `update_attestation(attestation_id, updates) -> bool`.
* `RegistryManager()`: `register_client(registry_type, client)`, `submit_to_registry(lawful_basis, registry_type) -> str`, `verify_registry_status(attestation_id, registry_type)`, `query_registry_status(attestation_id, registry_type)`.
* `RegistryValidator` (static methods): `validate_scitt_receipt(receipt)`, `validate_attestation_status(status)`, `validate_registry_metadata(metadata)`. It lives in the same registry module.

```python
from datetime import datetime, timezone
from vcon.extensions.lawful_basis import (
    LawfulBasisAttachment, LawfulBasisType, PurposeGrant, ProofMechanism, ProofType,
)

grant = PurposeGrant("recording", True, datetime.now(timezone.utc))
proof = ProofMechanism(ProofType.VERBAL_CONFIRMATION, datetime.now(timezone.utc),
                       {"recording_ref": "call-0001"})
lb = LawfulBasisAttachment(LawfulBasisType.CONSENT, None, [grant], proof_mechanisms=[proof])
print(lb.is_valid(), lb.has_permission("recording"))

vcon.add_lawful_basis_attachment(
    lawful_basis="consent", expiration=None, party_index=1,
    purpose_grants=[{"purpose": "recording", "granted": True,
                     "granted_at": datetime.now(timezone.utc).isoformat()}],
    proof_mechanisms=[proof],
)
print(vcon.check_lawful_basis_permission("recording", party_index=1))
```

### WTF extension

Data classes in `vcon.extensions.wtf`, each with `to_dict()` and `from_dict()`:

| Class | Constructor |
|---|---|
| `Transcript` | `(text, language, duration, confidence)` |
| `Segment` | `(id, start, end, text, confidence, speaker=None, words=None)` |
| `Word` | `(id, start, end, text, confidence, speaker=None, is_punctuation=None)` |
| `Speaker` | `(id, label, segments, total_time, confidence)` |
| `Quality` | `(audio_quality, background_noise, multiple_speakers, overlapping_speech, silence_ratio, average_confidence, low_confidence_words, processing_warnings)` |
| `Metadata` | `(created_at, processed_at, provider, model, processing_time=None, audio=None, options=None)` |
| `WTFAttachment` | `(transcript, segments, metadata, words=None, speakers=None, alternatives=None, enrichments=None, extensions=None, quality=None, streaming=None)` |

`created_at`, `processed_at`, `provider` and `model` are required in `metadata`. `Metadata` has no `audio_quality`; that belongs to `Quality`.

`WTFAttachment` methods: `get_speaking_time()`, `find_low_confidence_segments(threshold=0.5)`, `extract_keywords(min_confidence=0.8)`, `export_to_srt()`, `export_to_vtt()`.

`WTFProvider` enum: `WHISPER`, `DEEPGRAM`, `ASSEMBLYAI`, `GOOGLE`, `AMAZON`, `AZURE`, `REV_AI`, `SPEECHMATICS`, `WAV2VEC2`, `PARAKEET`. Adapters exist for Whisper, Deepgram and AssemblyAI: `WhisperAdapter().convert(provider_data) -> WTFAttachment`.

`WTFExtension` (also at `vcon.extensions.WTFExtension`), all taking and returning dicts:

* `create_wtf_attachment(transcript, segments, metadata, **kwargs) -> dict`
* `convert_from_provider(provider_data, provider) -> dict`
* `validate_wtf_attachment(attachment) -> bool`
* `export_transcription(attachment, format="srt") -> str`
* `analyze_transcription(attachment) -> dict`
* `compare_transcriptions(attachments) -> dict`

`WTFProcessor` has the same methods but works on `WTFAttachment` objects, plus `process(vcon_dict)` and `can_process(name)`. `WTFValidator` has `validate_attachment`, `validate_extension_usage` and `validate_wtf_attachment`.

```python
from vcon.extensions.wtf import WhisperAdapter, WTFAttachment

wtf = WhisperAdapter().convert({
    "text": "Hello world from Whisper",
    "segments": [{"start": 0.0, "end": 2.0, "text": "Hello world from Whisper"}],
})
print(wtf.export_to_vtt())

vcon.add_wtf_transcription_analysis(
    transcript=wtf.transcript.to_dict(),
    segments=[s.to_dict() for s in wtf.segments],
    metadata=wtf.metadata.to_dict(),
    dialog_index=0,
)
entry = vcon.find_analysis_by_type("transcription")
print(WTFAttachment.from_dict(Vcon.decoded_body(entry)).export_to_srt())
```

### Constants

| Constant | Value |
|---|---|
| `Dialog.VALID_TYPES` | `recording`, `text`, `transfer`, `incomplete`, `audio`, `video` |
| `Dialog.VALID_DISPOSITIONS` | `no-answer`, `congestion`, `failed`, `busy`, `hung-up`, `voicemail-no-message` |
| `PartyHistory.VALID_EVENTS` | `join`, `drop`, `hold`, `unhold`, `mute`, `unmute` |
| `Attachment.VALID_ENCODINGS` | `base64url`, `json`, `none` |
| `Dialog.MIME_TYPES` | `text/plain`, common `audio/*` and `video/*` types, `multipart/mixed`, `message/rfc822`, `image/jpeg`, `image/tiff`, `application/pdf`, `application/json` |
| `LawfulBasisType` | `consent`, `contract`, `legal_obligation`, `vital_interests`, `public_task`, `legitimate_interests` |
| `ProofType` | `verbal_confirmation`, `signed_document`, `cryptographic_signature`, `external_system` |
| `HashAlgorithm` (lawful basis content hash) | `SHA_256` (`sha-256`), `SHA_3_256` (`sha-3-256`), `BLAKE2B_256` (`blake2b-256`) |
| `CanonicalizationMethod` | `JCS` (`jcs`) |
| `ExtensionType` | `compatible`, `incompatible`, `experimental` |

`HashAlgorithm` applies only to the lawful basis `ContentHash`. Dialog and attachment `content_hash` strings use `sha512-` (default) or `sha256-` prefixes.

```python
from vcon.extensions.lawful_basis import HashAlgorithm, LawfulBasisType

print([a.value for a in HashAlgorithm], [t.value for t in LawfulBasisType])
print(Dialog.VALID_DISPOSITIONS)
```

### More examples

#### Signing

```python
private_key, public_key = Vcon.generate_key_pair()
signed = Vcon.build_new()
signed.add_party(Party(name="Alice"))
signed.sign(private_key)
print(signed.verify(public_key))
```

#### Video and images

```python
# placeholder: needs a local video file and the ffmpeg binary, not run
video = Dialog(type="video", start=now, parties=[0, 1], mediatype="video/mp4",
               resolution="1920x1080", frame_rate=30.0, codec="H.264")
video.add_video_data(open("meeting.mp4", "rb").read(), filename="meeting.mp4")
print(video.extract_video_metadata(), video.generate_thumbnail())
```

```python
import os, tempfile
from PIL import Image   # installed by vcon[image]

path = os.path.join(tempfile.mkdtemp(), "pic.jpg")
Image.new("RGB", (20, 20), "red").save(path)

attachment = vcon.add_image(path, purpose="photo")
print(attachment.to_dict()["encoding"])
image_dialog = Dialog(type="text", start=now, parties=[0], mediatype="image/jpeg")
image_dialog.add_image_data(path, "image/jpeg")
print(image_dialog.is_image())
```

#### HTTP

```python
# placeholder URLs, not run
Vcon.load_from_url("https://example.com/conversation.vcon.json")
vcon.post_to_url("https://example.com/vcons", headers={"x-api-token": "..."})
```

#### Error handling

```python
from vcon.dialog import Dialog

for call in (
    lambda: Dialog(type="bogus", start=now, parties=[0]),
    lambda: vcon.add_incomplete_dialog(now, "NO_ANSWER", parties=[0]),
    lambda: Vcon.load_from_file("/nonexistent.vcon.json"),
):
    try:
        call()
    except (ValueError, FileNotFoundError) as e:
        print(type(e).__name__, e)
```
