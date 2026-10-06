---
description: >-
  Worked examples for the extensions adapters use most: WTF transcription,
  lawful basis, SIP signaling, agent session.
---

# 📚 Extensions Cookbook

Most of what adapters care about beyond the core spec (transcripts, recording consent, SIP signaling, AI-agent sessions) lives in extensions. Each recipe below shows the library or template call and the resulting JSON. The draft is the authority for every field; the field rules and spec target are on the [Spec Compliance Checklist](spec-compliance-checklist.md), and the field definitions are in the [vCon field reference](../vcons/field-reference.md).

The recipes use the template's `new_vcon()` and `vcon` library 0.10.0 or later. `add_party()` and `add_dialog()` take `Party` and `Dialog` objects (`from vcon.party import Party`, `from vcon.dialog import Dialog`); `add_attachment()` and `add_analysis()` take keyword arguments. JSON bodies are written as values, not `json.dumps()` strings.

***

## WTF Transcription

**Spec:** [WTF Transcription Extension](../extensions/wtf-transcription.md) and [`draft-howe-vcon-wtf-extension`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/). **Extension token and analysis type:** `wtf_transcription`.

Use it when your adapter calls a speech-to-text provider. Transcripts are derived data, so they go in `analysis[]`, never `attachments[]`.

```python
from vcon.dialog import Dialog
from vcon.party import Party

from foo_adapter.vcon_builder import new_vcon

v = new_vcon(subject="Support call", extensions=["wtf_transcription"])
v.add_party(Party(tel="+15555550100", role="caller"))
v.add_party(Party(tel="+15555550200", role="agent"))
v.add_dialog(Dialog(
    type="recording",
    start="2026-05-19T14:32:00Z",
    parties=[0, 1],
    url="https://recordings.example.com/abc.wav",
    content_hash="sha512-...",
    mediatype="audio/wav",
))

v.add_analysis(
    type="wtf_transcription",
    dialog=0,
    vendor="openai-whisper",
    product="whisper-large-v3",
    body={
        "transcript": {"text": "Hello, this is Example Corp."},
        "segments": [
            {"start": 0.0, "end": 2.3, "speaker": 0, "text": "Hello, this is Example Corp."},
        ],
        "language": "en-US",
    },
    encoding="json",
    schema="https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/",
)
```

The `analysis[]` entry carries `type`, `dialog`, `vendor`, `product`, `encoding: "json"`, `schema` and the document in `body`. Check the segment and word shapes against the draft before relying on them. Do not add a second plain-text transcript analysis; the WTF document already carries `transcript.text`.

***

## Lawful Basis

**Spec:** [Lawful Basis Extension](../extensions/lawful-basis.md) and [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/). **Extension token:** `lawful_basis`.

Use it for any conversation covered by GDPR, CCPA, HIPAA, TCPA or a recording-consent law. Build the attachment with the template's helper, not by hand, and never default the basis in code. What the basis is, and what evidence supports it, is a decision for the operator of the deployment. The adapter only records it.

```python
from datetime import datetime, timezone

from foo_adapter.vcon_builder import LawfulBasisConfig, add_lawful_basis, new_vcon

# Reads LAWFUL_BASIS, LAWFUL_BASIS_PURPOSE, LAWFUL_BASIS_JURISDICTION,
# LAWFUL_BASIS_EXPIRATION, LAWFUL_BASIS_PROOF_MECHANISM from the environment.
# `load_config()` already does this merge with the YAML block as Config.lawful_basis.
cfg = LawfulBasisConfig.from_env()

v = new_vcon()
v.add_party(Party(tel="+15555550100", role="caller"))
add_lawful_basis(v, cfg, granted_at=datetime.now(timezone.utc).isoformat(), party=0, dialog=0)
```

If `LAWFUL_BASIS` is unset, `add_lawful_basis()` returns `False`, logs one warning per process and adds nothing. An invalid value raises `ValueError` when the config loads. Do not use `Vcon.add_lawful_basis_attachment()` from the library; the template's notes record that its output lacks `start` and `mediatype`.

With `LAWFUL_BASIS=consent`, `LAWFUL_BASIS_PURPOSE=recording,transcription` and an expiration set, the helper writes this attachment and adds `lawful_basis` to `extensions[]`:

```json
{
  "purpose": "lawful_basis",
  "start": "2026-05-19T14:32:00+00:00",
  "party": 0,
  "dialog": 0,
  "encoding": "json",
  "mediatype": "application/json",
  "body": {
    "lawful_basis": "consent",
    "expiration": "2027-05-19T00:00:00Z",
    "purpose_grants": [
      {"purpose": "recording", "granted": true, "granted_at": "2026-05-19T14:32:00+00:00"},
      {"purpose": "transcription", "granted": true, "granted_at": "2026-05-19T14:32:00+00:00"}
    ]
  }
}
```

The draft also defines `proof_mechanisms[]`, each with `proof_type`, `timestamp` and `proof_data`, with `proof_type` values `verbal_confirmation`, `signed_document`, `cryptographic_signature` and `external_system`. The draft requires `expiration` in the body, as a timestamp or `null`. The template currently omits it when unset and writes proof entries with `mechanism_type` and `description` instead; compare the helper's output with the draft before you depend on either.

