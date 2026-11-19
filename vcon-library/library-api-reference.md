---
description: Signatures, return types and working examples for every public class and function in the vcon 1.0 Python library.
---

# 🔌 Library API Reference

## vCon Library API Reference

Reference for `vcon` **1.0.4**, which implements [`draft-ietf-vcon-vcon-core-05`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) (`"vcon": "0.5.0"`) and reads and migrates 0.4.0. The [quickstart](quickstart.md) has release notes and known limits. What each field means, and when it is required, is in the [field reference](../vcons/field-reference.md); that page describes core-04 (`"0.4.0"`), while the library writes core-05.

### Installation

```bash
pip install vcon
pip install "vcon[image]"    # Pillow and pypdf for image and PDF helpers
pip install "vcon[video]"    # ffmpeg-python for video helpers
```

Python 3.12 or later. Video helpers need the `video` extra and the `ffmpeg` binary on `PATH`; without the extra they raise `ImportError`. JWS signing uses joserfc, and `import vcon` prints no warnings.

### Imports

The package root exports the names in `vcon.__all__`. Everything else comes from its module.

```python
from vcon import (
    Vcon, Dialog, Session, SessionReference, Event,
    AlgorithmNotPermittedError, MigrationError, MigrationReport, MigrationResult,
    migrate_0_4_0_to_0_5_0, compute_content_hash, parse_content_hash_algorithm,
)
from vcon.party import Party, PartyHistory          # PartyHistory is deprecated
from vcon.dialog import b64url_encode, b64url_decode
from vcon.civic_address import CivicAddress
from vcon.vcon import Attachment
from vcon.body import decode_body
from vcon.event import CORE_EVENT_TYPES, EVENT_PARAMETER_RULES, validate_event
from vcon.session import SESSION_REFERENCE_TYPES, validate_session
from vcon.migration import MIGRATION_STEPS
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

`from vcon import Party` raises `ImportError`.

The library logs through `logging.getLogger(__name__)` in each module and does not configure logging on import. Configure logging in your application to see its messages.

### Vcon

#### Construction and I/O

| Method | Returns | Notes |
|---|---|---|
| `Vcon(vcon_dict=None, property_handling="default", migrate_legacy=True, strict_migration=False)` | `Vcon` | Wraps an existing dict. A dict with `"vcon": "0.4.0"` is migrated (see [Migration](#migration)). `created_at` is kept exactly as written; an invalid value is reported by `is_valid()`, not raised. |
| `Vcon.build_new(created_at=None, property_handling="default")` | `Vcon` | New vCon with `vcon: "0.5.0"`, a UUID8, `created_at` and empty `parties`, `dialog`, `attachments`, `analysis`. No `sessions` until you add one. |
| `Vcon.build_from_json(json_string, property_handling="default", migrate_legacy=True, strict_migration=False)` | `Vcon` | |
| `Vcon.load(source, property_handling="default", migrate_legacy=True, strict_migration=False)` | `Vcon` | Path or URL. |
| `Vcon.load_from_file(file_path, ...)` / `Vcon.load_from_url(url, ...)` | `Vcon` | Same keywords. `FileNotFoundError` for a missing file. |
| `Vcon.migrate_0_4_0_to_0_5_0(source, strict=False, property_handling="default")` | `Vcon` | Explicit migration of a dict or JSON string, signed or not. |
| `migration_report` | `MigrationReport` or `None` | Set when the input was migrated. |
| `save_to_file(file_path)` | `None` | |
| `post_to_url(url, headers=None)` | `requests.Response` | |
| `to_json()` / `dumps()` | `str` | |
| `to_dict()` | `dict` | |
| `is_valid()` | `(bool, list[str])` | Core-05 rules, see below. |
| `get_validation_warnings()` | `list[str]` | SHOULD-level findings, not errors. |
| `Vcon.validate_file(file_path)` / `Vcon.validate_json(json_str)` | `(bool, list[str])` | |
| `sign(private_key, alg="RS256")` / `verify(public_key)` | `None` / `bool` | JWS through joserfc. Keys are `cryptography` RSA, EC or Ed25519 objects; pass the matching `alg` (`ES256`, `EdDSA`, ...). Signing adds `payload` and `signatures`. `alg` "none" raises `AlgorithmNotPermittedError` in both. `verify()` returns `False` when no `alg` is declared and raises `ValueError` for an unsigned vCon. |
| `Vcon.generate_key_pair()` | `(private_key, public_key)` | RSA 2048. |
| `Vcon.uuid8_domain_name(domain)` / `Vcon.uuid8_time(custom_c_62_bits)` | `str` | UUID8 helpers. |
| `set_created_at(ts)` / `set_updated_at(ts)` | `None` | `str` or `datetime`. |

`property_handling` is `"default"` (keep unknown properties), `"strict"` (remove them) or `"meta"` (move them into `meta`). The constants are `vcon.vcon.PROPERTY_HANDLING_DEFAULT`, `_STRICT` and `_META`. An unmigrated 0.4.0 dict (`migrate_legacy=False`, or a signed one) is kept exactly as given and `property_handling` is not applied to it.

`is_valid()` checks required fields; that `vcon`, when present, is `"0.5.0"`; that a session exists when there is dialog or events; every party, dialog, session, attachment and analysis index; the event parameter table; that attachments have `party` and `dialog` or `session`; that `redacted_from` and `amended_from` are not both present; that `extensions` lists every extension in use and every `critical` entry; that a dialog or attachment with inline content has a `mediatype`; that every `__REDACTED_REF_#` has a `redacted_strings` entry; that `created_at` is ISO 8601; that every `content_hash` (dialog, attachment, analysis, `redacted_from`, `amended_from`) is `<algorithm>-<unpadded base64url digest>`, with an 86-character digest for `sha512` and 43 for `sha256`, so hex and padded digests fail; and that every `purpose: "lawful_basis"` attachment body has `lawful_basis`, `expiration` and `purpose_grants` and well-formed grants and proof mechanisms.

