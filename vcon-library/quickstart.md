---
description: The Python vCon library
---

# 🐰 Quickstart



Get up and running with the vCon library in minutes! This guide will walk you through the essential features for working with Virtual Conversation objects.

### What is vCon?

vCon (Virtual Conversation) is a standardized format for representing conversations and related metadata. The vCon library provides a complete Python implementation of the vCon 0.3.0 specification, supporting:

* **Conversation Management**: Parties, dialogs, attachments, and analysis
* **Contact Information**: Multiple contact methods (tel, email, SIP, DID)
* **Media Support**: Audio, video, text, and image formats
* **Security**: Digital signatures and content hashing
* **Extensibility**: Extensions and must\_support fields

### Installation

#### Option 1: Install from PyPI (Recommended)

```bash
pip install vcon
```

#### Option 2: Install from Source

```bash
git clone https://github.com/vcon-dev/vcon-lib.git
cd vcon-lib
pip install -e .
```

#### Option 3: Install with Optional Dependencies

For image processing support (Pillow, PyPDF):

```bash
pip install vcon[image]
```

### Quick Start Examples

#### 1. Create Your First vCon

```python
from vcon import Vcon
from vcon.party import Party
from vcon.dialog import Dialog
from datetime import datetime

# Create a new vCon object
vcon = Vcon.build_new()

# Add parties to the conversation
caller = Party(
    tel="+1234567890",
    name="Alice",
    role="caller"
)

agent = Party(
    tel="+1987654321", 
    name="Bob",
    role="agent"
)

vcon.add_party(caller)
vcon.add_party(agent)

# Add a text dialog
dialog = Dialog(
    type="text",
    start=datetime.now().isoformat(),
    parties=[0, 1],  # References to party indices
    originator=0,    # Alice is the originator
    mimetype="text/plain",
    body="Hello, I need help with my account."
)

vcon.add_dialog(dialog)

# Save to file
vcon.save_to_file("my_conversation.vcon.json")
print("vCon created successfully!")
```

#### 2. Load and Read a vCon

```python
from vcon import Vcon

# Load from file
vcon = Vcon.load("my_conversation.vcon.json")

# Load from URL
# vcon = Vcon.load("https://example.com/conversation.vcon.json")

# Access conversation data
print(f"vCon UUID: {vcon.uuid}")
print(f"Created: {vcon.created_at}")
print(f"Parties: {len(vcon.parties)}")
print(f"Dialogs: {len(vcon.dialog)}")

# Iterate through parties
for i, party in enumerate(vcon.parties):
    print(f"Party {i}: {party.name} ({party.tel})")

# Iterate through dialogs
for i, dialog in enumerate(vcon.dialog):
    print(f"Dialog {i}: {dialog.type} - {dialog.body[:50]}...")
```

#### 3. Add Audio Content

```python
import base64

# Read audio file and encode as base64
with open("recording.mp3", "rb") as audio_file:
    audio_data = base64.b64encode(audio_file.read()).decode("utf-8")

# Create audio dialog
audio_dialog = Dialog(
    type="recording",
    start=datetime.now().isoformat(),
    parties=[0, 1],
    originator=0,
    mimetype="audio/mp3",
    body=audio_data,
    encoding="base64",
    filename="recording.mp3"
)

vcon.add_dialog(audio_dialog)
```

#### 4. Add Analysis Data

```python
# Add sentiment analysis
sentiment_data = {
    "overall_sentiment": "positive",
    "confidence": 0.85,
    "emotions": ["satisfied", "helpful"]
}

vcon.add_analysis(
    type="sentiment",
    dialog=[0, 1],  # Analyze first two dialogs
    vendor="SentimentAnalyzer",
    body=sentiment_data,
    encoding="json"
)

# Add transcription
transcript_data = {
    "text": "Hello, I need help with my account. Certainly! I'd be happy to help.",
    "confidence": 0.92,
    "language": "en-US"
}

vcon.add_analysis(
    type="transcription",
    dialog=[2],  # Analyze the audio dialog
    vendor="SpeechToText",
    body=transcript_data,
    encoding="json"
)
```

#### 5. Add Attachments

```python
# Add a transcript as attachment
transcript = """
Alice: Hello, I need help with my account.
Bob: Certainly! I'd be happy to help. Can you please provide your account number?
[Audio conversation follows...]
"""

vcon.add_attachment(
    body=transcript,
    type="transcript",
    encoding="none"
)

# Add metadata tags
vcon.add_tag("customer_id", "12345")
vcon.add_tag("interaction_type", "support_call")
vcon.add_tag("priority", "high")
```

#### 6. Security Features

```python
# Generate key pair for digital signatures
private_key, public_key = Vcon.generate_key_pair()

# Sign the vCon
vcon.sign(private_key)

# Verify signature
is_valid = vcon.verify(public_key)
print(f"Signature valid: {is_valid}")

# Calculate content hash for integrity verification
content_hash = vcon.calculate_content_hash("sha256")
print(f"Content hash: {content_hash}")
```

#### 7. Advanced Features

**Extensions and Must-Support**

```python
# Add extensions used in this vCon
vcon.add_extension("video")
vcon.add_extension("encryption")

# Add extensions that must be supported
vcon.add_must_support("encryption")

print(f"Extensions: {vcon.get_extensions()}")
print(f"Must support: {vcon.get_must_support()}")
```

