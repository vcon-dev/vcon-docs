---
description: >-
  What vcon-js 0.5.2 covers, how it differs from the Python library, and where
  to start.
icon: plug
---

# vCon-JS Library

`vcon-js` is the TypeScript and JavaScript implementation of the vCon specification. It is a peer to the [Python `vcon` library](../vcon-library/) and targets [`draft-ietf-vcon-vcon-core`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) with syntax parameter `"0.4.0"`.

> **Current version:** `vcon-js` **0.5.2** (npm). Install with `npm install vcon-js`. Node 18 or later. Source and changelog: [vcon-dev/vcon-js](https://github.com/vcon-dev/vcon-js).

## When to use vcon-js

* **vcon-js** for Node services, edge functions, Cloudflare Workers, browser code, or any TypeScript codebase that reads or writes vCons.
* **Python `vcon`** for data pipelines, ML preprocessing, and conserver links.

Both libraries write the same JSON structure and each reads the other's output. They do not produce identical bytes. Python 0.10 writes the tags attachment body as a list and drops empty `meta`; vcon-js writes tags as a JSON object. Treat the JSON as interchangeable and do not compare serialized strings across libraries. Field rules live in the [field reference](../vcons/field-reference.md).

## What 0.5.2 covers

* Classes: `Vcon`, `Party`, `Dialog`, `Attachment`, `PartyHistory`. `Analysis` is a type, passed as a plain object to `addAnalysis`.
* Current core-draft surface: `recording-set` dialogs (`recordings`, `recording_set`), analysis `attachment` reference with `dialog` optional, party `type`, `org`, `dept`, and a typed `provenance` parameter on dialog and analysis.
* Spec-correct names: `amended`, `critical`, `purpose` on attachments, `mediatype`. `vcon: "0.4.0"` is set for you.
* Inline and external content: `body` plus `encoding`, or `url` plus `content_hash` (format `sha512-<base64url>`, checked by `validate()`).
* `addAnalysis` accepts an object or array body, serializes it to a string, and sets `encoding: "json"`.
* Tags through `addTag`, `getTag`, and the `tags` getter, stored as one `purpose: "tags"` attachment with `start` set.
* Declared extensions through `addExtension`, `addCriticalExtension`, `hasExtension`, `isCriticalExtension`.
* Finders: `findPartyIndex`, `findDialog`, `findAttachmentByPurpose`, `findAnalysisByType`.
* Per-class validators: `Dialog.validate()`, `Attachment.validate()`, `Party.validate()`, `PartyHistory.validate()`.
* Schema conformance: the test suite validates emitted vCons against the core JSON Schema.

## What it does not do

* **No per-extension helpers.** Python has `add_lawful_basis_attachment()`, `add_wtf_transcription_attachment()`, and `add_wtf_transcription_analysis()`. In vcon-js you build the attachment or analysis yourself with `addAttachment` or `addAnalysis`. See [Lawful Basis](../extensions/lawful-basis.md) and [WTF Transcription](../extensions/wtf-transcription.md) for the shapes. Extension parameters round-trip untyped, except `provenance`.
* **No signing or encryption.** Version 0.5.2 removed the unused `jose` and `jsonwebtoken` dependencies and the `Signature` type and the `signature`, `signatures`, and `payload` fields on `VconData`. Key management and JWS handling are the caller's job. The per-dialog `alg` and `signature` fields for url-referenced content are unchanged.
* **No whole-vCon validator and no content-hash helper.** Compute hashes yourself (the [quickstart](quickstart.md) shows `node:crypto`).
* **No extension-specific search helpers.** Filter `attachments` and `analysis` with `findAttachmentByPurpose` or `Array.filter`.

## Examples

The repository ships runnable TypeScript examples under [`examples/`](https://github.com/vcon-dev/vcon-js/tree/main/examples): a text chat, a call recording with external media, a video conference, and an inline recording. Run them with the `npm run example:*` scripts in the repo.

## In this section

* [Quickstart](quickstart.md) builds a vCon with parties, dialog, analysis, a lawful basis attachment, and tags.
* [API Reference](api-reference.md) lists every exported class, method, and type.
* [LLM Guide](llm-guide.md) is a rule sheet to paste into a model's context when it generates vcon-js code.

## See also

* [Python vCon Library](../vcon-library/)
* [vCon-C Library](../vcon-c-library/)
* [Extensions](../extensions/)
