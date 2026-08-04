---
description: Public API of vcon-js 0.5.0 — every exported class, method, and constant.
---

# 🔌 API Reference

Complete API reference for **`vcon-js` 0.5.0**, targeting [`draft-ietf-vcon-vcon-core`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) (syntax `"0.4.0"`).

See the [Quickstart](quickstart.md) for a code-walkthrough; this page is a reference.

## Top-level exports

```typescript
import {
  Vcon, Party, Dialog, Attachment, PartyHistory,
  // Types:
  VconData, Analysis, Encoding, CivicAddress, Redacted, Amended,
  DialogType, DialogTypeEnum, DialogDisposition, SessionId, ContentHash,
  AttachmentType, PartyType, PartyHistoryType,
  // Constants:
  VCON_VERSION,  // '0.4.0' (syntax parameter; unchanged as the draft advanced)
} from 'vcon-js';
```

## `Vcon`

Main container class.

### Static methods

| Method                             | Returns | Description                                                           |
| ---------------------------------- | ------- | --------------------------------------------------------------------- |
| `Vcon.buildNew()`                  | `Vcon`  | Create an empty vCon. Sets `uuid`, `created_at`, and `vcon: "0.4.0"`. |
| `Vcon.buildFromJson(json: string)` | `Vcon`  | Parse a vCon from JSON.                                               |

### Instance properties

`uuid`, `vcon`, `created_at`, `updated_at`, `subject`, `parties`, `dialog`, `attachments`, `analysis`, `tags`, `extensions`, `critical`, `redacted`, `amended`, `meta`.

Note: `amended` is the spec-correct name. `redacted` and `amended` are mutually exclusive — assigning one while the other is set throws. `group` is reserved by the spec and is not exposed.

### Adding content

| Method                               | Description                                                                    |
| ------------------------------------ | ------------------------------------------------------------------------------ |
| `addParty(party: Party)`             | Append a party. The returned index is `parties.length - 1`.                    |
| `addDialog(dialog: Dialog)`          | Append a dialog entry.                                                         |
| `addAttachment(attachment)`          | Append an attachment. See the [Extensions section](../extensions/) for shapes. |
| `addAnalysis(analysis)`              | Append an analysis entry. `vendor` is REQUIRED.                                |
| `addTag(key: string, value: string)` | Add a tag (surfaces in conserver / MCP search).                                |

### Extensions

| Method                                       | Description                                                                                                         |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `addExtension(name: string)`                 | Add to `extensions[]`. Consumers MAY understand it.                                                                 |
| `addCriticalExtension(name: string)`         | Add to both `extensions[]` and `critical[]`. Consumers MUST understand it or refuse the vCon. |
| `hasExtension(name: string): boolean`        | Check whether an extension is declared.                                                                             |
| `isCriticalExtension(name: string): boolean` | Check whether an extension is in `critical[]`.                                                                      |

### Search helpers

| Method                                | Returns              | Description                                                     |
| ------------------------------------- | -------------------- | -------------------------------------------------------------- |
| `findPartyIndex(by, val)`             | `number \| undefined` | Index of the first party whose `by` field equals `val`.        |
| `findDialog(by, val)`                 | `Dialog \| undefined` | First dialog whose `by` field equals `val`.                    |
| `findAttachmentByPurpose(purpose)`    | `Attachment \| undefined` | First attachment with a given purpose.                    |
| `findAnalysisByType(type)`            | `Analysis \| undefined` | First analysis of a given type.                             |

### Serialization

| Method     | Returns  | Description                                    |
| ---------- | -------- | ---------------------------------------------- |
| `toJson()` | `string` | JSON string ready to sign, store, or transmit. |
| `toDict()` | `object` | Plain JS object.                               |

## `Party`

```typescript
new Party({
  tel?: string,             // E.164 telephone number
  sip?: string,             // SIP URI
  mailto?: string,          // Email address
  stir?: string,            // STIR PASSporT (JWS compact)
  did?: string,             // Decentralized Identifier
  name?: string,
  role?: string,            // 'customer', 'agent', 'supervisor', ...
  type?: string,            // participant type (schema-defined token)
  org?: string,             // organization the party belongs to
  dept?: string,            // department the party belongs to
  validation?: string,      // 'verified', 'unverified', 'synthetic', ...
  gmlpos?: string,          // GML position (lat/long)
  civicaddress?: CivicAddress,
  timezone?: string,
  uuid?: string,
  meta?: object,            // Extension-specific data (e.g. agent_session)
});
```