For synthetic data (fixtures, training sets, demos), mark each party `validation: "synthetic"`. If you choose to attach a basis, record the synthetic origin through an `external_system` proof, and never present it as a real consent. See the [checklist](spec-compliance-checklist.md#synthetic-test-data).

***

## SIP Signaling

**Spec:** [SIP Signaling Extension](../extensions/sip-signaling.md) and [`draft-howe-vcon-sip-signaling`](https://datatracker.ietf.org/doc/draft-howe-vcon-sip-signaling/). **Extension token:** `sip-signaling`.

Use it for telephony adapters (SIPREC, FreeSWITCH, Twilio and similar). The draft puts SIP identifiers on parties and dialogs, and puts message data in attachments with registered `purpose` values.

Party fields: `sip_contact`, `sip_user_agent`, `sip_display_name`. Dialog fields: `sip_call_id`, `sip_from_tag`, `sip_to_tag`, `sip_cseq`. Attachment purposes: `sip-invite`, `sip-response`, `sip-ack`, `sip-bye`, `sip-cancel`, `sip-update`, `sip-refer` (media type `message/sip`), `sip-message-trace`, `sip-headers` (JSON), `sip-sdp` (`application/sdp`), `stir-certificate`, `stir-verification-report` and `stir-passport-extended`.

```python
v = new_vcon(extensions=["sip-signaling"])
v.add_party(Party(tel="+15555550100", sip="sip:alice@example.com", role="caller",
                 sip_user_agent="ExamplePhone/2.1"))
v.add_party(Party(tel="+15555550200", sip="sip:bob@example.com", role="agent"))

v.add_dialog(Dialog(
    type="recording",
    start="2026-05-19T14:32:00Z",
    parties=[0, 1],
    url="https://recordings.example.com/abc.wav",
    content_hash="sha512-...",
    mediatype="audio/wav",
    sip_call_id="a84b4c76e66710@pc33.example.com",
    sip_from_tag="1928301774",
    sip_to_tag="a6c85cf",
))

v.add_attachment(
    purpose="sip-message-trace",
    start="2026-05-19T14:31:55Z",
    party=0,
    dialog=0,
    mediatype="application/json",
    encoding="json",
    body={
        "version": "1.0",
        "call_id": "a84b4c76e66710@pc33.example.com",
        "messages": [
            {"timestamp": "2026-05-19T14:31:55.001+00:00", "direction": "sent",
             "party": 0, "method": "INVITE"},
            {"timestamp": "2026-05-19T14:31:55.050+00:00", "direction": "received",
             "party": 1, "status_code": 180, "status_text": "Ringing"},
        ],
    },
)
```

`Party` and `Dialog` pass the `sip_*` keywords through to the output (checked against `vcon` 0.10.0). The trace `call_id` must match the dialog's `sip_call_id`. Where both exist, include the RFC 7989 `session_id` as well as `sip_call_id`.

***

## Agent Session

**Spec:** [Agent Session Extension](../extensions/agent-session.md) and [`draft-howe-vcon-agent-session`](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/). **Extension token:** `agent_session`.

Use it for AI-agent conversations. The draft puts the agent's identity on its party in `meta.agent_session`, the user and assistant turns in ordinary `dialog[]` entries, and the internal trace (tool calls, results, reasoning) in one `agent_trace` analysis.

```python
v = new_vcon(extensions=["agent_session"])
v.add_party(Party(name="User", role="user"))
v.add_party(Party(
    name="Example Agent",
    role="agent",
    validation="system",
    meta={"agent_session": {
        "model_id": "example-model-1",
        "provider": "example-provider",
        "recording_agent": "example-harness/1.0",
    }},
))

v.add_dialog(Dialog(type="text", start="2026-05-19T14:32:00Z", parties=[0], originator=0,
                    mediatype="text/plain", encoding="none",
                    body="Help me debug this function"))
v.add_dialog(Dialog(type="text", start="2026-05-19T14:32:02Z", parties=[1], originator=1,
                    mediatype="text/plain", encoding="none",
                    body="Paste the function and a failing input."))

v.add_analysis(
    type="agent_trace",
    dialog=[0, 1],
    vendor="example-provider",
    product="example-model-1",
    schema="https://datatracker.ietf.org/doc/draft-birkholz-verifiable-agent-conversations/",
    encoding="json",
    body=vac_record,  # the verifiable-agent-record, as a JSON value
)
```

`model_id` and `provider` are required on the party; `recording_agent` is recommended; `environment` is optional. The analysis `vendor` mirrors `provider`, and `product` mirrors `model_id`. The draft's own example shows the trace `body` as a JSON string; under core-04 write the value. File changes go in attachments with `purpose` `agent_file_change`, `agent_artifact` or `agent_environment`. The draft asks that an agent session have a lawful basis whose `purpose_grants` cover `agent_session_recording` and `agent_session_analysis`, and `agent_session_redistribution` where it applies; pass those through `LAWFUL_BASIS_PURPOSE`.

For a working converter, see [`vcon-vac-adapter`](https://github.com/vcon-dev/vcon-vac-adapter).

***

## Combining extensions

List every extension you emit in top-level `extensions[]`. There is no ordering rule.

```python
v = new_vcon(extensions=["sip-signaling", "wtf_transcription"])
add_lawful_basis(v, cfg, granted_at=...)  # adds "lawful_basis" for you
```

## The drafts

When you need an authoritative answer, read the draft, not this page:

* [WTF Transcription](../extensions/wtf-transcription.md): [`draft-howe-vcon-wtf-extension`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/)
* [Lawful Basis](../extensions/lawful-basis.md): [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/)
* [SIP Signaling](../extensions/sip-signaling.md): [`draft-howe-vcon-sip-signaling`](https://datatracker.ietf.org/doc/draft-howe-vcon-sip-signaling/)
* [Agent Session](../extensions/agent-session.md): [`draft-howe-vcon-agent-session`](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/)
* [Lifecycle](../extensions/lifecycle.md): [`draft-howe-vcon-lifecycle`](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/)
