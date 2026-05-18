---
description: The Python vCon library
---

# 🐰 Quickstart

## vCon Library

A Python library for working with vCon (Virtual Conversation) objects according to the vCon specification.

> **Current version:** `vcon` **0.9.2** (May 2026). Install with `pip install vcon`.
>
> **Spec target:** [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) · syntax parameter `"vcon": "0.4.0"`.

### What's new in 0.9.2

- **WTF transcription helper.** `add_wtf_transcription_attachment()` generates a World Transcription Format payload. See [WTF Transcription extension](../extensions/wtf-transcription.md) for placement guidance — the recommended location is `analysis[]`, not `attachments[]`.
- **Spec-correct field names.** Pre-0.9.1 code used `appended` and `must_support`; the current spec uses `amended` and `critical` (the latter typically appears as `must_understand[]` at the top level). The library now uses the new names. If you have stored vCons using the old names, they remain parseable for compatibility, but new code should use the spec-correct names.
- **Drops empty defaults.** Pre-0.9.1 vCons emitted empty `group: []` and `redacted: {}` blocks by default; 0.9.2 omits them, producing cleaner output.
- **`vcon` syntax parameter.** The library now emits `"vcon": "0.4.0"` automatically on output. (Older versions required you to set it manually — see the quirks list at the bottom of this page.)

### Overview

The vCon library provides a complete implementation of the vCon format for representing conversations and related metadata. It supports all features defined in the current vCon specification including:

* **Conversation Management**: Parties, dialogs, attachments, and analysis
* **Contact Information**: Multiple contact methods (tel, email, SIP, DID)
* **Media Support**: Audio, video, text, and image formats
* **Security**: Digital signatures and content hashing
* **Extensibility**: Extensions and `must_understand` (a.k.a. `critical`) fields
* **Location Data**: Civic address information (GEOPRIV)
* **Event Tracking**: Party history with join/drop/hold/mute events
* **Privacy Compliance**: [Lawful Basis](../extensions/lawful-basis.md) extension for GDPR compliance
* **Transcription Support**: [WTF (World Transcription Format)](../extensions/wtf-transcription.md) extension for standardized speech-to-text

### Quick Start

#### Basic vCon Creation

```python
from vcon import Vcon
from vcon.party import Party
from vcon.dialog import Dialog
from datetime import datetime, timezone

# Create a new vCon
vcon = Vcon.build_new()

# Add parties
caller = Party(tel="+1234567890", name="Alice", role="caller")
agent = Party(tel="+1987654321", name="Bob", role="agent")
vcon.add_party(caller)
vcon.add_party(agent)

# Add dialog
dialog = Dialog(
    type="text",
    start=datetime.now(timezone.utc),
    parties=[0, 1],
    body="Hello, I need help with my account."
)
vcon.add_dialog(dialog)

# Save to file
vcon.save_to_file("conversation.vcon.json")
print(f"Created vCon: {vcon.uuid}")
```

#### Privacy Compliance with Lawful Basis Extension

```python
from datetime import timedelta

# Add lawful basis for consent
vcon.add_lawful_basis_attachment(
    lawful_basis="consent",
    expiration=(datetime.now(timezone.utc) + timedelta(days=365)).isoformat(),
    purpose_grants=[
        {
            "purpose": "recording",
            "granted": True,
            "granted_at": datetime.now(timezone.utc).isoformat()
        },
        {
            "purpose": "transcription",
            "granted": True,
            "granted_at": datetime.now(timezone.utc).isoformat()
        }
    ],
    party_index=0
)

# Check permissions
can_record = vcon.check_lawful_basis_permission("recording", party_index=0)
print(f"Can record: {can_record}")
```

#### Transcription with WTF Extension

