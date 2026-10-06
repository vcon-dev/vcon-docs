---
description: >-
  Shows how to store speech-to-text output from any provider in one shape, so
  you can switch or compare providers without rewriting downstream code.
---

# 🗣️ WTF Transcription Extension

**Draft:** [`draft-howe-vcon-wtf-extension-02`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/) · **Extension token:** `wtf_transcription`

## Purpose of the format

Every speech-to-text provider returns its own JSON. The World Transcription Format (WTF) defines one shape for that output: the full transcript, time-aligned segments, optional word timing, speaker diarization, quality metrics and processing metadata, with provider-specific extras kept in a separate `extensions` object. A transcript in WTF form reads the same whether Whisper, Deepgram or another engine produced it.

WTF data is analysis, derived from the dialog. The draft says it MUST be stored in `analysis[]` (Section 6.1). The extension is Compatible and does not need to be listed in `critical` (Section 5.1).

## Analysis entry

This vCon reduces the draft's example in Section 12.1 to two words:

```json
{
  "vcon": "0.4.0",
  "uuid": "01928e10-193e-8231-b9a2-279e0d16bc46",
  "created_at": "2025-01-02T12:00:00Z",
  "extensions": ["wtf_transcription"],
  "parties": [
    { "tel": "+12025550100", "name": "Alice" },
    { "tel": "+12025550199", "name": "Bob" }
  ],
  "dialog": [
    {
      "type": "recording",
      "start": "2025-01-02T12:15:30Z",
      "duration": 65.2,
      "parties": [0, 1],
      "mediatype": "audio/x-wav",
      "url": "https://example.com/recordings/call-recording.wav",
      "content_hash": "sha512-GLy6IPaIUM1GqzZqfIPZlWjaDsNgNvZM0iCONNThnH0a75fhUM6cYzLZ5GynSURREvZwmOh54-2lRRieyj82UQ"
    }
  ],
  "analysis": [
    {
      "type": "wtf_transcription",
      "dialog": 0,
      "vendor": "deepgram",
      "product": "nova-2",
      "mediatype": "application/json",
      "encoding": "json",
      "body": {
        "transcript": {
          "text": "Hello,",
          "language": "en-US",
          "duration": 65.2,
          "confidence": 0.92
        },
        "segments": [
          {
            "id": 0,
            "start": 0.5,
            "end": 1.1,
            "text": "Hello,",
            "confidence": 0.95,
            "speaker": 0,
            "words": [0, 1]
          }
        ],
        "words": [
          { "id": 0, "start": 0.5, "end": 0.8, "text": "Hello", "confidence": 0.98, "speaker": 0, "is_punctuation": false },
          { "id": 1, "start": 0.9, "end": 1.1, "text": ",", "confidence": 0.95, "speaker": 0, "is_punctuation": true }
        ],
        "speakers": {
          "0": { "id": 0, "label": "Alice", "segments": [0], "total_time": 0.6, "confidence": 0.95 }
        },
        "metadata": {
          "created_at": "2025-01-02T12:15:30Z",
          "processed_at": "2025-01-02T12:16:35Z",
          "provider": "deepgram",
          "model": "nova-2",
          "audio": { "duration": 65.2 }
        }
      }
    }
  ]
}
```

The draft's example points the dialog at a file with no `url` or `body`; this reduction adds a `url` and a placeholder `content_hash` so the dialog is complete under core-04. It has no lawful basis attachment because the point is the analysis shape; see [Lawful Basis](lawful-basis.md) for that.

## Analysis fields

From draft Section 6.1 and core-04 Section 4.5:

* `type` MUST be `"wtf_transcription"`.
* `encoding` MUST be `"json"`, and `body` is the WTF object itself, not a string.
* `dialog` SHOULD give the transcribed dialog index.
* `vendor` is required by core-04. Use the provider; put the model in `product`.

## Body structure

**Required** (Section 6.2.1):

* `transcript`: `text`, `language` (BCP 47, such as `en-US`), `duration` in seconds, `confidence` from 0 to 1.
* `segments`: array of `id`, `start`, `end` (seconds, `end` after `start`), `text`, `confidence`, and optionally `speaker` (integer or string) and `words`.
* `metadata`: `created_at`, `processed_at`, `provider` (lowercase), `model`, and optionally `processing_time`, `audio` (`duration`, `sample_rate`, `channels`, `format`, `bitrate`) and `options`.

**Optional** (Section 6.2.2):

* `words`: a top-level array of word objects (`id`, `start`, `end`, `text`, `confidence`, `speaker`, `is_punctuation`). A segment's `words` array holds indexes into this array, not word objects.
* `speakers`: an object keyed by speaker ID, each value with `id`, `label`, `segments`, `total_time`, `confidence`. Not an array.
* `alternatives`, `enrichments`, `quality`, `streaming`, and `extensions` for provider-specific data such as `extensions.whisper` or `extensions.deepgram`.

All confidence values are normalized to the range 0 to 1 (Section 8.1).

## Python

Build the analysis entry with the [`vcon`](https://pypi.org/project/vcon/) library's `add_analysis()`, which takes keyword arguments only:

```python
from vcon import Vcon

v = Vcon.build_new()  # sets "vcon": "0.4.0"
# add parties and the recording dialog first

wtf = {
    "transcript": {"text": "Hello, I need help with my account.", "language": "en-US",
                   "duration": 3.2, "confidence": 0.95},
    "segments": [{"id": 0, "start": 0.0, "end": 3.2,
                  "text": "Hello, I need help with my account.", "confidence": 0.95}],
    "metadata": {"created_at": "2026-10-06T14:00:00Z", "processed_at": "2026-10-06T14:00:04Z",
                 "provider": "whisper", "model": "whisper-large-v3"},
}

v.add_analysis(
    type="wtf_transcription",
    dialog=0,
    vendor="openai",
    product="whisper-large-v3",
    mediatype="application/json",
    encoding="json",
    body=wtf,
)
v.add_extension("wtf_transcription")
```

Avoid the library's WTF helpers in 0.10.0 for now. `add_wtf_transcription_attachment()` stores WTF in `attachments[]`, which the draft does not allow. `add_wtf_transcription_analysis()` writes `type: "transcription"` instead of `"wtf_transcription"` and passes the body through `json.dumps`, producing a string. Both are upstream bugs in vcon 0.10.0.

## See also

* [Field reference](../vcons/field-reference.md) for analysis fields and body encodings.
* [Standard Links](../conserver/standard-links.md) for the conserver's transcription links.
* [Comparing transcription engines](../use-cases-studies/patterns.md#comparing-transcription-engines) for multi-provider comparison.