`get_validation_warnings()` reports a dialog that no session lists, a session without `start` or `parties`, a `url` without a `content_hash` (an `is_valid()` error from 1.1), an `encoding: "json"` body that is a serialized JSON string, and lawful basis proof types, hash algorithms or canonicalization methods the draft does not define.

`PERMITTED_JWS_ALGORITHMS` in `vcon.vcon` lists `RS256`, `RS384`, `RS512`, `PS256`, `PS384`, `PS512`, `ES256`, `ES384`, `ES512` and `EdDSA`. `AlgorithmNotPermittedError` is a `ValueError`.

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
                       body="Hello", encoding="none", mediatype="text/plain"), session=0)
print(vcon.vcon, vcon.is_valid(), vcon.get_validation_warnings())   # 0.5.0 (True, []) []

ref = Dialog(type="recording", start=now, parties=[0], mediatype="audio/x-wav",
             url="https://example.com/call.wav")   # placeholder URL, no content_hash
draft = Vcon(vcon.to_dict())
draft.add_dialog(ref, session=0)
print(draft.is_valid()[0], draft.get_validation_warnings()[0][:40])   # True Dialog at index 1 has a url but no conte
ref.content_hash = "sha512-" + "ab" * 64                               # hex, not base64url
draft.vcon_dict["dialog"][1] = ref.to_dict()
print(draft.is_valid()[1][0][:46])                                     # Dialog at index 1 has an invalid content_hash
```

#### Properties

Read-only: `vcon`, `uuid`, `subject`, `created_at`, `updated_at`, `parties` (a `list[dict]`), `sessions`, `dialog`, `events`, `attachments`, `analysis`, `redacted_from`, `amended_from`, `redacted_strings`, `group`, `meta`, `tags`, `migration_report`. There is no setter for `subject`; write `vcon.vcon_dict["subject"] = "..."`. The instance holds its data in `vcon.vcon_dict`.

`redacted` and `amended` are deprecated aliases for `redacted_from` and `amended_from` and emit a `DeprecationWarning`. There is no `appended`, no `must_support` or `must_understand`, and no `extensions` or `critical` property. Use the methods below.

#### Parties, sessions, dialogs and events

| Method | Notes |
|---|---|
| `add_party(party: Party)` | |
| `get_party_objects() -> list[Party]` | `parties` returns dicts, this returns `Party` objects. |
| `find_party_index(by, val) -> int or None` | `by` is a party attribute such as `"name"` or `"tel"`. |

`add_party()`, `add_dialog()`, `add_analysis()` and the `find_*` methods work on a vCon that has no `parties`, `dialog` or `analysis` key (all optional in core-05).
| `add_session(session=None) -> int` | A `Session`, a dict, or nothing for an empty `{}` session. Returns the index. |
| `add_dialog(dialog: Dialog, session=None) -> int` | Returns the dialog index. `session` is the index of an existing session; the dialog index is appended to its `dialog` list. An out-of-range index raises `ValueError`. |
| `add_event(event: Event or dict) -> int` | Returns the event index. An `Event` is checked when built; a dict is checked by `is_valid()`. |
| `find_dialog(by, val) -> Dialog or None` | |
| `find_dialogs_by_type(type) -> list[dict]` | |
| `add_incomplete_dialog(start, disposition, details=None, parties=None, metadata=None, session=None) -> int` | `disposition` is lowercase and hyphenated, see constants; other values raise `ValueError`. Writes no `body`; `details` goes to `metadata["details"]`. `parties` is the party that attempted the call. |
| `add_transfer_dialog(...)` | Removed. Raises `NotImplementedError`; record a transfer with transfer events. |

#### Attachments and analysis

```text
add_attachment(purpose, body=None, encoding="none", mediatype=None, filename=None,
               url=None, content_hash=None, start=None, party=None, dialog=None,
               session=None) -> Attachment