**Civic Address Support**

```python
from vcon.civic_address import CivicAddress

# Create civic address
address = CivicAddress(
    country="US",
    a1="CA",
    a3="San Francisco", 
    sts="Market Street",
    hno="123",
    pc="94102"
)

# Add to party
party_with_address = Party(
    name="Jane",
    tel="+1555123456",
    civicaddress=address
)
```

**Party History Events**

```python
from vcon.party import PartyHistory

# Track party events
history = [
    PartyHistory(0, "join", datetime.now()),
    PartyHistory(1, "join", datetime.now()),
    PartyHistory(0, "hold", datetime.now()),
    PartyHistory(0, "unhold", datetime.now()),
    PartyHistory(1, "drop", datetime.now())
]

# Add to dialog
dialog_with_history = Dialog(
    type="recording",
    start=datetime.now(),
    parties=[0, 1],
    party_history=history
)
```

### Validation

```python
# Validate a vCon
is_valid, errors = vcon.is_valid()

if is_valid:
    print("✅ vCon is valid")
else:
    print("❌ Validation errors:")
    for error in errors:
        print(f"  - {error}")

# Validate from file
is_valid, errors = Vcon.validate_file("my_conversation.vcon.json")
```

### Property Handling Modes

```python
# Strict mode - only allow standard properties
vcon = Vcon.load("file.json", property_handling="strict")

# Meta mode - move non-standard properties to meta object  
vcon = Vcon.load("file.json", property_handling="meta")

# Default mode - keep all properties
vcon = Vcon.load("file.json", property_handling="default")
```

### HTTP Operations

```python
# Post vCon to a server
response = vcon.post_to_url(
    "https://api.example.com/vcon",
    headers={
        'Authorization': 'Bearer your-token',
        'Content-Type': 'application/json'
    }
)

if response.status_code == 200:
    print("Successfully posted vCon")
else:
    print(f"Error: {response.status_code}")
```

### Complete Example

Here's a complete example that demonstrates most features:

```python
from vcon import Vcon
from vcon.party import Party
from vcon.dialog import Dialog
from datetime import datetime
import base64

def create_complete_vcon():
    # Create vCon
    vcon = Vcon.build_new()
    
    # Add parties
    caller = Party(
        tel="+1234567890",
        name="Alice",
        role="caller",
        timezone="America/New_York"
    )
    
    agent = Party(
        tel="+1987654321",
        name="Bob", 
        role="agent",
        timezone="America/New_York"
    )
    
    vcon.add_party(caller)
    vcon.add_party(agent)
    
    # Add text dialog
    text_dialog = Dialog(
        type="text",
        start=datetime.now().isoformat(),
        parties=[0, 1],
        originator=0,
        mimetype="text/plain",
        body="Hello, I need help with my account."
    )
    vcon.add_dialog(text_dialog)
    
    # Add response
    response_dialog = Dialog(
        type="text", 
        start=datetime.now().isoformat(),
        parties=[0, 1],
        originator=1,
        mimetype="text/plain",
        body="Certainly! I'd be happy to help. Can you provide your account number?"
    )
    vcon.add_dialog(response_dialog)
    
    # Add metadata
    vcon.add_tag("customer_id", "12345")
    vcon.add_tag("interaction_type", "support")
    
    # Add analysis
    vcon.add_analysis(
        type="sentiment",
        dialog=[0, 1],
        vendor="SentimentAnalyzer", 
        body={"sentiment": "positive", "confidence": 0.85},
        encoding="json"
    )
    
    # Add attachment
    vcon.add_attachment(
        body="Full conversation transcript...",
        type="transcript",
        encoding="none"
    )
    
    # Add extensions
    vcon.add_extension("sentiment_analysis")
    vcon.add_must_support("encryption")
    
    # Sign the vCon
    private_key, public_key = Vcon.generate_key_pair()
    vcon.sign(private_key)
    
    # Save to file
    vcon.save_to_file("complete_example.vcon.json")
    
    # Verify signature
    is_valid = vcon.verify(public_key)
    print(f"vCon created and signed. Signature valid: {is_valid}")
    
    return vcon

# Run the example
if __name__ == "__main__":
    vcon = create_complete_vcon()
    print("Complete vCon example created successfully!")
```

### Next Steps

1. **Explore the API**: Check out the full API documentation
2. **Run Tests**: `pytest tests/` to see all available functionality
3. **Read the Specification**: Review the vCon 0.3.0 specification
4. **Check Examples**: Look at the `samples/` directory for more examples

### Common Use Cases

* **Call Centers**: Store and analyze customer interactions
* **Contact Centers**: Track agent performance and customer satisfaction
* **Compliance**: Maintain records for regulatory requirements
* **Analytics**: Extract insights from conversation data
* **Integration**: Exchange conversation data between systems

### Getting Help

* **Documentation**: Full documentation
* **Issues**: [GitHub Issues](https://github.com/vcon-dev/vcon-lib/issues)
* **Examples**: Check the `samples/` directory
* **Tests**: Run `pytest tests/` to see usage examples

### Requirements

* Python 3.12+
* See `requirements.txt` for full dependency list
* Optional: Pillow and PyPDF for image processing support

Happy coding with vCon! ��
