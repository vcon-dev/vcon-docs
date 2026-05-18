---
description: >-
  The TypeScript / JavaScript implementation of vCon, parallel to the Python
  library.
icon: plug
---

# vCon-JS Library

`vcon-js` is the TypeScript / JavaScript implementation of the vCon specification. It is a peer to the [Python `vcon` library](../vcon-library/), targeting the same [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) spec.

> **Current version:** `vcon-js` **0.3.0** · Install with `npm install vcon-js` · [GitHub: vcon-dev/vcon-js](https://github.com/vcon-dev/vcon-js)

## When to use vcon-js vs the Python library

* **vcon-js** if you're writing Node services, Edge functions, Cloudflare Workers, browser code, or any TypeScript codebase that needs to read or write vCons.
* **Python `vcon`** if you're writing data pipelines, ML preprocessing, or anything inside the conserver runtime.

The two libraries produce byte-compatible vCons. You can build with one and consume with the other.

## Parity with the Python library

What you get in 0.3.0:

* ✅ Full core spec coverage: `Vcon`, `Party`, `Dialog`, `Attachment`, `Analysis`, `PartyHistory`
* ✅ Spec-correct field names: `amended` (not `appended`), `purpose` on attachments, `vcon: "0.4.0"` syntax param set automatically
* ✅ External and inline media (`body` + `encoding` or `url` + `content_hash`)
* ✅ JWS-shaped serialization helpers
* ✅ Generic extension declaration (`addExtension`, `addCriticalExtension`)
* ✅ Tags via `addTag()`

What's not in 0.3.0 yet:

* ❌ Per-extension helpers. The Python library has `add_lawful_basis_attachment()` and `add_wtf_transcription_attachment()`; vcon-js does not. You add extension data by appending to the relevant array yourself. The [Extensions section](../extensions/) shows what shape each extension requires.
* ❌ Built-in signing/encryption convenience. The Python lib wraps JWS/JWE; in vcon-js you handle key management and signing via `jsonwebtoken` (a peer dependency).

## Documentation in this section

* [Quickstart](quickstart.md) — create a vCon, add parties and dialog, add an analysis, serialize.
* [API Reference](api-reference.md) — every exported class and method.
* [LLM Guide](llm-guide.md) — paste this into a model's context window when you want it to generate vcon-js code.

## See also

* [Python vCon Library](../vcon-library/) — the peer implementation
* [Extensions](../extensions/) — the shape of each extension, useful when you need to add extension data manually in vcon-js