add_image(image_path, purpose="image", party=None, dialog=None, session=None) -> Attachment
find_attachment_by_purpose(purpose) -> dict | None
add_analysis(*, type, vendor, body, dialog=None, sessions=None, attachment=None,
             analysis=None, encoding="none", mediatype=None, filename=None, product=None,
             url=None, content_hash=None, schema=None, meta=None, **extra) -> None
find_analysis_by_type(type) -> dict | None
```

`encoding` is `"json"`, `"base64url"` or `"none"`; anything else raises `ValueError` from `add_attachment` and `Exception` from `add_analysis`. With `encoding="json"`, `body` is the JSON value itself. `party` defaults to `0`, `start` to the vCon's `created_at`, and `mediatype` to `application/json` for JSON bodies. `dialog` has no default. With neither `dialog` nor `session`, `add_attachment()` uses the only session when the vCon has exactly one and raises `ValueError` otherwise. `session` may be an index or a list of indices. `add_analysis` takes keyword arguments only and requires `vendor`; `dialog`, `sessions`, `attachment` and `analysis` are optional references (an index or a list). `type=` is analysis only; attachments use `purpose=`.

`add_image()` reads a JPEG, TIFF or PDF file (anything else raises `ValueError`; it needs `vcon[image]`) and writes a `base64url` attachment. It resolves `dialog` and `session` like `add_attachment()`: with neither, it uses the only session or raises `ValueError`.

```python
vcon.add_attachment(purpose="notes", body={"agent_note": "Verified."}, encoding="json", session=0)
vcon.add_analysis(type="sentiment", dialog=0, vendor="example-vendor",
                  body={"sentiment": "positive", "confidence": 0.95}, encoding="json")
print(vcon.find_attachment_by_purpose("notes"))
print(Vcon.decoded_body(vcon.find_analysis_by_type("sentiment")))   # {'sentiment': 'positive', 'confidence': 0.95}
```

`Vcon.decoded_body(entry)` is a static method, also available as `vcon.body.decode_body(entry)`. It returns the decoded value of an attachment, analysis or dialog entry and accepts both a JSON value and a stringified JSON body written by older code.

#### Tags

`add_tag(tag_name, tag_value, party=None)`, `get_tag(tag_name) -> str | None`, and the `tags` property (the tags attachment). Tags are stored as a list of `"name:value"` strings in a `purpose: "tags"` attachment. The tags attachment lists every session in `session`; if the vCon has no session, `add_tag()` adds an empty `{}` session first.

`party` is the party that adds the tags and is used when the tags attachment is created; it defaults to 0. An index that does not exist raises `ValueError`. On a vCon with no parties, `add_tag()` still writes party 0 but emits a `DeprecationWarning`, because the result fails `is_valid()`; from 1.1 it will raise.

```python
tagged = Vcon.build_new()
tagged.add_party(Party(name="Alice"))
try:
    tagged.add_tag("priority", "high", party=3)
except ValueError as e:
    print(str(e)[:47])   # add_tag() needs a valid party index for the tag