Methods: `toDict()`, `hasIdentifier()`, `getPrimaryIdentifier()`, `validate()`.

## `Dialog`

```typescript
new Dialog({
  type: 'recording' | 'text' | 'transfer' | 'incomplete' | 'recording-set',
  start: string,            // ISO 8601 with timezone
  parties: number[] | number[][],   // party indices, or [[primary], [secondary]] for multi-channel
  originator?: number,
  mediatype?: string,
  duration?: number,
  // Inline content (mutually exclusive with url):
  body?: string,
  encoding?: 'base64url' | 'json' | 'none',
  // External content (mutually exclusive with body):
  url?: string,
  content_hash?: string | string[],   // sha512-<base64url>
  // Optional:
  filename?: string,
  disposition?: string,     // for type: 'incomplete'
  party_history?: PartyHistory[],
  session_id?: SessionId,   // RFC 7989
  // recording-set:
  recordings?: number[],    // on a 'recording-set' dialog: member recording indices
  recording_set?: number,   // on a recording: back-reference to its 'recording-set'
  // Transfer-specific (index or array of indices):
  transferor?: number,
  transferee?: number,
  transfer_target?: number | number[],
  original?: number | number[],
  consultation?: number | number[],
  target_dialog?: number | number[],
  // Extension parameter:
  provenance?: object,      // draft-howe-vcon-provenance, for machine-generated dialog
});
```

Methods: `toDict()`, `addInlineData(body, encoding, mediatype?)`, `addExternalData(url, contentHash, mediatype?)`, `isText()`, `isRecording()`, `isRecordingSet()`, `isTransfer()`, `isIncomplete()`, `validate()`.

## `Attachment`

```typescript
new Attachment({
  purpose: string,          // REQUIRED for core attachments — e.g. 'contract', 'screenshot'
  party: number,            // REQUIRED — party index, or 0 for vCon-level
  dialog: number,           // REQUIRED — dialog index, or 0 for vCon-level
  start?: string,
  mediatype?: string,
  filename?: string,
  // Inline:
  body?: string,
  encoding?: Encoding,
  // External:
  url?: string,
  content_hash?: string | string[],
  // Lawful Basis exception only:
  type?: string,            // Use 'lawful_basis' for the lawful-basis extension
});
```

Methods: `toDict()`, `addInlineData()`, `addExternalData()`, `isExternalData()`, `isInlineData()`, `validate()`.

> The core spec uses `purpose`, not `type`, on attachments. The Lawful Basis extension is the documented exception — see [Lawful Basis](../extensions/lawful-basis.md).

## `Analysis` (type)

```typescript
interface Analysis {
  type: string;             // 'transcript', 'summary', 'translation', 'sentiment', 'tts', ...
  dialog?: number | number[];    // optional: analysis may key off `attachment` instead
  attachment?: number | number[]; // attachment index/indices the analysis is based on
  vendor: string;           // REQUIRED
  product?: string;
  schema?: string;          // URL or identifier of the body format
  mediatype?: string;
  filename?: string;
  body?: string;            // JSON content goes here as a string (use JSON.stringify)
  encoding?: Encoding;      // 'json', 'base64url', 'none'
  url?: string;
  content_hash?: string | string[];
  provenance?: object;      // draft-howe-vcon-provenance: model/provider/params for the analysis
}
```

## `PartyHistory`

```typescript
new PartyHistory(
  party: number,
  event: 'join' | 'drop' | 'hold' | 'unhold' | 'mute' | 'unmute' | 'keydown' | 'keyup',
  time: Date | string,
  button?: string,          // for keydown/keyup
);
```

Methods: `toDict()`, `PartyHistory.fromDict(obj)`, `validate()`.

## Types

* **`SessionId`** — `{ local: string; remote: string }` per RFC 7989.
* **`CivicAddress`** — GEOPRIV-style address (country, a1–a6, sts, hno, lmk, pc, …).
* **`Redacted`** / **`Amended`** — top-level reference objects to a related vCon UUID (mutually exclusive).
* **`Encoding`** — `'base64url' | 'json' | 'none'`.

## See also

* [Quickstart](quickstart.md) — walking through the API in context
* [LLM Guide](llm-guide.md) — same material formatted for model context
* [Extensions section](../extensions/) — the JSON shape you'll be passing into `addAttachment` / `addAnalysis` for each extension