```python
# Add transcription
vcon.add_wtf_transcription_attachment(
    transcript={
        "text": "Hello, I need help with my account.",
        "language": "en",
        "duration": 4.2,
        "confidence": 0.92
    },
    segments=[
        {
            "id": 0,
            "start": 0.0,
            "end": 4.2,
            "text": "Hello, I need help with my account.",
            "confidence": 0.92,
            "speaker": 0
        }
    ],
    metadata={
        "created_at": datetime.now(timezone.utc).isoformat(),
        "processed_at": datetime.now(timezone.utc).isoformat(),
        "provider": "whisper",
        "model": "whisper-1"
    },
    party_index=0,
    dialog_index=0
)

# Export to SRT format
attachments = vcon.find_wtf_attachments(party_index=0)
if attachments:
    from vcon.extensions.wtf import WTFAttachment
    wtf_attachment = WTFAttachment.from_dict(attachments[0]["body"])
    srt_content = wtf_attachment.export_to_srt()
    print("SRT Export:")
    print(srt_content)
```

#### Complete Example with Extensions

```python
# Create comprehensive vCon with extensions
vcon = Vcon.build_new()

# Add parties
caller = Party(tel="+1234567890", name="Alice", role="caller")
agent = Party(tel="+1987654321", name="Bob", role="agent")
vcon.add_party(caller)
vcon.add_party(agent)

# Add recording dialog
dialog = Dialog(
    type="recording",
    start=datetime.now(timezone.utc),
    parties=[0, 1],
    mimetype="audio/mp3"
)
vcon.add_dialog(dialog)

# Add lawful basis for consent
vcon.add_lawful_basis_attachment(
    lawful_basis="consent",
    expiration=(datetime.now(timezone.utc) + timedelta(days=365)).isoformat(),
    purpose_grants=[
        {
            "purpose": "recording",
            "granted": True,
            "granted_at": datetime.now(timezone.utc).isoformat()
        },
        {
            "purpose": "transcription",
            "granted": True,
            "granted_at": datetime.now(timezone.utc).isoformat()
        }
    ],
    party_index=0
)

# Add transcription
vcon.add_wtf_transcription_attachment(
    transcript={
        "text": "Hello, I need help with my account.",
        "language": "en",
        "duration": 4.2,
        "confidence": 0.92
    },
    segments=[
        {
            "id": 0,
            "start": 0.0,
            "end": 4.2,
            "text": "Hello, I need help with my account.",
            "confidence": 0.92,
            "speaker": 0
        }
    ],
    metadata={
        "created_at": datetime.now(timezone.utc).isoformat(),
        "processed_at": datetime.now(timezone.utc).isoformat(),
        "provider": "whisper",
        "model": "whisper-1"
    },
    party_index=0,
    dialog_index=0
)

# Add extensions
vcon.add_extension("lawful_basis")
vcon.add_extension("wtf_transcription")

# Validate extensions
validation_results = vcon.validate_extensions()
print("Extension validation:", validation_results)

# Save vCon
vcon.save_to_file("complete_conversation.vcon.json")
print(f"Created complete vCon with extensions: {vcon.uuid}")
```

### Key Features

This library implements the latest vCon specification with the following features:

#### Enhanced Party Information

```python
from vcon import Vcon, Party

# Create a party with enhanced contact information
party = Party(
    tel="+1234567890",
    name="John Doe",
    sip="sip:john@example.com",
    did="did:example:123456789abcdef",
    jCard={
        "fn": "John Doe",
        "tel": "+1234567890",
        "email": "john@example.com"
    },
    timezone="America/New_York"
)
```

#### Extensions and Must-Support

```python
vcon = Vcon.build_new()

# Add extensions used in this vCon
vcon.add_extension("video")
vcon.add_extension("encryption")

# Add extensions that must be supported
vcon.add_must_support("encryption")

print(vcon.get_extensions())  # ['video', 'encryption']
print(vcon.get_must_support())  # ['encryption']
```

#### Enhanced Dialog Support

```python
from vcon import Dialog
from datetime import datetime

# Create dialog with new fields
dialog = Dialog(
    type="text",
    start=datetime.now(),
    parties=[0, 1],
    session_id="session-12345",
    content_hash="c8d3d67f662a787e96e74ccb0a77803138c0f13495a186ccbde495c57c385608",
    application="chat-app",
    message_id="<message-id@example.com>"
)
```

#### Party History Events

```python
from vcon import PartyHistory
from datetime import datetime

# Track party events
history = [
    PartyHistory(0, "join", datetime.now()),
    PartyHistory(1, "join", datetime.now()),
    PartyHistory(0, "hold", datetime.now()),
    PartyHistory(0, "unhold", datetime.now()),
    PartyHistory(1, "drop", datetime.now())
]
```

