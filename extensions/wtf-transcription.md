---
description: World Transcription Format — a vendor-neutral analysis shape for speech-to-text output.
---

# 🗣️ WTF Transcription Extension

**Draft:** [`draft-howe-vcon-wtf-extension`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf/) · **Extension name:** `"wtf"` (often emitted as `"wtf_transcription"` by older library code)

## What it is

Every speech-to-text provider — Whisper, Deepgram, AssemblyAI, Google, AWS, Azure, ElevenLabs, the next one — has its own JSON output shape. If you want to swap providers, compare them on the same audio, or build downstream tooling that doesn't care who did the transcription, you end up writing adapter code over and over.

The World Transcription Format (WTF) extension defines a single canonical shape for that output. It covers the transcript text, time-aligned segments, optional word-level timing, optional speaker labels, quality metrics, and provider metadata.

WTF data lives in `analysis[]`, not `attachments[]` — it's *derived from* the conversation, not supplied alongside it.

## When to use it

- Recording → transcription pipelines where you may switch providers later.
- Quality benchmarking: same audio, multiple providers, identical downstream code.
- LLM ingestion: a stable transcript shape means prompts and parsers don't change when the ASR vendor does.
- Speaker diarization workflows: WTF carries speaker labels in a standard way.

## Spec surface

The WTF document is added as an `analysis[]` entry. The recommended form per the speckit is to use `type: "transcript"` and identify WTF via the `schema:` URL — this stays compatible with consumers that just want "any transcript". Older library code (and the draft's own examples) use `type: "wtf_transcription"`; both forms are valid as long as `schema:` points at the WTF draft.

```json
{
  "analysis": [
    {
      "type": "transcript",
      "dialog": 0,
      "vendor": "openai-whisper",
      "product": "whisper-large-v3",
      "encoding": "json",
      "schema": "https://datatracker.ietf.org/doc/draft-howe-vcon-wtf/",
      "body": "{\"transcript\":{\"text\":\"Hello, I need help with my account.\",\"language\":\"en\",\"duration\":3.2,\"confidence\":0.95},\"segments\":[{\"id\":0,\"start\":0.0,\"end\":3.2,\"text\":\"Hello, I need help with my account.\",\"confidence\":0.95}],\"metadata\":{\"created_at\":\"2026-05-18T10:00:00Z\",\"provider\":\"whisper\",\"model\":\"whisper-large-v3\"}}"
    }
  ]
}
```

**Required analysis fields:**

- `type` — `"transcript"` (recommended) or `"wtf_transcription"`.
- `dialog` — index of the dialog this transcription covers.
- `vendor` — REQUIRED by the core spec. Identifies the ASR provider (e.g. `"openai-whisper"`, `"deepgram"`, `"assemblyai"`).
- `product` — the specific model (e.g. `"whisper-large-v3"`, `"nova-2"`).
- `encoding` — `"json"`.
- `schema` — URL pointing at the WTF draft, so consumers know how to parse `body`.
- `body` — JSON-encoded WTF document as a **string**. The body is always a string in vCon; pair it with `encoding: "json"` to indicate the string is itself JSON.

## The WTF document shape

Inside `body` (decoded), the WTF document has four top-level sections:

```json
{
  "transcript": {
    "text": "Hello, I need help with my account.",
    "language": "en",
    "duration": 3.2,
    "confidence": 0.95
  },
  "segments": [
    { "id": 0, "start": 0.0, "end": 3.2, "text": "Hello, I need help with my account.",
      "confidence": 0.95, "speaker": 0 }
  ],
  "speakers": [
    { "id": 0, "label": "Customer", "segments": [0], "total_time": 3.2, "confidence": 0.95 }
  ],
  "metadata": {
    "created_at": "2026-05-18T10:00:00Z",
    "provider": "whisper",
    "model": "whisper-large-v3"
  }
}
```

`transcript` and `segments` are required. `speakers` is optional (use it when diarization was performed). `metadata` carries provider/model details and processing context.

Word-level timing is optional and lives inside each segment as a `words[]` array:

```json
{ "id": 0, "start": 0.0, "end": 0.5, "text": "Hello", "confidence": 0.98, "speaker": 0 }
```

Don't forget to declare the extension at the top level:

```json
{
  "vcon": "0.4.0",
  "extensions": ["wtf"]
}
```

## Python helper

The `vcon` Python library has `add_wtf_transcription_attachment()`. Be aware of two quirks:

1. The helper places the transcription as an **attachment**, not an analysis entry. Spec-compliant code puts WTF data in `analysis[]`. You can either rebuild the attachment manually (shown above) or call the helper and then move the entry.
2. The helper emits `type: "wtf_transcription"` — you may want to rename this to `purpose:` (if you keep it as an attachment) or to `type: "transcript"` (if you move it into `analysis[]`).

The direct-construction pattern is usually simpler:

```python
import json
from vcon import Vcon

v = Vcon.build_new()
v.vcon_dict["vcon"] = "0.4.0"
# ... add parties, dialog ...

wtf_doc = {
    "transcript": {"text": "...", "language": "en", "duration": 3.2, "confidence": 0.95},
    "segments": [{"id": 0, "start": 0.0, "end": 3.2, "text": "...", "confidence": 0.95}],
    "metadata": {"provider": "whisper", "model": "whisper-large-v3",
                 "created_at": "2026-05-18T10:00:00Z"},
}

v.vcon_dict["analysis"].append({
    "type": "transcript",
    "dialog": 0,
    "vendor": "openai-whisper",
    "product": "whisper-large-v3",
    "encoding": "json",
    "schema": "https://datatracker.ietf.org/doc/draft-howe-vcon-wtf/",
    "body": json.dumps(wtf_doc),
})
v.add_extension("wtf")
```

## See also

- [Standard Links — Conserver](../conserver/standard-links.md) — the conserver ships Whisper and Deepgram links that emit WTF-shaped analysis.
- [Speech Recognition Test Set](../use-cases-studies/speech-recognition-test-set.md) — multi-provider benchmarking is one of WTF's design goals.
