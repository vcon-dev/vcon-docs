---
description: >-
  A rule sheet to paste into an LLM's context window so it generates correct
  vcon-js 0.5.2 code.
---

# 📜 LLM Guide

Paste this page into a model's context when it writes code against `vcon-js` 0.5.2.

> **Spec target:** [`draft-ietf-vcon-vcon-core`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), syntax `"vcon": "0.4.0"`. Library version `0.5.2`. Field rules: [field reference](../vcons/field-reference.md).

## Rules

Output that breaks these is not a valid vCon.

1. **Syntax parameter.** `Vcon.buildNew()` sets `vcon: "0.4.0"`. Do not override it.
2. **Field names.** `amended`, never `appended`. `critical`, never `must_support` or `must_understand`. `mediatype`, never `mimetype`. Use `addCriticalExtension(name)` to write `critical[]`.
3. **Attachments use `purpose`.** There is no `type` field on attachments, including the lawful basis attachment: `purpose: 'lawful_basis'`. Always pass `start`; the core schema requires it and `addAttachment` does not fill it in.
4. **Analysis requires `vendor`.** Use `schema` to name the body format. Never write `schema_version`.
5. **Structured bodies.** Pass objects and arrays as the value. `addAnalysis` serializes an object or array body to a string and sets `encoding: 'json'` for you. `addAttachment` stores the body as given, so pass `encoding: 'json'` with an object body.
6. **External media needs `url` and `content_hash`.** The hash is `sha512-<base64url>` of the file bytes. Compute it with `createHash('sha512').update(bytes).digest('base64url')` from `node:crypto`. Never write a placeholder hash into output.
7. **Timestamps are RFC 3339 with timezone.** Use `new Date().toISOString()` or an explicit offset.
8. **Do not emit empty `group: []` or `redacted: {}`.**
9. **Transcripts are analysis**, not attachments. Use `type: 'wtf_transcription'` for the WTF extension.
10. **There is no signing in the library.** Do not call `vcon.sign()` or read `vcon.signatures`. Version 0.5.2 removed `Signature`, `signature`, `signatures`, and `payload` from the types.

## API at a glance

```typescript
import { Vcon, Party, Dialog, Attachment, PartyHistory, VCON_VERSION } from 'vcon-js';
```

**`Vcon`**: `buildNew()`, `buildFromJson(json)`, `addParty(party)` (returns `void`), `addDialog(dialog)`, `addAttachment(params)`, `addAnalysis(params)`, `addTag(key, value)`, `getTag(key)`, `tags`, `addExtension(name)`, `addCriticalExtension(name)`, `hasExtension(name)`, `isCriticalExtension(name)`, `findPartyIndex(by, val)`, `findDialog(by, val)`, `findAttachmentByPurpose(purpose)`, `findAnalysisByType(type)`, `toJson()`, `toDict()`.

**`Party`**: identifiers `tel`, `sip`, `mailto`, `stir`, `did`. Descriptive `name`, `role`, `type`, `org`, `dept`, `validation`, `civicaddress`, `timezone`, `meta`.

**`Dialog`**: `type` is `'recording' | 'text' | 'transfer' | 'incomplete' | 'recording-set'`. Required `type` and `start`. Inline: `addInlineData(body, mediatype, { encoding, filename })`. External: `addExternalData(url, mediatype, { filename, content_hash })`. Checks: `isText`, `isRecording`, `isAudio`, `isVideo`, `isEmail`, `isInlineData`, `isExternalData`, `validate()`.

**`Attachment`**: `purpose` required, `party` and `dialog` default to `0`. `Attachment.VALID_ENCODINGS` is `['base64url', 'json', 'none']`.

**`Analysis`** is a type. Pass a plain object to `addAnalysis()` with `type` and `vendor`. `dialog` is optional because an analysis may key off `attachment`.

**Party indices.** `addParty` returns nothing. Use `vcon.parties.length - 1` or `vcon.findPartyIndex('name', 'Alice')`.

## Canonical example

```typescript
import { createHash } from 'node:crypto';
import { Vcon, Party, Dialog } from 'vcon-js';

const vcon = Vcon.buildNew();
vcon.subject = 'Refund discussion';

vcon.addParty(new Party({ tel: '+15551234567', name: 'Alice', role: 'customer' }));
const customer = vcon.parties.length - 1;
vcon.addParty(new Party({ mailto: 'bob@example.com', name: 'Bob', role: 'agent' }));
const agent = vcon.parties.length - 1;

const callStart = '2026-05-18T14:00:00Z';
const audio = Buffer.from('placeholder audio bytes'); // real recording bytes in practice
const hash = 'sha512-' + createHash('sha512').update(audio).digest('base64url');

const recording = new Dialog({
  type: 'recording',
  start: callStart,
  parties: [customer, agent],
  originator: customer,
  duration: 137.5,
});
recording.addExternalData('https://media.example.com/recordings/abc123.wav', 'audio/x-wav', {
  content_hash: hash,
});
vcon.addDialog(recording);

vcon.addAnalysis({
  type: 'wtf_transcription',
  dialog: 0,
  vendor: 'openai-whisper',
  product: 'whisper-large-v3',
  schema: 'https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/',
  body: {
    transcript: { text: '...', language: 'en', duration: 137.5, confidence: 0.93 },
    segments: [],
    metadata: { provider: 'whisper', model: 'whisper-large-v3', created_at: callStart },
  },
});

// Illustrative values only. Never invent a real consent record.
vcon.addAttachment({
  purpose: 'lawful_basis',
  start: callStart,
  party: customer,
  dialog: 0,
  encoding: 'json',
  body: {
    lawful_basis: 'consent',
    expiration: '2027-05-18T00:00:00Z',
    purpose_grants: [
      { purpose: 'recording', granted: true, granted_at: callStart },
    ],
    proof_mechanisms: [
      { proof_type: 'verbal_confirmation', timestamp: callStart,
        proof_data: { dialog_reference: 0, time_offset: '00:00:05' } },
    ],
  },
});
vcon.addExtension('lawful_basis');

vcon.addTag('region', 'us-east');

console.log(vcon.toJson());
```

## Common bugs in generated code

* `const idx = vcon.addParty(...)`. `addParty` returns `void`. Use `vcon.parties.length - 1`.
* `vcon.addAttachment({ type: 'lawful_basis', ... })`. Attachments have no `type`. Use `purpose: 'lawful_basis'`.
* `vcon.addAttachment({ type: 'transcript', ... })`. Transcripts live in `analysis[]`. Use `addAnalysis`.
* `appended: {...}` or `must_support: [...]`. Use `amended` and `critical[]`.
* `start: '2026-05-18 14:00:00'` with no timezone. Use `.toISOString()`.
* External media without `content_hash`, or with a made-up hash.
* Importing `Signature` or calling a signing method. Removed in 0.5.2.
* Mutating an `Attachment` after `vcon.addAttachment(att)` and expecting the vCon to change. The vCon holds a copy.

## Extension shapes

vcon-js has no per-extension helpers. Build the attachment or analysis yourself from the page for that extension: [Lawful Basis](../extensions/lawful-basis.md), [WTF Transcription](../extensions/wtf-transcription.md), and the rest of the [Extensions section](../extensions/). Only `provenance` on dialog and analysis is typed.

## See also

* [Quickstart](quickstart.md)
* [API Reference](api-reference.md)
* [Python Library Guide for LLMs](../vcon-library/vcon-library-guide-for-llms.md)
