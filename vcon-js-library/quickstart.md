---
description: Build a vCon in TypeScript with parties, dialog, analysis, a lawful basis attachment, and tags, then serialize it.
---

# 🐰 Quickstart (vcon-js)

This is the TypeScript counterpart to the [Python Quickstart](../vcon-library/quickstart.md). Both libraries write the same JSON structure. Field rules are in the [field reference](../vcons/field-reference.md).

## Install

```bash
npm install vcon-js
```

vcon-js 0.5.2 targets [`draft-ietf-vcon-vcon-core`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) and sets `vcon: "0.4.0"` for you. It needs Node 18 or later.

## Parties and a text dialog

`addParty` returns nothing. A party's index is its position in `parties`.

```typescript
import { Vcon, Party, Dialog } from 'vcon-js';

const vcon = Vcon.buildNew();
vcon.subject = 'Customer Support Chat';

vcon.addParty(new Party({ tel: '+15551234567', name: 'Alice', role: 'customer' }));
vcon.addParty(new Party({ mailto: 'bob@example.com', name: 'Bob', role: 'agent' }));
const customer = vcon.findPartyIndex('name', 'Alice'); // 0
const agent = vcon.parties.length - 1;                 // 1

const chat = new Dialog({
  type: 'text',
  start: new Date().toISOString(),
  parties: [0, 1],
  originator: 0,
});
chat.addInlineData('Hi, I need help with my account.', 'text/plain', { encoding: 'none' });
vcon.addDialog(chat);
```

`addInlineData(body, mediatype, { encoding, filename })` sets the body and clears any `url`. `encoding` defaults to `'none'`. Valid encodings are `'base64url'`, `'json'`, and `'none'` (`Attachment.VALID_ENCODINGS`).

## Recording with external media

Keep audio and video out of the JSON. Reference it with `url` and a `content_hash` of the form `sha512-<base64url>`. `addExternalData(url, mediatype, { filename, content_hash })` sets all three and clears any inline body.

```typescript
import { createHash } from 'node:crypto';

const audio = Buffer.from('placeholder audio bytes'); // your recording bytes
const hash = 'sha512-' + createHash('sha512').update(audio).digest('base64url');

const recording = new Dialog({
  type: 'recording',
  start: '2026-05-18T14:00:00Z',
  parties: [customer!, agent],
  originator: customer,
  duration: 137.5,
});
recording.addExternalData('https://media.example.com/recordings/abc123.wav', 'audio/x-wav', {
  filename: 'abc123.wav',
  content_hash: hash,
});

const check = recording.validate();
if (!check.valid) throw new Error(check.errors.join('; '));
vcon.addDialog(recording);
```

`Dialog` also has `isText()`, `isRecording()`, `isAudio()`, `isVideo()`, `isEmail()`, `isInlineData()`, and `isExternalData()`.

## Analysis

Pass an object or array as `body`. `addAnalysis` serializes it to a string and sets `encoding: 'json'`. Always set `vendor`. This example is a WTF transcription; see [WTF Transcription](../extensions/wtf-transcription.md) and [draft-howe-vcon-wtf-extension](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/).

```typescript
vcon.addAnalysis({
  type: 'wtf_transcription',
  dialog: 0,
  vendor: 'openai-whisper',
  product: 'whisper-large-v3',
  schema: 'https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/',
  body: {
    transcript: { text: 'Hello, I need help.', language: 'en', duration: 1.5, confidence: 0.95 },
    segments: [{ id: 0, start: 0.0, end: 1.5, text: 'Hello, I need help.', confidence: 0.95 }],
    metadata: { provider: 'whisper', model: 'whisper-large-v3', created_at: '2026-05-18T14:00:00Z' },
  },
});
```

## Lawful basis

The lawful basis extension is an attachment with `purpose: 'lawful_basis'` and a JSON body, defined in [draft-howe-vcon-lawful-basis](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/). The body carries `lawful_basis`, `expiration` (an RFC 3339 time or `null`), and `purpose_grants`, each with `purpose`, `granted`, and `granted_at`. `proof_mechanisms` is optional, each entry with `proof_type`, `timestamp`, and `proof_data`. The values below are an illustration, not a real consent record. Declare the extension with `addExtension`.

```typescript
vcon.addAttachment({
  purpose: 'lawful_basis',
  start: '2026-05-18T14:00:00Z',
  party: 0,
  dialog: 0,
  encoding: 'json',
  body: {
    lawful_basis: 'consent',
    expiration: '2027-05-18T00:00:00Z',
    purpose_grants: [
      { purpose: 'recording', granted: true, granted_at: '2026-05-18T14:00:00Z' },
      { purpose: 'transcription', granted: true, granted_at: '2026-05-18T14:00:00Z' },
    ],
    proof_mechanisms: [
      { proof_type: 'verbal_confirmation', timestamp: '2026-05-18T14:00:05Z',
        proof_data: { dialog_reference: 0, time_offset: '00:00:05' } },
    ],
  },
});
vcon.addExtension('lawful_basis');          // consumers MAY understand it
// vcon.addCriticalExtension('lawful_basis'); // consumers MUST understand it or refuse the vCon
```

Attachment bodies are stored as you pass them, so an object stays an object. Always pass `start`: the core schema requires it and the library does not fill it in for you (only `addTag` does). `party` and `dialog` default to `0`.

## Tags

```typescript
vcon.addTag('region', 'us-east');
vcon.getTag('region'); // 'us-east'
vcon.tags;             // { region: 'us-east' }
```

Tags are stored as one `purpose: 'tags'` attachment. The conserver and the [vCon MCP server](../mcp-server/README.md) read them for filtering.

## Serialize and load

```typescript
const json = vcon.toJson();
const restored = Vcon.buildFromJson(json);
```

## Common mistakes

* Using `appended` instead of `amended`, or `must_support` instead of `critical`. Use `addCriticalExtension()` for `critical`.
* Putting transcripts in `attachments[]`. Transcripts belong in `analysis[]`.
* Writing `type: 'lawful_basis'` on an attachment. Attachments use `purpose`, not `type`.
* Omitting `vendor` on an analysis.
* Omitting `start` on an attachment you add with `addAttachment`.
* Reading the return value of `addParty` as an index. It returns `void`.

## See also

* [API Reference](api-reference.md)
* [LLM Guide](llm-guide.md)
* [Python Quickstart](../vcon-library/quickstart.md)