tagged.add_tag("priority", "high")
print(tagged.get_tag("priority"), tagged.tags["party"])   # high 0
```

#### Extensions and critical

| Method | Returns |
|---|---|
| `add_extension(name)` / `remove_extension(name)` | `None` |
| `get_extensions()` | `list[str]` |
| `add_critical(name)` / `remove_critical(name)` | `None` |
| `get_critical()` | `list[str]` |

`critical` lists extensions a receiver must understand to process the vCon. Every `critical` entry must also be in `extensions`, and every extension in use (`lawful_basis`, `wtf_transcription`, or a value outside the core vocabulary) must be listed; `is_valid()` reports both.

```python
vcon.add_extension("x-example")
vcon.add_critical("x-example")
print(vcon.get_extensions(), vcon.get_critical())   # ['x-example'] ['x-example']
vcon.remove_critical("x-example")
vcon.remove_extension("x-example")
```

#### Extension helpers on Vcon

| Method | Notes |
|---|---|
| `add_lawful_basis_attachment(lawful_basis, expiration, purpose_grants, party_index=None, dialog_index=None, session_index=None, **kwargs)` | `purpose_grants` is a list of dicts. Extra keywords go to `LawfulBasisAttachment` (`terms_of_service`, `status_interval`, `content_hash`, `registry`, `proof_mechanisms`, `metadata`). Writes `purpose: "lawful_basis"` and adds the `lawful_basis` extension. `expiration` may be `None`. Pass `session_index`; with only `dialog_index` it logs a warning and writes a dialog-only scope, and with neither it uses the only session or raises `ValueError`. A missing `party_index` defaults to 0 with a warning. |
| `find_lawful_basis_attachments(party_index=None)` | |
| `check_lawful_basis_permission(purpose, party_index=None) -> bool` | |
| `add_wtf_transcription_analysis(transcript, segments, metadata, dialog_index=None, session_index=None, **kwargs)` | Writes a `type: "wtf_transcription"` entry to `analysis[]` and adds the `wtf_transcription` extension. Preferred. |
| `add_wtf_transcription_attachment(transcript, segments, metadata, party_index=None, dialog_index=None, session_index=None, **kwargs)` | Deprecated since 1.0.4; emits a `DeprecationWarning`. Writes to `attachments[]`, scoped like `add_attachment()`. The WTF draft does not allow WTF in `attachments[]`; use `add_wtf_transcription_analysis()`. |
| `find_wtf_attachments(party_index=None)` | Searches `attachments[]` only. |
| `validate_extensions() -> dict` | Keyed by extension name, plus `attachments`. |
| `process_extensions() -> dict` | Name to `ProcessingResult`. |

`add_wtf_transcription_analysis()` writes the entry [`draft-howe-vcon-wtf-extension`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/) defines: `type: "wtf_transcription"`, `encoding: "json"`, the WTF document as the `body` value, `vendor` and `product` from `metadata["provider"]` and `metadata["model"]`, and the draft URL in `schema`. `dialog_index` becomes `dialog: [n]`; `session_index` becomes `sessions`. Releases before 1.0.3 wrote `type: "transcription"` with a JSON-string body; `Vcon.decoded_body()` reads both.

### Session

```text
Session(start=None, duration=None, parties=None, originator=None, session_id=None,
        sessions=None, dialog=None)
SessionReference(session: int, type: str, description=None)
```

A session groups the dialogs and events of one conversation or call leg. `start` is a `datetime` or ISO 8601 string and SHOULD be present. `duration` is in seconds; no `duration` means the session has not ended. `parties` lists the receivers; the first one is the originator, so set `originator` only when it is someone else. `session_id` is a SessionId object (`{"local": ..., "remote": ...}`). `dialog` lists dialog indices; `add_dialog(..., session=n)` maintains it for you. `sessions` relates this session to others with `SessionReference` objects or dicts; `type` is `child`, `peer`, `precursor` or `breakout` (`vcon.session.SESSION_REFERENCE_TYPES`), or a value from an extension. `to_dict()` omits unset fields, so `Session()` is `{}`, which core-05 allows.

`validate_session(session_dict, party_count=None, dialog_count=None, session_count=None)` returns `(errors, warnings)`.

```python
from vcon import Session, SessionReference

print(Session(start="2026-10-06T14:00:00Z", parties=[0, 1], duration=62.5).to_dict())
# {'start': '2026-10-06T14:00:00+00:00', 'duration': 62.5, 'parties': [0, 1]}
vcon.add_party(Party(name="Carol", role="supervisor"))   # party 2
# Bob (1) calls Carol for a consultation; the call is related to session 0
consult = vcon.add_session(Session(start=now, parties=[1, 2],
                                   sessions=[SessionReference(0, "peer")]))
print(consult, vcon.sessions[consult]["sessions"])   # 1 [{'session': 0, 'type': 'peer'}]
```

### Event

```text
Event(type, time, sessions=None, party=None, dialog=None, channel=None, contributors=None,
      button=None, transferee=None, transfer_target=None, consultation=None,
      target_session=None, validate=True, **extra)
Event.from_dict(data, validate=False)
```

An event is a point in time inside one or more sessions. `time` is a `datetime` or ISO 8601 string. With `validate=True`, the constructor applies the parameter table of core-05 Section 4.3.1.5 and raises `ValueError` for a missing required (M) or forbidden (X) parameter; a SHOULD NOT (SN) parameter logs a warning. Event types outside the core list are checked only for `type` and `time`. Extra keywords are written as extension parameters.

| Event types | Required | Optional or forbidden |
|---|---|---|
| `join`, `drop`, `hold`, `unhold`, `mute`, `unmute` | `sessions`, `party` | `dialog` should not be used |
| `keydown`, `keyup` | `sessions`, `party`, `button` | |
| `consultation-start` | `sessions`, `party`, `transferee`, `transfer_target`, `consultation` | no `target_session` |
| `blind-transfer` | `sessions`, `party`, `transferee`, `transfer_target`, `target_session` | no `consultation` |
| `consultative-transfer` | `sessions`, `party`, `transferee`, `transfer_target`, `consultation`, `target_session` | |
| `transfer-abandoned` | `sessions`, `party`, `transferee` | `transfer_target`, `consultation` optional; no `target_session` |
| `channel-contributors` | `dialog`, `channel` | `contributors` optional; no `sessions` or `party`. The dialog must be a `recording`. |

`sessions` is a session index or a list (one index for transfer types). `party`, `transferee`, `transfer_target` and `contributors` are party indices; `consultation` and `target_session` are session indices. `vcon.event.CORE_EVENT_TYPES` and `EVENT_PARAMETER_RULES` hold the tables. `validate_event(event_dict, party_count=None, dialog_count=None, session_count=None)` returns `(errors, warnings)`.

```python
from vcon import Event

