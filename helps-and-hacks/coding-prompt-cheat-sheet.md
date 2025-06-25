---
description: For when you need to tell cursor or replit what a vCon is in a prompt...
---

# Coding Prompt Cheat Sheet

## vCon (Virtual Conversation) Standard - LLM Context

### What is a vCon?

A vCon is a standardized JSON container for storing and exchanging real-time human conversation data. It supports multiple communication types: phone calls, video conferences, SMS, MMS, emails, web chat, and more. vCons enable consistent storage, analysis, and interchange of conversational data across different platforms and services.

### Core Structure

Every vCon has exactly 5 main sections:

1. **metadata** - conversation context and identifiers
2. **parties** - participant information
3. **dialog** - actual conversation content
4. **analysis** - derived insights (transcripts, sentiment, etc.)
5. **attachments** - supplemental files

### vCon States

* **unsigned** - initial/intermediate state during collection
* **signed** - verified with JWS digital signature for immutability
* **encrypted** - secured with JWE for sensitive data

### Complete JSON Schema

#### Top-Level vCon Object Properties

```json
{
  "vcon": "0.0.2",                    // REQUIRED: syntax version
  "uuid": "string",                   // REQUIRED: globally unique identifier 
  "created_at": "Date",               // REQUIRED: creation timestamp (RFC3339)
  "updated_at": "Date",               // OPTIONAL: last modification timestamp
  "subject": "string",                // OPTIONAL: conversation topic/subject
  "parties": [],                      // REQUIRED: array of Party objects
  "dialog": [],                       // OPTIONAL: array of Dialog objects
  "analysis": [],                     // OPTIONAL: array of Analysis objects  
  "attachments": [],                  // OPTIONAL: array of Attachment objects
  "redacted": {},                     // OPTIONAL: Redacted object (mutually exclusive with appended/group)
  "appended": {},                     // OPTIONAL: Appended object (mutually exclusive with redacted/group)
  "group": []                         // OPTIONAL: array of Group objects (mutually exclusive with redacted/appended)
}
```

#### Party Object Properties

```json
{
  "tel": "string",                    // OPTIONAL: telephone number (E.164 format preferred)
  "stir": "string",                   // OPTIONAL: STIR PASSporT in JWS Compact form
  "mailto": "string",                 // OPTIONAL: email address
  "name": "string",                   // OPTIONAL: participant name
  "validation": "string",             // OPTIONAL: identity validation method used
  "jcard": "object",                  // OPTIONAL: jCard object for contact info
  "gmlpos": "string",                 // OPTIONAL: GML position (lat/long)
  "civicaddress": {                   // OPTIONAL: civic address object
    "country": "string",              // OPTIONAL: country code
    "a1": "string",                   // OPTIONAL: national subdivision (state/province)
    "a2": "string",                   // OPTIONAL: county/parish/district
    "a3": "string",                   // OPTIONAL: city/township
    "a4": "string",                   // OPTIONAL: city division/borough
    "a5": "string",                   // OPTIONAL: neighborhood/block
    "a6": "string",                   // OPTIONAL: street/group of streets
    "prd": "string",                  // OPTIONAL: leading street direction
    "pod": "string",                  // OPTIONAL: trailing street suffix
    "sts": "string",                  // OPTIONAL: street suffix
    "hno": "string",                  // OPTIONAL: house number
    "hns": "string",                  // OPTIONAL: house number suffix
    "lmk": "string",                  // OPTIONAL: landmark
    "loc": "string",                  // OPTIONAL: additional location info
    "flr": "string",                  // OPTIONAL: floor
    "nam": "string",                  // OPTIONAL: name/description
    "pc": "string"                    // OPTIONAL: postal code
  },
  "timezone": "string",               // OPTIONAL: timezone identifier
  "uuid": "string",                   // OPTIONAL: unique participant identifier
  "role": "string",                   // OPTIONAL: participant role (agent, customer, supervisor, etc.)
  "contact_list": "string"            // OPTIONAL: reference to contact list
}
```

#### Dialog Object Properties

