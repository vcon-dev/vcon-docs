---
description: >-
  The TypeScript / JavaScript implementation of vCon, parallel to the Python
  library.
icon: plug
---

# vCon-JS Library

`vcon-js` is the TypeScript / JavaScript implementation of the vCon specification. It is a peer to the [Python `vcon` library](../vcon-library/), targeting the same [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) spec.

> **Current version:** `vcon-js` **0.4.0** · Install with `npm install vcon-js` · [GitHub: vcon-dev/vcon-js](https://github.com/vcon-dev/vcon-js) · Targets [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/).

## When to use vcon-js vs the Python library

* **vcon-js** if you're writing Node services, Edge functions, Cloudflare Workers, browser code, or any TypeScript codebase that needs to read or write vCons.
* **Python `vcon`** if you're writing data pipelines, ML preprocessing, or anything inside the conserver runtime.

The two libraries produce byte-compatible vCons. You can build with one and consume with the other.

## Parity with the Python library

What you get in 0.4.0:

* ✅ Full core spec coverage: `Vcon`, `Party`, `Dialog`, `Attachment`, `Analysis`, `PartyHistory`
* ✅ Spec-correct field names: `amended` (not `appended`), `purpose` on attachments, `vcon: "0.4.0"` syntax param set automatically
* ✅ External and inline media (`body` + `encoding` or `url` + `content_hash`)
* ✅ Content hash validation (enforces `sha512-<base64url>` format)
* ✅ Auto-serialization: `addAnalysis({body: someObject, encoding: "json"})` JSON-stringifies the body for you
* ✅ Generic extension declaration (`addExtension`, `addCriticalExtension`)
* ✅ Tags via `addTag()`, with read-through `tags` property
* ✅ Per-class validators: `Dialog.validate()`, `Attachment.validate()`, `Party.validate()`, `PartyHistory.validate()`

What's not in 0.4.0 yet (vs. Python `vcon` 0.9.4):

* ❌ Per-extension helpers. The Python library has `add_lawful_basis_attachment()`, `add_wtf_transcription_attachment()`, and `add_wtf_transcription_analysis()`; vcon-js exposes the generic `addAttachment` / `addAnalysis` and you provide the extension shape yourself. The [Extensions section](../extensions/) shows what each one requires.
* ❌ Built-in signing/encryption convenience. The Python lib wraps JWS/JWE via `sign()`/`verify()` (RS256); in vcon-js the `signatures` and `payload` shapes are typed in `VconData` but you handle key management and signing through `jsonwebtoken` or a similar peer dependency.
* ❌ Extension-specific search helpers (`findLawfulBasisAttachments`, `findWtfAttachments`). Filter the arrays manually for now.

## Worked examples

The library ships three runnable TypeScript tutorials under [`examples/`](https://github.com/vcon-dev/vcon-js/tree/main/examples):

* `01-text-chat.ts` — multi-turn text chat with mixed-identifier parties, tags, serialization (`npm run example:chat`)
* `02-call-recording.ts` — phone recording with external media, content hash, STIR validation, sentiment/transcription analysis, contact_center extension, party history (`npm run example:call`)
* `03-video-conference.ts` — five-party video call, incomplete dialogs, multiple attachments, meeting series grouping, action-items analysis (`npm run example:conference`)

## Documentation in this section

* [Quickstart](quickstart.md) — create a vCon, add parties and dialog, add an analysis, serialize.
* [API Reference](api-reference.md) — every exported class and method.
* [LLM Guide](llm-guide.md) — paste this into a model's context window when you want it to generate vcon-js code.

## See also

* [Python vCon Library](../vcon-library/) — the peer implementation
* [Extensions](../extensions/) — the shape of each extension, useful when you need to add extension data manually in vcon-js