vcon.add_event(Event(type="join", time=now, sessions=0, party=1))
vcon.add_event(Event(type="keydown", time=now, sessions=0, party=0, button="5"))
vcon.add_event(Event(type="consultation-start", time=now, sessions=0, party=1,
                     transferee=0, transfer_target=2, consultation=consult))
try:
    Event(type="blind-transfer", time=now, sessions=0, party=1, transferee=0, transfer_target=2)
except ValueError as e:
    print(e)   # Event of type 'blind-transfer' is missing required field: target_session
print(len(vcon.events), vcon.is_valid())   # 3 (True, [])
```

### Migration

```text
vcon.migrate_0_4_0_to_0_5_0(vcon, *, strict=False) -> MigrationResult
```

Takes a 0.4.0 vCon as a dict or JSON string, leaves the input unchanged, and returns `MigrationResult(vcon, report)`, a named tuple that unpacks as `(dict, MigrationReport)`. A signed vCon is migrated as its unsigned payload and `report.resign_required` is `True`; the library never re-signs. Input with no `vcon` parameter is treated as 0.4.0 unless it has a 0.5.0-only parameter (`sessions`, `events`, `redacted_from`, `amended_from`, `redacted_strings`).

`MigrationError` (a `ValueError`) is raised for input that is not a 0.4.0 vCon (including a 0.5.0 one), is not a dict or JSON, or, with `strict=True`, when a required field has no source.

The steps follow core-05 Appendix A.1 and are listed in `vcon.migration.MIGRATION_STEPS` (`T1` to `T15`): a session is added for the dialogs; `party_history` becomes events; `originator` moves to the session; transfer dialogs become transfer events with sessions for each call; recording-set dialogs become sessions; removed dialogs are replaced by `{}` in place so indices do not change; references to removed dialogs are re-pointed; an attachment with no scope gets the only session when there is exactly one; `redacted` and `amended` become `redacted_from` and `amended_from`; and `extensions` gets every extension in use. A lawful basis attachment keeps its dialog-only scope and the report notes it.

`MigrationReport` fields and methods:

| Name | Meaning |
|---|---|
| `source_version`, `target_version`, `strict`, `resign_required` | |
| `entries` | `ReportEntry(step, kind, message, code, details)` items |
| `inferred`, `fallbacks`, `unresolved`, `notes` | Entries of kind `inferred`, `fallback`, `unresolved` and `info` |
| `removed_dialogs` | Original content of each dialog replaced by `{}`, keyed by index |
| `steps_run` | Step IDs in the order they ran |
| `summary()` | `"N inferred, N fallback, N unresolved"`, plus `"; re-signing required"` when set |
| `to_dict()` | JSON-ready dict; store it with the migrated record |

```python
from vcon import MigrationError, migrate_0_4_0_to_0_5_0

old = {
    "vcon": "0.4.0", "uuid": "01928e10-193e-8231-b9a2-279e0d16bc46",
    "created_at": "2026-10-06T14:00:00+00:00",
    "parties": [{"name": "Alice"}, {"name": "Bob"}],
    "dialog": [{"type": "text", "start": "2026-10-06T14:00:00+00:00", "parties": [0, 1],
                "body": "Hi", "encoding": "none", "mediatype": "text/plain"}],
    "redacted": {"uuid": "01928e10-0000-8000-8000-000000000000", "type": "pii"},
}
new, report = migrate_0_4_0_to_0_5_0(old)
print(new["vcon"], new["dialog"][0]["parties"], new["sessions"][0]["parties"])   # 0.5.0 [0] [0, 1]
print("redacted_from" in new, report.summary())   # True 1 inferred, 0 fallback, 0 unresolved

migrated = Vcon(old)
print(migrated.vcon, migrated.migration_report.summary())   # 0.5.0 1 inferred, 0 fallback, 0 unresolved
try:
    migrate_0_4_0_to_0_5_0(new)
except MigrationError as e:
    print(e)   # cannot migrate: input vcon is '0.5.0', expected '0.4.0'