```json
{
  "type": "string",                   // REQUIRED: "recording", "text", "transfer", or "incomplete"
  "start": "Date",                    // REQUIRED: start time (RFC3339)
  "duration": "number",               // OPTIONAL: duration in seconds (UnsignedInt or UnsignedFloat)
  "parties": [],                      // REQUIRED: array of party indices or arrays for multi-channel
  "originator": "number",             // OPTIONAL: index of originating party (if not first in parties)
  "mediatype": "string",              // OPTIONAL: MIME type (required for inline, optional if in HTTP header)
  "filename": "string",               // OPTIONAL: original filename
  
  // Content (for types other than "incomplete" and "transfer")
  "body": "string",                   // OPTIONAL: inline content (mutually exclusive with url)
  "encoding": "string",               // REQUIRED with body: "base64url", "json", or "none"
  "url": "string",                    // OPTIONAL: external reference (mutually exclusive with body)
  "content_hash": "string|string[]",  // REQUIRED with url: SHA-512 hash for integrity
  
  // Additional properties
  "disposition": "string",            // REQUIRED for "incomplete" type: reason for failure
  "party_history": [],               // OPTIONAL: array of party join/leave events
  "campaign": "string",               // OPTIONAL: campaign identifier
  "interaction_type": "string",       // OPTIONAL: type of interaction
  "interaction_id": "string",         // OPTIONAL: interaction identifier
  "skill": "string",                  // OPTIONAL: required skill for handling
  "application": "string",            // OPTIONAL: application/platform used
  "message_id": "string",             // OPTIONAL: unique message identifier
  
  // Transfer-specific properties (only for "transfer" type)
  "transferee": "number",             // Party index for transferee role
  "transferor": "number",             // Party index for transferor role  
  "transfer_target": "number",        // Party index for transfer target role
  "original": "number",               // Dialog index for original conversation
  "consultation": "number",           // Dialog index for consultation (optional)
  "target_dialog": "number"           // Dialog index for target conversation
}
```

#### Analysis Object Properties

```json
{
  "type": "string",                   // REQUIRED: "summary", "transcript", "translation", "sentiment", "tts"
  "dialog": "number|number[]",        // OPTIONAL: index(es) of related dialog objects
  "mediatype": "string",              // OPTIONAL: MIME type of analysis data
  "filename": "string",               // OPTIONAL: filename for analysis data
  "vendor": "string",                 // OPTIONAL: vendor/product name that generated analysis
  "product": "string",                // OPTIONAL: specific product name
  "schema": "string",                 // OPTIONAL: data format/schema identifier
  
  // Content
  "body": "string",                   // OPTIONAL: inline analysis data (mutually exclusive with url)
  "encoding": "string",               // REQUIRED with body: encoding method
  "url": "string",                    // OPTIONAL: external reference (mutually exclusive with body)
  "content_hash": "string|string[]"   // REQUIRED with url: integrity hash
}
```

#### Attachment Object Properties

```json
{
  "type": "string",                   // OPTIONAL: semantic type of attachment
  "purpose": "string",                // OPTIONAL: purpose of attachment
  "start": "Date",                    // REQUIRED: timestamp when attachment was exchanged
  "party": "number",                  // REQUIRED: index of party who contributed attachment
  "mediatype": "string",              // OPTIONAL: MIME type
  "filename": "string",               // OPTIONAL: original filename
  "dialog": "number",                 // REQUIRED: index of related dialog
  
  // Content
  "body": "string",                   // OPTIONAL: inline attachment data (mutually exclusive with url)
  "encoding": "string",               // REQUIRED with body: encoding method
  "url": "string",                    // OPTIONAL: external reference (mutually exclusive with body)
  "content_hash": "string|string[]"   // REQUIRED with url: integrity hash
}
```

#### Redacted Object Properties

```json
{
  "uuid": "string",                   // REQUIRED: UUID of unredacted version
  "type": "string",                   // REQUIRED: type of redaction performed
  "body": "string",                   // OPTIONAL: inline unredacted vCon (encrypted)
  "encoding": "string",               // REQUIRED with body: encoding method
  "url": "string",                    // OPTIONAL: external reference to unredacted vCon
  "content_hash": "string|string[]"   // REQUIRED with url: integrity hash
}
```

#### Appended Object Properties

```json
{
  "uuid": "string",                   // OPTIONAL: UUID of original vCon version
  "body": "string",                   // OPTIONAL: inline original vCon
  "encoding": "string",               // REQUIRED with body: encoding method
  "url": "string",                    // OPTIONAL: external reference to original vCon
  "content_hash": "string|string[]"   // OPTIONAL with url: integrity hash
}
```

#### Group Object Properties

```json
{
  "uuid": "string",                   // REQUIRED: UUID of vCon to aggregate
  "body": "string",                   // OPTIONAL: inline vCon (JSON form)
  "encoding": "string",               // REQUIRED with body: must be "json"
  "url": "string",                    // OPTIONAL: external reference to vCon
  "content_hash": "string|string[]"   // REQUIRED with url: integrity hash
}
```

#### Party History Object Properties (for Dialog.party\_history)

```json
{
  "party": "number",                  // REQUIRED: index of party
  "event": "string",                  // REQUIRED: "join", "drop", "hold", "unhold", "mute", "unmute"
  "time": "Date"                      // REQUIRED: timestamp of event
}
```

### Security Features

* **JWS Signing** - ensures integrity and authenticity using RS256 (recommended)
* **JWE Encryption** - protects sensitive content using RSA-OAEP + A256CBC-HS512 (recommended)
* **Content Hashing** - SHA-512 hashes for external file integrity (mandatory for external refs)
* **Versioning** - maintains history of changes and redactions via uuid references

### Signed vCon Structure (JWS)

```json
{
  "payload": "string",                // Base64url encoded unsigned vCon
  "signatures": [{                    // Array of signature objects
    "header": {                       // Unprotected header
      "alg": "RS256",                 // SHOULD be RS256
      "x5c": ["string"],              // REQUIRED: certificate chain OR x5u
      "x5u": "string",                // REQUIRED: cert chain URL OR x5c
      "uuid": "string"                // SHOULD be provided: vCon UUID for convenience
    },
    "protected": "string",            // Base64url encoded protected header
    "signature": "string"             // Base64url encoded signature
  }]
}
```

### Encrypted vCon Structure (JWE)

```json
{
  "unprotected": {                    // Unprotected header
    "cty": "application/vcon+json",   // SHOULD be application/vcon+json
    "enc": "A256CBC-HS512",           // SHOULD be A256CBC-HS512
    "uuid": "string"                  // SHOULD be provided: vCon UUID
  },
  "recipients": [{                    // Array of recipient objects
    "header": {                       // Per-recipient header
      "alg": "RSA-OAEP"               // SHOULD be RSA-OAEP
    },
    "encrypted_key": "string"         // Base64url encoded encrypted key
  }],
  "iv": "string",                     // Base64url encoded initialization vector
  "ciphertext": "string",             // Base64url encoded encrypted signed vCon
  "tag": "string"                     // Base64url encoded authentication tag
}
```

### Common Media Types

#### Dialog

* `text/plain` - plain text messages
* `audio/x-wav` - WAV audio files
* `audio/x-mp3` - MP3 audio files
* `audio/x-mp4` - MP4 audio files
* `audio/ogg` - OGG audio files
* `video/x-mp4` - MP4 video files
* `video/ogg` - OGG video files
* `multipart/mixed` - multipart content (emails)

#### Content Encoding Options

* `base64url` - Base64url encoded binary data
* `json` - Valid JSON object
* `none` - Valid JSON string, no encoding needed

### Implementation Guidelines

* Follow JSON schema strictly for compliance
* Use proper timestamps (RFC3339/ISO 8601 format)
* Ensure UUIDs are globally unique (prefer version 8 with domain-based generation)
* Implement proper signing/encryption for production use
* Maintain referential integrity between sections
* Always use HTTPS for external file references
* Validate content hashes for external files using SHA-512

### Python Development Notes

* Use `uuid` library for generating UUIDs (version 8 recommended)
* `datetime.isoformat()` for RFC3339 timestamps
* `json` module for serialization/deserialization
* `cryptography` library for JWS/JWE operations
* `hashlib` for SHA-512 content hash generation
* Validate against vCon JSON schema before processing
* Use `requests` with SSL verification for external file retrieval

### Key Considerations

* vCons can reference previous versions (redaction/append history via uuid)
* Media content can be embedded (body + encoding) or referenced externally (url + content\_hash)
* Privacy and compliance requirements vary by jurisdiction
* Large media files should typically be stored as external references
* Ensure proper escaping of JSON content in dialog sections
* Support both single-channel and multi-channel audio recordings
* Handle participant join/leave events in party\_history for complex conversations
* Maintain chain of custody through signing and encryption across security domains

### Validation Rules

* At most one of: redacted, appended, or group parameters in top-level object
* Dialog objects of type "incomplete" or "transfer" MUST NOT have body/url content
* Dialog objects of other types SHOULD have body+encoding OR url+content\_hash
* External references (url) MUST include content\_hash for integrity
* Signed vCons MUST include x5c OR x5u in header for certificate chain
* Party indices in dialog.parties must reference valid parties array elements
* Dialog indices in analysis.dialog must reference valid dialog array elements

### UUID Generation (Version 8 Recommended)

```python
# Recommended UUID generation approach
import hashlib
import uuid
from datetime import datetime

def generate_vcon_uuid(domain="example.com"):
    """Generate version 8 UUID for vCon with domain-based uniqueness"""
    timestamp = int(datetime.utcnow().timestamp() * 1000)  # milliseconds
    domain_hash = hashlib.sha1(domain.encode()).digest()[:8]  # 62 bits
    
    # Construct version 8 UUID (implementation details vary)
    # Use standard uuid library with custom generation
    return str(uuid.uuid4())  # Fallback to uuid4 if version 8 not available
```