#### Disposition Values for Incomplete Dialogs

```python
# Create incomplete dialog with proper disposition
incomplete_dialog = Dialog(
    type="incomplete",
    start=datetime.now(),
    parties=[0],
    disposition="no-answer"  # Valid: no-answer, congestion, failed, busy, hung-up, voicemail-no-message
)
```

#### Civic Address Support

```python
from vcon import CivicAddress

# Create civic address with GEOPRIV fields
address = CivicAddress(
    country="US",
    a1="CA",
    a3="San Francisco",
    sts="Market Street",
    hno="123",
    pc="94102"
)

party = Party(name="Jane", civicaddress=address)
```

#### Lawful Basis Extension (Privacy Compliance)

```python
from datetime import datetime, timezone, timedelta

# Add lawful basis for GDPR compliance
vcon.add_lawful_basis_attachment(
    lawful_basis="consent",
    expiration=(datetime.now(timezone.utc) + timedelta(days=365)).isoformat(),
    purpose_grants=[
        {
            "purpose": "recording",
            "granted": True,
            "granted_at": datetime.now(timezone.utc).isoformat()
        },
        {
            "purpose": "analysis",
            "granted": True,
            "granted_at": datetime.now(timezone.utc).isoformat(),
            "conditions": ["anonymized_data_only"]
        }
    ],
    party_index=0
)

# Check permissions
can_record = vcon.check_lawful_basis_permission("recording", party_index=0)
can_analyze = vcon.check_lawful_basis_permission("analysis", party_index=0)

# Find lawful basis attachments
attachments = vcon.find_lawful_basis_attachments(party_index=0)
```

#### WTF Extension (Transcription Support)

```python
# Add standardized transcription
vcon.add_wtf_transcription_attachment(
    transcript={
        "text": "Hello, this is a test transcription.",
        "language": "en",
        "duration": 3.5,
        "confidence": 0.95
    },
    segments=[
        {
            "id": 0,
            "start": 0.0,
            "end": 1.5,
            "text": "Hello, this is",
            "confidence": 0.95,
            "speaker": 0
        },
        {
            "id": 1,
            "start": 1.5,
            "end": 3.5,
            "text": "a test transcription.",
            "confidence": 0.94,
            "speaker": 0
        }
    ],
    metadata={
        "created_at": datetime.now(timezone.utc).isoformat(),
        "processed_at": datetime.now(timezone.utc).isoformat(),
        "provider": "whisper",
        "model": "whisper-1"
    },
    party_index=0,
    dialog_index=0
)

# Export to subtitle formats
attachments = vcon.find_wtf_attachments(party_index=0)
if attachments:
    from vcon.extensions.wtf import WTFAttachment
    wtf_attachment = WTFAttachment.from_dict(attachments[0]["body"])
    
    # Export to SRT format
    srt_content = wtf_attachment.export_to_srt()
    
    # Export to WebVTT format
    vtt_content = wtf_attachment.export_to_vtt()
```

#### Provider Data Conversion

```python
from vcon.extensions.wtf import WhisperAdapter, DeepgramAdapter

# Convert Whisper data to WTF format
whisper_data = {
    "text": "Hello world from Whisper",
    "segments": [
        {
            "start": 0.0,
            "end": 2.0,
            "text": "Hello world from Whisper"
        }
    ]
}

whisper_adapter = WhisperAdapter()
wtf_attachment = whisper_adapter.convert(whisper_data)

# Add to vCon
vcon.add_wtf_transcription_attachment(
    transcript=wtf_attachment.transcript.to_dict(),
    segments=[segment.to_dict() for segment in wtf_attachment.segments],
    metadata=wtf_attachment.metadata.to_dict()
)
```

### Installation

```bash
pip install vcon
```

### Basic Usage

#### Creating a vCon

