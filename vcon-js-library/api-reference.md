---
description: Every exported class, method, type, and constant in vcon-js 0.5.2.
---

# 🔌 API Reference

Reference for **`vcon-js` 0.5.2**, targeting [`draft-ietf-vcon-vcon-core`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) (syntax `"0.4.0"`). For a walkthrough see the [Quickstart](quickstart.md). For field rules see the [field reference](../vcons/field-reference.md).

## Top-level exports

```typescript
import {
  // Classes
  Vcon, Party, Dialog, Attachment, PartyHistory,
  // Types
  VconData, Analysis, Encoding, CivicAddress, Redacted, Amended,
  DialogType, DialogTypeEnum, DialogDisposition, SessionId, ContentHash,
  AttachmentType, PartyType, PartyHistoryType,
  // Constant
  VCON_VERSION,  // '0.4.0'
} from 'vcon-js';
```

The type `DialogType` is the dialog object interface, `AttachmentType` and `PartyType` are the attachment and party object interfaces, and `DialogTypeEnum` is the dialog type string union. 0.5.2 has no `Signature` type and no signing surface.

## `Vcon`

### Static methods

| Method | Returns | Description |
| --- | --- | --- |
| `Vcon.buildNew()` | `Vcon` | Empty vCon with `uuid`, `created_at`, and `vcon: "0.4.0"`. |
| `Vcon.buildFromJson(json: string)` | `Vcon` | Parse a vCon. Throws `Failed to parse vCon JSON` on bad input. |

### Properties

Getters: `uuid`, `vcon`, `created_at`, `updated_at`, `parties`, `dialog`, `attachments`, `analysis`, `tags`, `extensions`, `critical`. Getters and setters: `subject`, `redacted`, `amended`, `meta`.

`redacted` and `amended` are mutually exclusive. Setting one while the other is set throws. `tags` returns the decoded tags as an object.

### Adding content

| Method | Returns | Description |
| --- | --- | --- |
| `addParty(party: Party)` | `void` | Append a party. Its index is `parties.length - 1`. |
| `addDialog(dialog: Dialog)` | `void` | Append a dialog. |
| `addAttachment(params)` | `Attachment` | Append an attachment. `purpose` is required. `party` and `dialog` default to `0`. `body` may be any value and is stored as given. Pass `start`. |
| `addAnalysis(params)` | `void` | Append an analysis. An object or array `body` is serialized to a string and `encoding` is set to `'json'`. A string `body` is stored as is. |
| `addTag(key, value)` | `void` | Set a tag in the single `purpose: "tags"` attachment, creating it with `start` set if needed. |
| `getTag(key)` | `string \| undefined` | Read one tag. |

`addAttachment` params: `purpose`, `body?`, `encoding?`, `url?`, `content_hash?`, `mediatype?`, `filename?`, `start?`, `party?`, `dialog?`.

`addAnalysis` params: `type`, `dialog?`, `attachment?`, `vendor?`, `product?`, `schema?`, `body?`, `encoding?`, `url?`, `content_hash?`, `mediatype?`, `filename?`, `provenance?`. The type allows omitting `vendor`, but the spec requires it, so always set it.

### Extensions

| Method | Description |
| --- | --- |
| `addExtension(name)` | Add to `extensions[]` (once). |
| `addCriticalExtension(name)` | Add to `critical[]` and `extensions[]`. Consumers MUST understand it or refuse the vCon. |
| `hasExtension(name): boolean` | Is the name in `extensions[]`. |
| `isCriticalExtension(name): boolean` | Is the name in `critical[]`. |

### Finders

| Method | Returns | Description |
| --- | --- | --- |
| `findPartyIndex(by, val)` | `number \| undefined` | Index of the first party whose `by` field equals `val`. |
| `findDialog(by, val)` | `Dialog \| undefined` | First dialog whose `by` field equals `val`, as a `Dialog` instance. |
| `findAttachmentByPurpose(purpose)` | `Attachment \| undefined` | First attachment with that `purpose` (plain object). |
| `findAnalysisByType(type)` | `Analysis \| undefined` | First analysis of that `type`. |

### Serialization

| Method | Returns | Description |
| --- | --- | --- |
| `toJson()` | `string` | JSON string. |
| `toDict()` | `VconData` | Shallow copy of the data object. |

## `Party`

```typescript
new Party({
  tel?, sip?, mailto?, stir?, did?,   // identifiers (strings)
  name?, role?, type?, org?, dept?,
  validation?, gmlpos?, civicaddress?: CivicAddress, timezone?,
  uuid?, meta?: object,
});
```

Methods: `toDict()`, `hasIdentifier()`, `getPrimaryIdentifier()`, `validate()`. `validate()` returns `{ valid, warnings }`.

## `Dialog`

