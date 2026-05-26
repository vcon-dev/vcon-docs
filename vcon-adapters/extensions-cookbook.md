---
description: >-
  Worked examples for the extensions adapters use most — WTF transcription,
  lawful basis, SIP signaling, agent session.
---

# 📚 Extensions Cookbook

The vCon core spec is intentionally small. Most of what adapters actually care about — transcripts, recording consent, SIP signaling provenance, AI-agent session tracking — lives in **extensions**. This page is a recipe book for the four extensions adapters use in practice.

Each recipe shows the exact `vcon` library call, the resulting JSON shape, and links to the corresponding extension page for the full spec.

**Spec target:** [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), syntax `"0.4.0"`. Library: [`vcon`](https://pypi.org/project/vcon/) ≥0.9.4.

***

## WTF Transcription

📄 **Spec:** [WTF Transcription Extension](../extensions/wtf-transcription.md) · [`draft-howe-vcon-wtf-extension`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf/) · **Extension name:** `"wtf"` (older code uses `"wtf_transcription"`)

Use WTF when your adapter calls a speech-to-text provider — Whisper, Deepgram, AssemblyAI, ElevenLabs, AWS, Azure, Google. The point is that downstream tooling shouldn't have to special-case each provider's output.

**Transcripts go in `analysis[]`, not `attachments[]`.** They're derived data, not supplied data.

### Recipe

```python
import json
from foo_adapter.vcon_builder import new_vcon

v = new_vcon(subject="Sales call", extensions=["wtf"])
v.add_party(tel="+15555550100", role="caller")
v.add_party(tel="+15555550200", role="agent")
v.add_dialog(
    type="recording",
    start="2026-05-19T14:32:00Z",
    parties=[0, 1],
    url="https://recordings.example/abc.wav",
    content_hash="sha512-...",
    mediatype="audio/wav",
)

wtf_document = {
    "transcript": {"text": "Hello, this is Foo Corp..."},
    "segments": [
        {"start": 0.0, "end": 2.3, "speaker": 0, "text": "Hello, this is Foo Corp."},
        {"start": 2.3, "end": 5.1, "speaker": 1, "text": "Hi, I'm calling about my account."},
    ],
    "language": "en-US",
}

v.add_analysis(
    type="transcript",
    dialog=0,
    vendor="openai-whisper",
    product="whisper-large-v3",
    body=json.dumps(wtf_document),
    encoding="json",
    schema="https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/",
)
```

### Resulting `analysis[]` entry

```json
{
  "type": "transcript",
  "dialog": 0,
  "vendor": "openai-whisper",
  "product": "whisper-large-v3",
  "encoding": "json",
  "schema": "https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/",
  "body": "{\"transcript\":{\"text\":\"Hello, this is Foo Corp...\"}, ...}"
}
```

### Common mistakes

* ❌ Putting the transcript in `attachments[]` instead of `analysis[]`
* ❌ Storing the WTF document as a Python dict in `body` — `body` is always a string, paired with `encoding="json"`
* ❌ Adding a separate plain-text transcript analysis — the WTF document already carries `transcript.text`
* ❌ Forgetting `vendor` (the library will raise `TypeError`)
* ❌ Writing `schema_version` instead of `schema`

***

## Lawful Basis (recording consent)

⚖️ **Spec:** [Lawful Basis Extension](../extensions/lawful-basis.md) · [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/) · **Extension name:** `"lawful_basis"`

Use this when your adapter handles conversations covered by GDPR, CCPA, HIPAA, TCPA, state-by-state recording consent laws, or when you're generating synthetic data for which you want auditable origin tracking.

**Critical exception:** the lawful\_basis attachment is the one place where attachments use `type` instead of `purpose`. This is documented in the spec.

### Recipe — consent-based (GDPR Article 6(1)(a))

The library's `add_lawful_basis_attachment` helper requires model objects for `purpose_grants` and `proof_mechanisms`. For most adapters it's easier to build the attachment dict directly and append it:

```python
import json
from foo_adapter.vcon_builder import new_vcon

v = new_vcon(extensions=["lawful_basis"])
v.add_party(tel="+15555550100", role="caller", validation="self-reported")

lawful_basis_attachment = {
    "type": "lawful_basis",  # NOTE: type, not purpose — extension-defined exception
    "encoding": "json",
    "party": 0,
    "dialog": 0,
    "body": json.dumps({
        "lawful_basis": "consent",
        "regulation": "GDPR",
        "expiration": "2027-05-19T00:00:00Z",
        "purpose_grants": [
            {"purpose": "call_recording", "granted_at": "2026-05-19T14:32:00Z"},
            {"purpose": "transcription", "granted_at": "2026-05-19T14:32:00Z"},
            {"purpose": "analysis", "granted_at": "2026-05-19T14:32:00Z"},
        ],
        "proof_mechanisms": [
            {
                "type": "audio_prompt",
                "description": "Caller responded 'yes' to IVR prompt",
                "captured_at": "2026-05-19T14:32:00Z",
            }
        ],
    }),
}
v.vcon_dict["attachments"].append(lawful_basis_attachment)
```

### Recipe — synthetic data

When generating synthetic conversations (test fixtures, training data, demos), document the synthetic origin rather than forging a real consent record:

```python
lawful_basis_attachment = {
    "type": "lawful_basis",
    "encoding": "json",
    "party": 0,
    "dialog": 0,
    "body": json.dumps({
        "lawful_basis": "legitimate_interests",
        "expiration": None,
        "purpose_grants": [
            {"purpose": "recording"},
            {"purpose": "transcription"},
            {"purpose": "analysis"},
            {"purpose": "redistribution"},
        ],
        "proof_mechanisms": [
            {
                "type": "external_system",
                "description": "Synthetic data generated by vcon-faker v2.3 on 2026-05-19",
            }
        ],
    }),
}
v.vcon_dict["attachments"].append(lawful_basis_attachment)
```

Also mark each synthetic party with `validation: "synthetic"`. See the [synthetic data section of the compliance checklist](spec-compliance-checklist.md#synthetic-test-data).

### Common mistakes

* ❌ Using `purpose: "lawful_basis"` instead of `type: "lawful_basis"` — this attachment is the _one_ core spec exception
* ❌ Inventing an ad-hoc `synthetic_data_consent` attachment shape instead of using lawful\_basis with `legitimate_interests` + `external_system` proof
* ❌ Omitting `"lawful_basis"` from top-level `extensions[]`
* ❌ Treating `expiration` as optional for consent-based grounds — GDPR consent without an expiry is brittle

***

## SIP Signaling

📞 **Spec:** [SIP Signaling Extension](../extensions/sip-signaling.md) · [`draft-howe-vcon-sip-signaling`](https://datatracker.ietf.org/doc/draft-howe-vcon-sip-signaling/) · **Extension name:** `"sip-signaling"`

Use this when your adapter handles telephony — SignalWire, Twilio, Sippy, SIPREC streams, FreeSWITCH. The extension records the SIP-level facts (Call-ID, From/To URIs, P-Asserted-Identity, SDP fingerprints, signaling timestamps) that don't fit cleanly into a generic vCon party/dialog shape.

### Recipe

```python
import json

v = new_vcon(extensions=["sip-signaling"])
v.add_party(tel="+15555550100", sip="sip:caller@example.com", role="caller")
v.add_party(tel="+15555550200", sip="sip:agent@example.com", role="agent")
v.add_dialog(
    type="recording",
    start="2026-05-19T14:32:00Z",
    parties=[0, 1],
    duration=125.3,
    url="https://recordings.example/abc.wav",
    content_hash="sha512-...",
    mediatype="audio/wav",
)

v.add_attachment(
    purpose="sip_signaling",
    body=json.dumps({
        "call_id": "abc123@signalwire.com",
        "from": "sip:caller@example.com",
        "to": "sip:agent@example.com",
        "p_asserted_identity": "+15555550100",
        "invite_at": "2026-05-19T14:31:55Z",
        "answer_at": "2026-05-19T14:32:00Z",
        "bye_at": "2026-05-19T14:34:05Z",
        "sdp_fingerprint": "sha-256 AB:CD:...",
    }),
    encoding="json",
    party=0,
    dialog=0,
)
```

Note this uses `purpose=`, not `type=` — SIP signaling follows the standard core attachment shape (only `lawful_basis` is the exception).

***

## Agent Session

🤖 **Spec:** [Agent Session Extension](../extensions/agent-session.md) · [`draft-howe-vcon-agent-session`](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/) · **Extension name:** `"agent_session"`

Use this when your adapter handles AI-agent conversations — Claude AI exports, LLM tool-use traces, voice-agent sessions. The extension carries the session-level facts the core spec doesn't model: model identity, tool invocations, system-prompt fingerprints, token counts.

### Recipe

```python
v = new_vcon(extensions=["agent_session"])
v.add_party(name="User", role="user")
v.add_party(name="Claude Sonnet 4.6", role="agent", validation="ai_agent")

v.add_dialog(
    type="text",
    start="2026-05-19T14:32:00Z",
    parties=[0],
    originator=0,
    body="Help me debug this Python function",
    mimetype="text/plain",
)
v.add_dialog(
    type="text",
    start="2026-05-19T14:32:02Z",
    parties=[1],
    originator=1,
    body="Sure — paste the function and a sample of the failing input.",
    mimetype="text/plain",
)

v.add_attachment(
    purpose="agent_session",
    body=json.dumps({
        "model": "claude-sonnet-4-6",
        "model_id": "claude-sonnet-4-6-20260319",
        "system_prompt_hash": "sha256:7b8a2f0e...",
        "tools_available": ["Read", "Edit", "Bash"],
        "tool_calls": [],
        "total_input_tokens": 1247,
        "total_output_tokens": 89,
        "session_id": "session_abc123",
    }),
    encoding="json",
    party=0,
    dialog=0,
)
```

The [`vcon-anthropic-chats`](../tools/vcon-anthropic-chats.md) adapter is a reference implementation if you're building an LLM-export adapter.

***

## Combining extensions

Most production adapters combine three or more. A contact-center adapter typically ships SIP signaling + WTF + lawful basis on every vCon. A voice-AI adapter combines agent session + WTF + lawful basis. Just list every extension you emit in the top-level `extensions[]`:

```python
v = new_vcon(extensions=["sip-signaling", "wtf", "lawful_basis"])
```

There's no ordering constraint and no limit. List them all, then add the corresponding attachments and analyses below.

## Where to read the actual specs

When you need authoritative answers, read the drafts — not this cookbook:

* [WTF Transcription](../extensions/wtf-transcription.md) → [`draft-howe-vcon-wtf-extension`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf/)
* [Lawful Basis](../extensions/lawful-basis.md) → [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/)
* [SIP Signaling](../extensions/sip-signaling.md) → [`draft-howe-vcon-sip-signaling`](https://datatracker.ietf.org/doc/draft-howe-vcon-sip-signaling/)
* [Agent Session](../extensions/agent-session.md) → [`draft-howe-vcon-agent-session`](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/)
* [Lifecycle (SCITT)](../extensions/lifecycle.md) → [`draft-howe-vcon-lifecycle`](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/)