```python
from vcon import Vcon, Party, Dialog
from datetime import datetime

# Create a new vCon
vcon = Vcon.build_new()

# Add parties
alice = Party(tel="+1234567890", name="Alice", role="caller")
bob = Party(tel="+1987654321", name="Bob", role="agent")

vcon.add_party(alice)
vcon.add_party(bob)

# Add dialog
dialog = Dialog(
    type="text",
    start=datetime.now(),
    parties=[0, 1],
    body="Hello, this is a test message!"
)

vcon.add_dialog(dialog)

# Save to file
vcon.save_to_file("conversation.vcon.json")
```

#### Loading a vCon

```python
# Load from file
vcon = Vcon.load("conversation.vcon.json")

# Load from URL
vcon = Vcon.load("https://example.com/conversation.vcon.json")
```

#### Validation

```python
# Validate a vCon
is_valid, errors = vcon.is_valid()

if is_valid:
    print("vCon is valid")
else:
    print("Validation errors:", errors)

# Validate from file
is_valid, errors = Vcon.validate_file("conversation.vcon.json")
```

### Media Support

#### Audio and Video

```python
# Add audio recording
audio_dialog = Dialog(
    type="recording",
    start=datetime.now(),
    parties=[0, 1],
    url="https://example.com/recording.wav",
    mimetype="audio/x-wav"
)

# Add video with metadata
video_dialog = Dialog(
    type="video",
    start=datetime.now(),
    parties=[0, 1],
    url="https://example.com/video.mp4",
    mimetype="video/mp4",
    resolution="1920x1080",
    frame_rate=30.0,
    codec="H.264"
)
```

#### Supported Media Types

**Audio**: `audio/x-wav`, `audio/x-mp3`, `audio/x-mp4`, `audio/ogg` **Video**: `video/x-mp4`, `video/ogg` **Text**: `text/plain` **Multipart**: `multipart/mixed`

### Security Features

#### Digital Signatures

```python
from cryptography.hazmat.primitives import serialization

# Generate key pair
private_key, public_key = Vcon.generate_key_pair()

# Sign the vCon
vcon.sign(private_key)

# Verify signature
is_valid = vcon.verify(public_key)
```

#### Content Hashing

```python
# Calculate content hash for external files
content_hash = dialog.calculate_content_hash("sha256")

# Verify content integrity
is_valid = dialog.verify_content_hash(expected_hash, "sha256")
```

### Advanced Features

#### Property Handling

```python
# Strict mode - only allow standard properties
vcon = Vcon.load("file.json", property_handling="strict")

# Meta mode - move non-standard properties to meta object
vcon = Vcon.load("file.json", property_handling="meta")

# Default mode - keep all properties
vcon = Vcon.load("file.json", property_handling="default")
```

#### Transfer Dialogs

```python
# Create transfer dialog
transfer_data = {
    "transferee": 0,
    "transferor": 1,
    "transfer_target": 2,
    "original": 0,
    "target_dialog": 1
}

vcon.add_transfer_dialog(
    start=datetime.now(),
    transfer_data=transfer_data,
    parties=[0, 1, 2]
)
```

#### Analysis Data

```python
# Add analysis
vcon.add_analysis(
    type="sentiment",
    dialog=0,
    vendor="example-vendor",
    body={"sentiment": "positive", "confidence": 0.95},
    encoding="json"
)
```

### Specification Compliance

This library implements the latest vCon specification with:

* ✅ All required fields and validation
* ✅ Proper media type support
* ✅ Civic address (GEOPRIV) compliance
* ✅ Party history event tracking
* ✅ Transfer dialog support
* ✅ Content hashing and security
* ✅ Extensions and must\_support
* ✅ Flexible versioning (version field is optional)
* ✅ Backward compatibility
* ✅ **Lawful Basis Extension** - GDPR-compliant privacy management
* ✅ **WTF Extension** - World Transcription Format for standardized speech-to-text
* ✅ **Extension Framework** - Comprehensive validation and processing
* ✅ **Multi-Provider Support** - Whisper, Deepgram, AssemblyAI, and more
* ✅ **Export Capabilities** - SRT and WebVTT subtitle formats

### Testing

Run the test suite:

```bash
pytest tests/
```

All tests pass, covering:

* Basic functionality
* Enhanced vCon features
* Validation and error handling
* Media type support
* Security features
* Flexible versioning
* Backward compatibility
* **Extension Framework** - Lawful Basis and WTF extensions
* **Privacy Compliance** - GDPR-compliant consent management
* **Transcription Support** - Multi-provider transcription handling
* **Provider Adapters** - Data conversion and validation
* **Export Functionality** - SRT and WebVTT export testing

### Extension Framework

The vCon library includes a comprehensive extension framework that allows for standardized implementation of additional functionality:

#### Available Extensions

* **Lawful Basis Extension** - GDPR-compliant privacy management and consent tracking
* **WTF Extension** - World Transcription Format for standardized speech-to-text data

#### Extension Features

* **Validation Framework** - Comprehensive validation for all extension data
* **Processing Framework** - Standardized processing and analysis
* **Provider Adapters** - Automatic conversion from provider-specific formats
* **Export Capabilities** - Multiple export formats (SRT, WebVTT)
* **Permission Management** - Granular permission checking and validation

#### Extension Usage

```python
# Validate all extensions
validation_results = vcon.validate_extensions()

# Process all extensions
processing_results = vcon.process_extensions()

# Check specific permissions
can_record = vcon.check_lawful_basis_permission("recording", party_index=0)

# Export transcriptions
attachments = vcon.find_wtf_attachments(party_index=0)
```

### 0.9.2 quirks and post-processing steps

The library is broadly spec-compliant, but a handful of edges still need manual handling for full `draft-ietf-vcon-vcon-core-02` compliance. Apply these after building the vCon and before serializing:

1. **Set the syntax parameter.** `Vcon.build_new()` does not set the `vcon` syntax param. Write it yourself:
   ```python
   v.vcon_dict["vcon"] = "0.4.0"
   ```
2. **JSON-bodied attachments.** `add_attachment()` rejects `encoding: "json"`. For JSON attachments, append directly to `vcon_dict["attachments"]`:
   ```python
   v.vcon_dict["attachments"].append({
       "purpose": "ticket_metadata",
       "encoding": "json",
       "party": 0, "dialog": 0,
       "body": {"ticket_id": "T-1234", "priority": "high"},
   })
   ```
3. **`subject` has no setter.** Write via the dict:
   ```python
   v.vcon_dict["subject"] = "Refund request"
   ```
4. **WTF helper field rename.** `add_wtf_transcription_attachment()` emits `type: "wtf_transcription"`. The spec uses `purpose:` on attachments (or `type: "transcript"` if placed in `analysis[]`, which is the recommended location). Rename or relocate before serializing — see [WTF Transcription extension](../extensions/wtf-transcription.md) for the recommended pattern.
5. **Lawful Basis: easier to build by hand.** `add_lawful_basis_attachment()` requires model objects for `purpose_grants` and `proof_mechanisms`. Building the attachment dict directly per [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/) and appending to `vcon_dict["attachments"]` is usually simpler — then call `v.add_extension("lawful_basis")`. See [Lawful Basis extension](../extensions/lawful-basis.md).
6. **Drop empty `group` and `redacted`.** `build_new()` may initialize empty `group: []` and `redacted: {}` blocks; the speckit treats `group` as reserved. Delete the empty entries:
   ```python
   v.vcon_dict.pop("group", None)
   if v.vcon_dict.get("redacted") == {}:
       v.vcon_dict.pop("redacted", None)
   ```

### Field-name migration (pre-0.9.1 → 0.9.2)

If you're upgrading from pre-0.9.1 code or working with vCons produced by older libraries, you'll encounter two renamed fields:

| Old name | New name | Where |
|----------|----------|-------|
| `appended` | `amended` | Top-level vCon, mutually exclusive with `redacted` / `group` |
| `must_support` | `critical` (or `must_understand`) | Inside `extensions[]` declarations |

The library reads both forms but writes the new ones. If you have downstream tooling that depends on the old names, update it; both `vcon-mcp` (database migration `20251120150100_field_renames.sql`) and `vcon-server` have already moved.

### Documentation

* **API Reference** - Complete API documentation
* **User Guide** - Comprehensive usage guide
* **LLM Guide** - Guide for AI-assisted development
* **Migration Guide** - Upgrading from older versions

### License

This project is licensed under the MIT License - see the LICENSE file for details.