```typescript
new Dialog({
  type: 'recording' | 'text' | 'transfer' | 'incomplete' | 'recording-set',
  start: string | Date,            // RFC 3339 with timezone
  parties?: number | number[],
  originator?: number,
  mediatype?: string, filename?: string, duration?: number,
  body?: string, encoding?: 'base64url' | 'json' | 'none',   // inline
  url?: string, content_hash?: string | string[],             // external
  disposition?: string,            // required when type is 'incomplete'
  party_history?: PartyHistory[],
  session_id?: SessionId | SessionId[],
  application?: string, message_id?: string,
  recordings?: number[], recording_set?: number,              // recording-set
  transferor?: number, transferee?: number,
  transfer_target?, original?, consultation?, target_dialog?, // number | number[]
  alg?: string, signature?: string,   // url-referenced content signature
  provenance?: object,                // draft-howe-vcon-provenance
});
```

Methods:

| Method | Description |
| --- | --- |
| `addInlineData(body, mediatype, { encoding?, filename? })` | Set inline content. `encoding` defaults to `'none'`. Clears `url` and `content_hash`. |
| `addExternalData(url, mediatype, { filename?, content_hash? })` | Set external content. Clears `body` and `encoding`. |
| `isText()`, `isRecording()`, `isRecordingSet()`, `isTransfer()`, `isIncomplete()` | Type checks. `isText()` is also true for `mediatype: 'text/plain'`. |
| `isAudio()`, `isVideo()`, `isEmail()` | Checks on `mediatype` (audio and video types, `message/rfc822`). |
| `isInlineData()`, `isExternalData()` | True when `body` or `url` is set. |
| `validate()` | Returns `{ valid, errors }`. Checks type, start, disposition, encoding, `content_hash` format, body versus url, transfer fields, and party history. |
| `toDict()` | Plain object. |

Static: `Dialog.DIALOG_TYPES`, `Dialog.DISPOSITIONS`, `Dialog.MIME_TYPES`, `Dialog.VALID_ENCODINGS`.

## `Attachment`

```typescript
new Attachment({
  purpose: string,           // e.g. 'contract', 'tags', 'lawful_basis'
  start?: string | Date,
  party?: number,            // defaults to 0
  dialog?: number | number[], // defaults to 0
  mediatype?: string, filename?: string,
  body?: any, encoding?: Encoding,         // inline
  url?: string, content_hash?: string | string[],  // external
});
```

The constructor throws on an invalid `encoding`, and sets `encoding: 'none'` when a `body` is given without one.

Methods: `addInlineData(body, mediatype, { encoding?, filename? })`, `addExternalData(url, mediatype, { filename?, content_hash? })`, `isInlineData()`, `isExternalData()`, `validate()`, `toDict()`. Static: `Attachment.VALID_ENCODINGS` (`['base64url', 'json', 'none']`).

`validate()` requires `purpose`, rejects both `body` and `url`, and requires a `sha512-<base64url>` `content_hash` when `url` is set.

Note that `Vcon.addAttachment` copies the attachment's `toDict()` into the vCon. Calling `addInlineData` or `addExternalData` on the returned instance afterwards does not change the vCon. Set content on an `Attachment` before adding it, or pass it in `params`.

### Lawful basis attachment

Use `purpose: 'lawful_basis'` per [draft-howe-vcon-lawful-basis](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/). See the [Quickstart](quickstart.md#lawful-basis) for the body shape.

```typescript
vcon.addAttachment({
  purpose: 'lawful_basis',
  start: '2026-05-18T14:00:00Z',
  encoding: 'json',
  body: { lawful_basis: 'consent', expiration: null, purpose_grants: [
    { purpose: 'recording', granted: true, granted_at: '2026-05-18T14:00:00Z' },
  ] },
});
```

## `Analysis` (type)

```typescript
interface Analysis {
  type: string;                    // 'transcript', 'summary', 'wtf_transcription', ...
  dialog?: number | number[];
  attachment?: number | number[];  // analysis may key off an attachment instead
  vendor?: string;                 // the spec requires it
  product?: string;
  schema?: string;                 // URL or identifier of the body format
  mediatype?: string; filename?: string;
  body?: string;                   // addAnalysis accepts an object or array and serializes it
  encoding?: Encoding;
  url?: string; content_hash?: string | string[];
  provenance?: object;             // draft-howe-vcon-provenance
}
```

## `PartyHistory`

```typescript
new PartyHistory(party: number, event: string, time: Date | string, button?: string);
```

Events: `join`, `drop`, `hold`, `unhold`, `mute`, `unmute`, `keydown`, `keyup`. `button` is required for `keydown` and `keyup`. Methods: `toDict()`, `validate()`, `PartyHistory.fromDict(obj)`. Static: `PartyHistory.VALID_EVENTS`.

## Types

* `SessionId`: `{ local: string; remote: string }` per RFC 7989.
* `CivicAddress`: GEOPRIV-style address fields.
* `Redacted`, `Amended`: top-level reference objects to a related vCon.
* `Encoding`: `'base64url' | 'json' | 'none'`.
* `VconData`: the plain object shape returned by `toDict()`.

## See also

* [Quickstart](quickstart.md)
* [LLM Guide](llm-guide.md)
* [Extensions](../extensions/)