```

### Party

```text
Party(tel=None, stir=None, mailto=None, name=None, validation=None, gmlpos=None,
      civicaddress=None, uuid=None, role=None, contact_list=None, meta=None,
      sip=None, did=None, jCard=None, timezone=None, **kwargs)
```

`to_dict()` omits unset fields.

`CivicAddress(country, a1..a6, prd, pod, sts, hno, hns, lmk, loc, flr, nam, pc)` holds GEOPRIV fields; pass it as `civicaddress=`.

`PartyHistory(party, event, time, button=None)` is deprecated and emits a `DeprecationWarning`. It cannot be attached to a dialog. `to_event(sessions)` returns the equivalent `Event`.

```python
from vcon.civic_address import CivicAddress
from vcon.party import Party

p = Party(name="Jane", sip="sip:jane@example.com", timezone="America/New_York",
          civicaddress=CivicAddress(country="US", a1="CA", a3="San Francisco", pc="94102"))
print(p.to_dict())
```

### Dialog

```text
Dialog(type, start, parties, mediatype=None, filename=None, body=None, encoding=None,
       url=None, disposition=None, campaign=None, interaction=None, skill=None,
       duration=None, meta=None, metadata=None, signaling=None, resolution=None,
       frame_rate=None, codec=None, bitrate=None, thumbnail=None, session_id=None,
       content_hash=None, application=None, message_id=None, redacted_segments=None,
       **kwargs)
Dialog.placeholder()
```

`type` is `recording`, `text` or `incomplete`. `audio` and `video` are accepted with a `DeprecationWarning` and become `recording`; the media kind stays in `mediatype`. Other values raise `ValueError`. `incomplete` needs a `disposition`. `parties` lists the contributors only; the receivers are the `parties` of the session that lists the dialog.

The 0.4.0 parameters `originator`, `party_history`, `transfer`, `transferee`, `transferor`, `transfer_target`, `original`, `consultation`, `target_dialog`, `recordings` and `recording_set` raise `ValueError`. Any other unknown keyword is accepted and written as a property, so `mimetype=` becomes a stray `mimetype` field. Use `mediatype`. A dialog with a `body` and no `mediatype` fails `is_valid()`.

`Dialog.placeholder()` builds an empty dialog that serializes as `{}`. It holds an array position so that other indices do not change.

| Method | Notes |
|---|---|
| `to_dict()` | Omits empty `meta` and `metadata`. |
| `add_external_data(url, filename, mediatype)` | Downloads the URL, sets `url` and a `sha512-` hash. Raises if the fetch fails. |
| `add_inline_data(body: bytes \| str, filename, mediatype)` | Sets `encoding="base64url"` and a `sha512-` hash over the decoded content. `bytes` are encoded to unpadded base64url; a `str` is taken to be base64url already. |
| `add_image_data(image_path, mediatype=None)`, `extract_image_metadata(image_data, mediatype)` | Needs `vcon[image]`. Body is unpadded base64url. |
| `add_video_data(video_data, filename=None, mediatype=None, inline=True, metadata=None)` | Sets `type` to `recording`; `mediatype` is guessed from the filename extension when not given. With `inline=False`, `video_data` must be a URL. |
| `add_video_with_optimal_storage(video_data, filename, mediatype=None, size_threshold_mb=10)` | |
| `add_streaming_video_reference(reference_id, mediatype, metadata=None)` | |
| `extract_video_metadata(video_path=None) -> dict` | Duration, size, codec, width, height, frame rate and more from ffprobe. Reads the inline body, the `url`, or `video_path`. Needs `vcon[video]` and ffmpeg. |
| `generate_thumbnail(*args, **kwargs)` | Calls `generate_video_thumbnail()` for a video dialog, otherwise `generate_image_thumbnail()`. |
| `generate_video_thumbnail(timestamp=0.0, width=320, height=240, quality=90) -> bytes` | JPEG bytes of one frame. Needs `vcon[video]` and ffmpeg; raises `ValueError` for a dialog that is not video. |
| `generate_image_thumbnail(max_size=(200, 200)) -> str \| None` | Base64url JPEG thumbnail of an image dialog. Needs `vcon[image]`. |
| `get_video_format_from_mediatype(mediatype=None) -> str` | Container name such as `mp4`; defaults to the dialog's `mediatype`. |
| `transcode_video(target_format, codec=None, bit_rate=None, width=None, height=None)` | Rewrites the inline body in the new format and records the original format. Needs `vcon[video]` and ffmpeg. |
| `to_inline_data()` | Fetches an external body and inlines it as unpadded base64url. |
| `is_external_data()`, `is_inline_data()`, `is_text()`, `is_recording()`, `is_incomplete()`, `is_placeholder()`, `is_audio()`, `is_video()`, `is_email()`, `is_image()`, `is_pdf()`, `has_thumbnail()` | `bool`. `is_audio()` and `is_video()` test `mediatype`. |
| `is_transfer()` | Deprecated; warns and returns `False`. |
| `set_session_id(session_id)` / `get_session_id()` | Dict or list of dicts. |
| `set_content_hash(h)` / `get_content_hash()` | |
| `calculate_content_hash(algorithm="sha512") -> str` | Hashes the content: the decoded bytes of a `base64url` body, otherwise the `body` as UTF-8. Raises `ValueError` with no body. |
| `verify_content_hash(expected_hash, algorithm=None) -> bool` | Reads the algorithm from the prefix of `expected_hash`. Also accepts a hash of the encoded `base64url` text, as 1.0.0 and earlier computed it. A legacy unprefixed SHA-256 value still verifies. |
| `is_external_data_changed() -> bool` | |

```python
from vcon import compute_content_hash, parse_content_hash_algorithm
from vcon.dialog import b64url_decode, b64url_encode

