---
description: The Python vCon library
---

# 🐰 Quickstart

This tutorial will guide you through creating a vCon (Virtual Conversation) object using the vCon API. We'll cover creating a new vCon, adding parties and dialogs, attaching metadata and analysis, and finally signing and verifying the vCon.

### Prerequisites

Before you begin, make sure you have the vCon library installed in your Python environment.

```bash
pip install vcon  
```

### Step 1: Import Required Modules

First, let's import the necessary modules:

```python
import datetime
from vcon import Vcon
from vcon.party import Party
from vcon.dialog import Dialog
from vcon.party import PartyHistory
```

### Step 2: Create a New vCon Object

To start, we'll create a new vCon object:

```python
vcon = Vcon.build_new()
```

### Step 3: Add Parties

Next, we'll add two parties to our vCon: a caller and an agent.

```python
caller = Party(tel="+1234567890", name="Alice", role="caller")
agent = Party(tel="+1987654321", name="Bob", role="agent")
vcon.add_party(caller)
vcon.add_party(agent)
```

### Step 4: Add Dialogs

Now, let's add some dialog to our vCon.

```python
start_time = datetime.datetime.now(datetime.timezone.utc)
dialog = Dialog(
    type="text",
    start=start_time.isoformat(),
    parties=[0, 1],  # Indices of the parties in the vCon
    originator=0,  # The caller (Alice) is the originator
    mimetype="text/plain",
    body="Hello, I need help with my account."
)
vcon.add_dialog(dialog)
```

Note that we're using ISO format strings for the datetime values and including UTC timezone information.

### Step 5: Add Metadata

We can add metadata to our vCon using tags:

```python
vcon.add_tag("customer_id", "12345")
vcon.add_tag("interaction_id", "INT-001")
```

### Step 6: Add an Attachment

Let's add a transcript as an attachment:

```python
transcript = "Alice: Hello, I need help with my account.\nBob: Certainly! I'd be happy to help. Can you please provide your account number?"
vcon.add_attachment(body=transcript, type="transcript", encoding="none")
```

### Step 7: Add Analysis

We can also add analysis data to our vCon. Here's an example of adding sentiment analysis:

```python
sentiment_analysis = {
    "overall_sentiment": "positive",
    "customer_sentiment": "neutral",
    "agent_sentiment": "positive"
}
vcon.add_analysis(
    type="sentiment",
    dialog=[0, 1],  # Indices of the dialogs analyzed
    vendor="SentimentAnalyzer",
    body=sentiment_analysis,
    encoding="none"
)
```

### Step 8: Sign and Verify the vCon

Finally, let's generate a key pair, sign the vCon, and verify the signature:

```python
# Generate a key pair for signing
private_key, public_key = Vcon.generate_key_pair()

# Sign the vCon
vcon.sign(private_key)

# Verify the signature
is_valid = vcon.verify(public_key)
print(f"Signature is valid: {is_valid}")
```

### Step 9: Serialize to JSON

To see the final result, we can serialize our vCon to JSON:

```python
print(vcon.to_json())
```

### Complete Example

Here's the complete example putting all these steps together:

```python
import datetime
from vcon import Vcon
from vcon.party import Party
from vcon.dialog import Dialog
from vcon.party import PartyHistory

def main():
    # Create a new vCon object
    vcon = Vcon.build_new()

    # Add parties
    caller = Party(tel="+1234567890", name="Alice", role="caller")
    agent = Party(tel="+1987654321", name="Bob", role="agent")
    vcon.add_party(caller)
    vcon.add_party(agent)

    # Add a dialog
    start_time = datetime.datetime.now(datetime.timezone.utc)
    dialog = Dialog(
        type="text",
        start=start_time.isoformat(),
        parties=[0, 1],  # Indices of the parties in the vCon
        originator=0,  # The caller (Alice) is the originator
        mimetype="text/plain",
        body="Hello, I need help with my account."
    )
    vcon.add_dialog(dialog)

    # Add a response from the agent
    response_time = start_time + datetime.timedelta(minutes=1)
    response = Dialog(
        type="text",
        start=response_time.isoformat(),
        parties=[0, 1],
        originator=1,  # The agent (Bob) is the originator
        mimetype="text/plain",
        body="Certainly! I'd be happy to help. Can you please provide your account number?"
    )
    vcon.add_dialog(response)

    # Add some metadata
    vcon.add_tag("customer_id", "12345")
    vcon.add_tag("interaction_id", "INT-001")

    # Add an attachment (e.g., a transcript)
    transcript = "Alice: Hello, I need help with my account.\nBob: Certainly! I'd be happy to help. Can you please provide your account number?"
    vcon.add_attachment(body=transcript, type="transcript", encoding="none")

    # Add some analysis (e.g., sentiment analysis)
    sentiment_analysis = {
        "overall_sentiment": "positive",
        "customer_sentiment": "neutral",
        "agent_sentiment": "positive"
    }
    vcon.add_analysis(
        type="sentiment",
        dialog=[0, 1],  # Indices of the dialogs analyzed
        vendor="SentimentAnalyzer",
        body=sentiment_analysis,
        encoding="none"
    )

    # Generate a key pair for signing
    private_key, public_key = Vcon.generate_key_pair()

    # Sign the vCon
    vcon.sign(private_key)

    # Verify the signature
    is_valid = vcon.verify(public_key)
    print(f"Signature is valid: {is_valid}")

    # Print the vCon as JSON
    print(vcon.to_json())

if __name__ == "__main__":
    main()
```

##

### Contributing

Contributions to the vCon library are welcome! Please submit pull requests or open issues on the GitHub repository.

### License

This project is licensed under the MIT License:

MIT License

Copyright (c) 2023 Thomas McCarthy-Howe

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

### Contact

For questions or support, please contact:

Thomas McCarthy-Howe Email: ghostofbasho@gmail.com
