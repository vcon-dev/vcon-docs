---
description: >-
  This guide provides essential information for Large Language Models tasked
  with generating vCon adapter code. Follow these patterns and requirements when
  creating adapters that convert conversation da
icon: robot
---

# LLM Guide: Creating vCon Adapters

## Core Requirements

#### Essential Imports

Always include these imports in vCon adapter code:

```python
from abc import ABC, abstractmethod
from typing import Dict, List, Any, Optional, Union
from vcon import Vcon, Party, Dialog
from datetime import datetime, timezone
import json
import base64
import logging
```

#### Base Adapter Pattern

Use this as the foundation for all adapters:

```python
class BaseVconAdapter(ABC):
    """Base class for all vCon adapters."""
    
    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.validation_errors = []
        self.logger = logging.getLogger(self.__class__.__name__)
    
    @abstractmethod
    def extract_data(self, source: Any) -> Dict[str, Any]:
        """Extract raw data from the source system."""
        pass
    
    @abstractmethod
    def transform_to_vcon(self, raw_data: Dict[str, Any]) -> Vcon:
        """Transform raw data into a vCon object."""
        pass
    
    def validate_vcon(self, vcon: Vcon) -> bool:
        """Validate the generated vCon."""
        is_valid, errors = vcon.is_valid()
        self.validation_errors = errors
        return is_valid
    
    def process(self, source: Any) -> Vcon:
        """Main processing pipeline."""
        raw_data = self.extract_data(source)
        vcon = self.transform_to_vcon(raw_data)
        
        if not self.validate_vcon(vcon):
            raise ValueError(f"Invalid vCon generated: {self.validation_errors}")
        
        return vcon
```

### Key Patterns to Follow

#### 1. vCon Creation

Always start with:

```python
def transform_to_vcon(self, raw_data: Dict[str, Any]) -> Vcon:
    vcon = Vcon.build_new()
    
    # Add metadata tags
    vcon.add_tag("source", "your_system_name")
    vcon.add_tag("adapter_version", "1.0")
    
    # Process data...
    return vcon
```

#### 2. Party Processing

Create a mapping between source participants and vCon parties:

```python
# Build participant mapping
participant_map = {}
for i, participant in enumerate(raw_data.get("participants", [])):
    party = Party(
        name=participant.get("name"),
        tel=participant.get("phone"),
        mailto=participant.get("email"),
        role=participant.get("role", "participant"),
        # Add new vCon 0.3.0 fields if available
        sip=participant.get("sip_uri"),
        did=participant.get("decentralized_id"),
        timezone=participant.get("timezone")
    )
    vcon.add_party(party)
    participant_map[participant["id"]] = i
```

#### 3. Dialog Processing

Handle different dialog types:

```python
for message in raw_data.get("messages", []):
    dialog = Dialog(
        type="text",  # or "recording", "video", "transfer", "incomplete"
        start=self.parse_timestamp(message["timestamp"]),
        parties=[participant_map[message["sender_id"]]],
        originator=participant_map[message["sender_id"]],
        mimetype="text/plain",
        body=message["content"],
        # Add new vCon 0.3.0 fields
        session_id=message.get("session_id"),
        application=message.get("app_name"),
        message_id=message.get("message_id")
    )
    vcon.add_dialog(dialog)
```

#### 4. Timestamp Handling

Always convert timestamps to ISO format:

```python
def parse_timestamp(self, timestamp) -> str:
    """Convert various timestamp formats to ISO 8601."""
    if isinstance(timestamp, datetime):
        return timestamp.isoformat()
    elif isinstance(timestamp, str):
        try:
            # Try parsing common formats
            from dateutil import parser
            return parser.parse(timestamp).isoformat()
        except:
            return datetime.now(timezone.utc).isoformat()
    elif isinstance(timestamp, (int, float)):
        # Assume Unix timestamp
        return datetime.fromtimestamp(timestamp, timezone.utc).isoformat()
    else:
        return datetime.now(timezone.utc).isoformat()
```

### Media Handling Patterns

#### Audio/Video Content

Handle media files properly:

```python
def add_media_dialog(self, media_data: Dict[str, Any], parties: List[int]) -> Dialog:
    """Add audio or video dialog."""
    if media_data.get("url"):
        # External media reference
        dialog = Dialog(
            type="recording" if media_data["type"] == "audio" else "video",
            start=self.parse_timestamp(media_data["timestamp"]),
            parties=parties,
            mimetype=media_data.get("mimetype", "audio/wav"),
            url=media_data["url"],
            duration=media_data.get("duration"),
            content_hash=media_data.get("hash")  # New in vCon 0.3.0
        )
    else:
        # Inline media (base64 encoded)
        dialog = Dialog(
            type="recording" if media_data["type"] == "audio" else "video",
            start=self.parse_timestamp(media_data["timestamp"]),
            parties=parties,
            mimetype=media_data.get("mimetype", "audio/wav"),
            body=media_data["base64_content"],
            encoding="base64",
            filename=media_data.get("filename")
        )
    
    return dialog
```

#### Transfer Dialogs

Handle call transfers:

```python
def add_transfer_dialog(self, transfer_data: Dict[str, Any]) -> None:
    """Add transfer dialog for call transfers."""
    vcon.add_transfer_dialog(
        start=self.parse_timestamp(transfer_data["timestamp"]),
        transfer_data={
            "reason": transfer_data.get("reason", "Call transferred"),
            "from": transfer_data.get("from_number"),
            "to": transfer_data.get("to_number"),
            "transfer_target": transfer_data.get("target_party_index"),
            "transferor": transfer_data.get("transferor_party_index"),
            "transferee": transfer_data.get("transferee_party_index")
        },
        parties=transfer_data.get("involved_parties", [])
    )
```

#### Incomplete Dialogs

Handle failed conversations:

```python
def add_incomplete_dialog(self, failed_call: Dict[str, Any]) -> None:
    """Add incomplete dialog for failed calls."""
    # Map common failure reasons to vCon dispositions
    disposition_map = {
        "no_answer": "no-answer",
        "busy": "busy",
        "failed": "failed",
        "hung_up": "hung-up",
        "voicemail": "voicemail-no-message",
        "congestion": "congestion"
    }
    
    disposition = disposition_map.get(
        failed_call.get("reason", "failed").lower(), 
        "failed"
    )
    
    vcon.add_incomplete_dialog(
        start=self.parse_timestamp(failed_call["timestamp"]),
        disposition=disposition,
        details=failed_call.get("details", {}),
        parties=failed_call.get("involved_parties", [])
    )
```

### Error Handling Requirements

#### Robust Data Extraction

Always handle missing or malformed data:

```python
def extract_data(self, source: Any) -> Dict[str, Any]:
    """Extract data with error handling."""
    try:
        if isinstance(source, str):
            # File path
            with open(source, 'r') as f:
                return json.load(f)
        elif isinstance(source, dict):
            # Direct data
            return source
        else:
            raise ValueError(f"Unsupported source type: {type(source)}")
    except Exception as e:
        self.logger.error(f"Failed to extract data: {e}")
        raise
```

#### Validation and Fallbacks

Provide fallbacks for missing required data:

```python
def transform_to_vcon(self, raw_data: Dict[str, Any]) -> Vcon:
    vcon = Vcon.build_new()
    
    # Ensure required fields exist
    if not raw_data.get("participants"):
        # Create a default participant if none exist
        default_party = Party(name="Unknown", role="participant")
        vcon.add_party(default_party)
        participant_map = {"default": 0}
    else:
        participant_map = self.process_participants(raw_data, vcon)
    
    # Handle missing timestamps
    default_timestamp = datetime.now(timezone.utc).isoformat()
    
    # Process messages with fallbacks
    for message in raw_data.get("messages", []):
        dialog = Dialog(
            type="text",
            start=self.parse_timestamp(message.get("timestamp", default_timestamp)),
            parties=[participant_map.get(message.get("sender_id"), 0)],
            originator=participant_map.get(message.get("sender_id"), 0),
            mimetype="text/plain",
            body=message.get("content", "")
        )
        vcon.add_dialog(dialog)
    
    return vcon
```

### Common Adapter Templates

#### Chat System Adapter

```python
class ChatSystemAdapter(BaseVconAdapter):
    """Template for chat/messaging systems."""
    
    def extract_data(self, chat_file: str) -> Dict[str, Any]:
        with open(chat_file, 'r') as f:
            return json.load(f)
    
    def transform_to_vcon(self, raw_data: Dict[str, Any]) -> Vcon:
        vcon = Vcon.build_new()
        vcon.add_tag("source", "chat_system")
        
        # Process participants
        participant_map = {}
        for i, user in enumerate(raw_data.get("users", [])):
            party = Party(
                name=user.get("display_name"),
                mailto=user.get("email"),
                role="participant"
            )
            vcon.add_party(party)
            participant_map[user["id"]] = i
        
        # Process messages
        for msg in raw_data.get("messages", []):
            dialog = Dialog(
                type="text",
                start=self.parse_timestamp(msg["timestamp"]),
                parties=[participant_map[msg["user_id"]]],
                originator=participant_map[msg["user_id"]],
                mimetype="text/plain",
                body=msg["text"]
            )
            vcon.add_dialog(dialog)
        
        return vcon
```

#### Call Center Adapter

```python
class CallCenterAdapter(BaseVconAdapter):
    """Template for call center systems."""
    
    def extract_data(self, call_record: Dict[str, Any]) -> Dict[str, Any]:
        return call_record
    
    def transform_to_vcon(self, raw_data: Dict[str, Any]) -> Vcon:
        vcon = Vcon.build_new()
        vcon.add_tag("source", "call_center")
        vcon.add_tag("call_id", raw_data.get("call_id"))
        
        # Add caller
        caller = Party(
            tel=raw_data.get("caller_number"),
            name=raw_data.get("caller_name"),
            role="caller"
        )
        vcon.add_party(caller)
        
        # Add agent
        agent = Party(
            tel=raw_data.get("agent_extension"),
            name=raw_data.get("agent_name"),
            role="agent"
        )
        vcon.add_party(agent)
        
        # Add call recording if available
        if raw_data.get("recording_url"):
            recording_dialog = Dialog(
                type="recording",
                start=self.parse_timestamp(raw_data["start_time"]),
                parties=[0, 1],
                mimetype="audio/wav",
                url=raw_data["recording_url"],
                duration=raw_data.get("duration")
            )
            vcon.add_dialog(recording_dialog)
        
        # Add transcript if available
        if raw_data.get("transcript"):
            for entry in raw_data["transcript"]:
                dialog = Dialog(
                    type="text",
                    start=self.parse_timestamp(entry["timestamp"]),
                    parties=[entry.get("speaker_id", 0)],
                    originator=entry.get("speaker_id", 0),
                    mimetype="text/plain",
                    body=entry["text"]
                )
                vcon.add_dialog(dialog)
        
        return vcon
```

### Critical Requirements

#### 1. Always Validate

```python
# At the end of transform_to_vcon
is_valid, errors = vcon.is_valid()
if not is_valid:
    self.logger.error(f"Generated invalid vCon: {errors}")
    # Either fix the issues or raise an exception
```

#### 2. Handle All Dialog Types

Support these dialog types based on source data:

* `"text"` - Text messages, chat, transcripts
* `"recording"` - Audio recordings
* `"video"` - Video calls/recordings
* `"transfer"` - Call transfers
* `"incomplete"` - Failed/incomplete calls

#### 3. Use Proper MIME Types

Use these MIME types:

* Text: `"text/plain"`
* Audio: `"audio/wav"`, `"audio/mp3"`, `"audio/x-wav"`, `"audio/x-mp3"`
* Video: `"video/mp4"`, `"video/webm"`, `"video/x-mp4"`
* Email: `"message/rfc822"`

#### 4. Include Extensions and Must-Support (vCon 0.3.0)

```python
# Add extensions if using advanced features
if using_video:
    vcon.add_extension("video")
if using_encryption:
    vcon.add_extension("encryption")
    vcon.add_must_support("encryption")
```

### Testing Pattern

Always include this test structure:

```python
def test_adapter():
    """Test the adapter with sample data."""
    adapter = YourAdapter({})
    
    sample_data = {
        # Your test data structure
    }
    
    vcon = adapter.process(sample_data)
    is_valid, errors = vcon.is_valid()
    
    assert is_valid, f"Invalid vCon: {errors}"
    assert len(vcon.parties) > 0, "No parties found"
    assert len(vcon.dialog) > 0, "No dialog found"
    
    return vcon
```

### Key Considerations for LLMs

1. **Always use the base adapter pattern** - don't create adapters from scratch
2. **Handle missing data gracefully** - provide defaults and fallbacks
3. **Validate all timestamps** - convert to ISO 8601 format
4. **Map participant IDs correctly** - maintain consistent party references
5. **Include proper error handling** - log errors and provide meaningful messages
6. **Use appropriate dialog types** - match the source content type
7. **Add relevant metadata** - use tags and extensions appropriately
8. **Test the generated vCon** - always validate before returning

When generating adapter code, focus on the specific source system requirements while following these patterns and ensuring compliance with the vCon specification.