d = Dialog(type="text", start=now, parties=[0], body="hello",
           encoding="none", mediatype="text/plain")
h = d.calculate_content_hash()
print(h, parse_content_hash_algorithm(h), d.verify_content_hash(h))
print(compute_content_hash(b"hello", "sha256"))
print(b64url_encode(b"ab"), b64url_decode("YWI"), b64url_decode("YWI="))   # YWI b'ab' b'ab'
print(Dialog.placeholder().to_dict())                                       # {}
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

`Attachment(purpose, body=None, encoding="none", mediatype=None, filename=None, url=None, content_hash=None, start=None, party=None, dialog=None, session=None)` defaults `party` to 0 and `mediatype` to `application/json` for JSON bodies. It does not default `dialog`, `session` or `start`; `Vcon.add_attachment()` fills `start` and the scope. `Attachment.from_image(image_path, purpose="image")` builds one from a JPEG, TIFF or PDF file. `to_dict()` serializes. `Attachment.VALID_ENCODINGS` is `["base64url", "json", "none"]`.

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

print(get_extension_registry().list_extensions())   # ['lawful_basis', 'wtf_transcription']
print(vcon.validate_extensions())
```

### Lawful Basis extension

```text
LawfulBasisAttachment(lawful_basis: LawfulBasisType, expiration, purpose_grants: list[PurposeGrant],
                      terms_of_service=None, status_interval=None, content_hash=None,
                      registry=None, proof_mechanisms=None, metadata=None)
```

Methods: `is_valid()`, `has_permission(purpose)`, `get_conditions(purpose)`, `validate_content_hash()`, `to_dict()`, `LawfulBasisAttachment.from_dict(d)`.

`Vcon.is_valid()` runs `LawfulBasisValidator` over the body of every `purpose: "lawful_basis"` attachment, so a hand-written body missing `lawful_basis`, `expiration` or `purpose_grants`, or with a malformed grant or proof mechanism, fails. A `proof_type`, hash algorithm or canonicalization method the draft does not define is a warning from `get_validation_warnings()`, not an error.

`PurposeGrant(purpose, granted, granted_at, conditions=None)` has `is_expired(status_interval=None)`. `ProofMechanism(proof_type, timestamp, proof_data)`. `ContentHash(algorithm, canonicalization, value)` takes `HashAlgorithm` and `CanonicalizationMethod` values or their strings (`"sha-256"`, `"jcs"`), raises `ValueError` for anything else, and has `validate(content)`. `RegistryInfo(registry_type, url)`.

`LawfulBasisExtension` offers `create_lawful_basis_attachment(lawful_basis, expiration, purpose_grants, **kwargs) -> dict` (the attachment dict, `purpose: "lawful_basis"`, without `party`, scope, `start` or `mediatype`), `validate_lawful_basis_attachment(attachment) -> bool` and `check_permission(vcon_dict, purpose, party_index=None) -> bool`. `LawfulBasisValidator` and `LawfulBasisProcessor` expose the `validate_*` and `check_permission` methods with `ValidationResult` and `ProcessingResult` returns.

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
print(lb.is_valid(), lb.has_permission("recording"))   # True True

vcon.add_lawful_basis_attachment(
    lawful_basis="consent", expiration=None, party_index=1, session_index=0,
    purpose_grants=[{"purpose": "recording", "granted": True,
                     "granted_at": datetime.now(timezone.utc).isoformat()}],
    proof_mechanisms=[proof],
)
print(vcon.check_lawful_basis_permission("recording", party_index=1))   # True

hand_written = Vcon(vcon.to_dict())
hand_written.add_attachment(purpose="lawful_basis", body={"lawful_basis": "consent"},
                            encoding="json", session=0)
print(hand_written.is_valid()[1])
# ['Attachment at index 2 (lawful_basis): Missing required field: expiration', 'Attachment at index 2 (lawful_basis): Missing required field: purpose_grants']
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
entry = vcon.find_analysis_by_type("wtf_transcription")
print(WTFAttachment.from_dict(Vcon.decoded_body(entry)).export_to_srt())
```

