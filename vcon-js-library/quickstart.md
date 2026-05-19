---
description: Create a vCon in TypeScript or JavaScript — parties, dialog, analysis, serialize.
---

# 🐰 Quickstart (vcon-js)

This is the TypeScript counterpart to the [Python Quickstart](../vcon-library/quickstart.md). Both libraries produce byte-compatible vCons.

## Install

```bash
npm install vcon-js
```

vcon-js targets [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) and sets the `vcon: "0.4.0"` syntax parameter automatically.

## Minimal example

```typescript
import { Vcon, Party, Dialog } from 'vcon-js';

const vcon = Vcon.buildNew();
vcon.subject = 'Customer Support Chat';

vcon.addParty(new Party({ tel: '+15551234567', name: 'Alice', role: 'customer' }));
vcon.addParty(new Party({ mailto: 'bob@example.com', name: 'Bob', role: 'agent' }));

vcon.addDialog(new Dialog({
  type: 'text',
  start: new Date().toISOString(),
  parties: [0, 1],
  originator: 0,
  mediatype: 'text/plain',
  body: 'Hi, I need help with my account.',
  encoding: 'none',
}));

console.log(vcon.toJson());
```

## Recording with external media

For audio/video, prefer the external-media pattern: keep the recording out of the JSON, reference it with `url`, and provide a SHA-512 `content_hash` so consumers can verify integrity.

```typescript
const dialog = new Dialog({
  type: 'recording',
  start: '2026-05-18T14:00:00Z',
  parties: [0, 1],
  mediatype: 'audio/x-wav',
  duration: 137.5,
  url: 'https://media.example.com/recordings/abc123.wav',
  content_hash: 'sha512-iWS5VtJSp7v...',
});
vcon.addDialog(dialog);
```

## Adding an analysis (transcript)

Analysis entries record what was derived from the conversation. Always include `vendor`. For JSON-bodied analyses, set `encoding: 'json'` and pass a string body (use `JSON.stringify`).

```typescript
vcon.addAnalysis({
  type: 'transcript',
  dialog: 0,
  vendor: 'openai-whisper',
  product: 'whisper-large-v3',
  encoding: 'json',
  schema: 'https://datatracker.ietf.org/doc/draft-howe-vcon-wtf/',
  body: JSON.stringify({
    transcript: { text: 'Hello, I need help.', language: 'en', duration: 1.5, confidence: 0.95 },
    segments: [{ id: 0, start: 0.0, end: 1.5, text: 'Hello, I need help.', confidence: 0.95 }],
    metadata: { provider: 'whisper', model: 'whisper-large-v3', created_at: new Date().toISOString() },
  }),
});
```

## Declaring an extension

Extension declarations are generic in vcon-js — there are no per-extension builder helpers in 0.4.0. You add extension data to the right array yourself and declare the extension:

```typescript
vcon.addExtension('lawful_basis');           // declared but optional for consumers
vcon.addCriticalExtension('lawful_basis');   // consumers MUST understand this to process the vCon
```

For the JSON shape of each extension, see the [Extensions section](../extensions/README.md).

### Lawful Basis example

```typescript
vcon.addAttachment({
  type: 'lawful_basis',  // <-- exception: lawful_basis uses `type`, not `purpose`
  encoding: 'json',
  party: 0,
  dialog: 0,
  body: JSON.stringify({
    lawful_basis: 'consent',
    expiration: '2027-05-18T00:00:00Z',
    purpose_grants: [
      { purpose: 'recording', granted: true, granted_at: new Date().toISOString() },
      { purpose: 'transcription', granted: true, granted_at: new Date().toISOString() },
    ],
    proof_mechanisms: [
      { mechanism_type: 'audio_recording', dialog_index: 0,
        description: 'Verbal consent at start of recording' },
    ],
  }),
});
vcon.addCriticalExtension('lawful_basis');
```

## Tags

```typescript
vcon.addTag('region', 'us-east');
vcon.addTag('campaign', 'spring-2026');
```

Tags are surfaced through the conserver and through the [vCon MCP server](../mcp-server/README.md) for fast filtering.

## Loading and saving

```typescript
import { Vcon } from 'vcon-js';

// Serialize
const json = vcon.toJson();
const obj  = vcon.toDict();

// Deserialize
const restored = Vcon.buildFromJson(json);
```

## Common mistakes (caught in code review)

- Using `appended` instead of `amended` — the library writes `amended`; if you carry data forward from older code, rename.
- Using `must_support` — use `addCriticalExtension()` (writes to `critical[]` / `must_understand[]`).
- Putting transcripts in `attachments[]` — transcripts belong in `analysis[]` (see [WTF Transcription](../extensions/wtf-transcription.md)).
- Passing a JS object directly as `body` for JSON content — `body` must be a string. Use `JSON.stringify`.
- Forgetting `vendor` on analysis entries — `vendor` is REQUIRED.

## See also

- [API Reference](api-reference.md) — full method list
- [LLM Guide](llm-guide.md) — drop into an LLM's context window for assisted development
- [Python Quickstart](../vcon-library/quickstart.md) — the Python equivalent
