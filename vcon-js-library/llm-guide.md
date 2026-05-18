---
description: Drop this in an LLM's context window when you want it to generate vcon-js code.
icon: scroll
---

# vCon-JS Library Guide for LLMs

This guide gives a Large Language Model everything it needs to generate spec-compliant code against `vcon-js` 0.3.0.

> **Spec target:** [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) · syntax parameter `"vcon": "0.4.0"` · library version `0.3.0`.

## Non-negotiable rules

If you violate any of these, the output is not a valid vCon. Apply them every time:

1. **Syntax param.** Every vCon must have `vcon: "0.4.0"`. `Vcon.buildNew()` sets this automatically — do not override.
2. **Field names.**
   - Top-level: `amended` (NOT `appended`).
   - Critical extensions: `addCriticalExtension(name)` writes to `critical[]` / `must_understand[]`. Do NOT emit `must_support`.
3. **Attachments use `purpose`** (REQUIRED). The single exception is the Lawful Basis extension, which uses `type: "lawful_basis"`.
4. **Analysis requires `vendor`.** Always set `vendor`. Use `schema` (URL or identifier) to declare the body format. Never write `schema_version`.
5. **Bodies are strings.** When `encoding: 'json'`, the `body` value is a `JSON.stringify(...)` string, not a JS object.
6. **External media needs both `url` and `content_hash`.** The hash is `sha512-<base64url>`.
7. **Timestamps are ISO 8601 with timezone.** Use `new Date().toISOString()` or an explicit offset; never bare local time.
8. **Don't emit empty `group: []` or `redacted: {}`.** `group` is reserved; omit it unless populated.

## Public API at a glance

```typescript
import {
  Vcon, Party, Dialog, Attachment, PartyHistory,
  VCON_VERSION,
} from 'vcon-js';
```

**`Vcon`** — `buildNew()`, `buildFromJson(json)`, `addParty()`, `addDialog()`, `addAttachment()`, `addAnalysis()`, `addTag(key, value)`, `addExtension(name)`, `addCriticalExtension(name)`, `toJson()`, `toDict()`.

**`Party`** — identifiers: `tel`, `sip`, `mailto`, `stir`, `did`; descriptive: `name`, `role`, `validation`, `civicaddress`, `timezone`, `meta`.

**`Dialog`** — `type: 'recording' | 'text' | 'transfer' | 'incomplete'`; required `start`, `parties`; inline `body` + `encoding` OR external `url` + `content_hash`.

**`Attachment`** — required `purpose`, `party`, `dialog`. Lawful Basis exception uses `type`.

**`Analysis`** is a type, not a class — pass a plain object to `addAnalysis()`. Required: `type` and `vendor`.

## Canonical end-to-end example

```typescript
import { Vcon, Party, Dialog } from 'vcon-js';

const vcon = Vcon.buildNew();
vcon.subject = 'Refund discussion';

const customerIdx = vcon.addParty(new Party({
  tel: '+15551234567', name: 'Alice', role: 'customer'
}));
const agentIdx = vcon.addParty(new Party({
  mailto: 'bob@example.com', name: 'Bob', role: 'agent'
}));

const callStart = '2026-05-18T14:00:00Z';

vcon.addDialog(new Dialog({
  type: 'recording',
  start: callStart,
  parties: [customerIdx, agentIdx],
  originator: customerIdx,
  mediatype: 'audio/x-wav',
  duration: 137.5,
  url: 'https://media.example.com/recordings/abc123.wav',
  content_hash: 'sha512-iWS5VtJSp7v...',
}));

// Transcript as analysis (NOT attachment)
vcon.addAnalysis({
  type: 'transcript',
  dialog: 0,
  vendor: 'openai-whisper',
  product: 'whisper-large-v3',
  encoding: 'json',
  schema: 'https://datatracker.ietf.org/doc/draft-howe-vcon-wtf/',
  body: JSON.stringify({
    transcript: { text: '...', language: 'en', duration: 137.5, confidence: 0.93 },
    segments: [],
    metadata: { provider: 'whisper', model: 'whisper-large-v3', created_at: new Date().toISOString() },
  }),
});

// Lawful basis (note the `type` field — extension exception)
vcon.addAttachment({
  type: 'lawful_basis',
  encoding: 'json',
  party: 0,
  dialog: 0,
  body: JSON.stringify({
    lawful_basis: 'consent',
    expiration: '2027-05-18T00:00:00Z',
    purpose_grants: [
      { purpose: 'recording', granted: true, granted_at: callStart },
    ],
    proof_mechanisms: [
      { mechanism_type: 'audio_recording', dialog_index: 0,
        description: 'Verbal consent at start of recording' },
    ],
  }),
});
vcon.addCriticalExtension('lawful_basis');

// Optional: tags for downstream search
vcon.addTag('region', 'us-east');

console.log(vcon.toJson());
```

## Common bugs in generated code (avoid)

- ❌ `vcon.addAttachment({ type: 'transcript', ... })` — wrong; transcripts live in `analysis[]`. Use `addAnalysis`.
- ❌ `vcon.addAttachment({ purpose: 'lawful_basis', ... })` — wrong; lawful_basis uses `type`.
- ❌ `body: { transcript: ... }` paired with `encoding: 'json'` — body must be a string. Use `JSON.stringify`.
- ❌ Emitting `appended: { uuid: '...' }` or `must_support: [...]` — use `amended` and `critical[]` (via `addCriticalExtension`).
- ❌ `start: '2026-05-18 14:00:00'` (no timezone) — use `.toISOString()`.
- ❌ External media without `content_hash` — both `url` and `content_hash` are required for external media.

## Where to find extension shapes

The vcon-js library does not include per-extension helpers in 0.3.0. When asked to add extension data, refer to the corresponding page in the [Extensions section](../extensions/README.md) for the exact JSON shape, then construct an attachment or analysis entry matching that shape.

## See also

- [Quickstart](quickstart.md)
- [API Reference](api-reference.md)
- [Python Library Guide for LLMs](../vcon-library/vcon-library-guide-for-llms.md) — same material, Python edition