### Constants

| Constant | Value |
|---|---|
| `Dialog.VALID_TYPES` | `recording`, `text`, `incomplete`, plus `audio` and `video`, which are accepted for backward compatibility and written as `recording` |
| `Dialog.VALID_DISPOSITIONS` | `no-answer`, `congestion`, `failed`, `busy`, `hung-up`, `voicemail-no-message` |
| `vcon.event.CORE_EVENT_TYPES` | `join`, `drop`, `hold`, `unhold`, `mute`, `unmute`, `keydown`, `keyup`, `consultation-start`, `blind-transfer`, `consultative-transfer`, `transfer-abandoned`, `channel-contributors` |
| `vcon.session.SESSION_REFERENCE_TYPES` | `child`, `peer`, `precursor`, `breakout` |
| `vcon.validation.SYNTAX_VERSION` | `"0.5.0"` |
| `vcon.vcon.PERMITTED_JWS_ALGORITHMS` | `RS256`, `RS384`, `RS512`, `PS256`, `PS384`, `PS512`, `ES256`, `ES384`, `ES512`, `EdDSA` |
| `Attachment.VALID_ENCODINGS` | `base64url`, `json`, `none` |
| `Dialog.MIME_TYPES` | `text/plain`, common `audio/*` and `video/*` types, `multipart/mixed`, `message/rfc822`, `image/jpeg`, `image/tiff`, `application/pdf`, `application/json`. For dialogs, `is_valid()` ignores media type parameters (`; boundary=...`) and also accepts any `multipart/*`, `message/*` or `text/*` type. |
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
from vcon import AlgorithmNotPermittedError

private_key, public_key = Vcon.generate_key_pair()
signed = Vcon.build_new()
signed.add_party(Party(name="Alice"))
signed.sign(private_key)
print(signed.verify(public_key))   # True
try:
    Vcon.build_new().sign(private_key, alg="none")
except AlgorithmNotPermittedError as e:
    print(e)                       # JWS alg "none" is not permitted (core-05 section 6.2.2)
```

#### Video and images

Video helpers need `pip install "vcon[video]"` and the `ffmpeg` binary. This example makes a one-second test clip with ffmpeg-python.

```python
import os, tempfile
import ffmpeg   # installed by vcon[video]

clip = os.path.join(tempfile.mkdtemp(), "clip.mp4")
ffmpeg.input("testsrc=size=320x240:rate=10:duration=1", f="lavfi").output(
    clip, pix_fmt="yuv420p", loglevel="error").run()

video = Dialog(type="recording", start=now, parties=[0, 1])
with open(clip, "rb") as f:
    video.add_video_data(f.read(), filename="clip.mp4")   # mediatype video/mp4
meta = video.extract_video_metadata()
print(meta["width"], meta["height"], meta["duration"])     # 320 240 1.0
thumb = video.generate_thumbnail(timestamp=0.5)            # generate_video_thumbnail()
print(thumb[:3])                                           # b'\xff\xd8\xff' (JPEG)
```

```python
import os, tempfile
from PIL import Image   # installed by vcon[image]

path = os.path.join(tempfile.mkdtemp(), "pic.jpg")
Image.new("RGB", (20, 20), "red").save(path)

attachment = vcon.add_image(path, purpose="photo", session=0)
print(attachment.to_dict()["encoding"], vcon.is_valid())   # base64url (True, [])
image_dialog = Dialog(type="text", start=now, parties=[0], mediatype="image/jpeg")
image_dialog.add_image_data(path, "image/jpeg")
print(image_dialog.is_image())     # True
print(image_dialog.verify_content_hash(image_dialog.content_hash))   # True
```

#### HTTP

```python
# placeholder URLs, not run
Vcon.load_from_url("https://example.com/conversation.vcon.json")
vcon.post_to_url("https://example.com/vcons", headers={"x-api-token": "..."})
```

#### Error handling

```python
for call in (
    lambda: Dialog(type="transfer", start=now, parties=[0]),
    lambda: Dialog(type="recording", start=now, parties=[0], party_history=[]),
    lambda: vcon.add_incomplete_dialog(now, "NO_ANSWER", parties=[0]),
    lambda: Event(type="join", time=now, party=0),
    lambda: vcon.add_transfer_dialog(),
    lambda: Vcon.build_new().add_attachment(purpose="notes", body="x"),
    lambda: Vcon.load_from_file("/nonexistent.vcon.json"),
):
    try:
        call()
    except (ValueError, NotImplementedError, FileNotFoundError) as e:
        print(type(e).__name__, str(e)[:60])
```
