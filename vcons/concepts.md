---
description: >-
  Defines the terms you meet in the vCon spec and on this site in a sentence
  or two each, with links to the exact field definitions.
---

# 💡 Concepts

Short definitions, aligned with [`draft-ietf-vcon-vcon-core-04`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/). For every field name, type and requirement, see the [field reference](field-reference.md).

## vCon

A JSON container for one conversation: who took part, what was said, analysis of it, and related files. Conversations can be phone calls, video meetings, text messages, chat or email threads. The name echoes [vCard](https://datatracker.ietf.org/doc/html/rfc6350), which did the same for contact details. A vCon carries `uuid`, `created_at` and, under core-04, `"vcon": "0.4.0"`, plus four arrays: `parties`, `dialog`, `analysis` and `attachments`.

## Party

An entry in `parties` for each participant or observer, human or bot. Every party field is optional, and a party can be empty when nothing is known about it. Identifiers include `tel`, `sip`, `mailto`, `did` and a STIR PASSporT in `stir`. Descriptive fields include `name`, `type` (`person`, `bot` or `organization`), `org`, `dept`, location in `gmlpos` or `civicaddress`, and `validation`, which names how identity was checked without storing the data used to check it. Other entries refer to a party by its index in the array.

## Dialog

An entry in `dialog` for captured conversation. Core-04 defines five types: `recording` (audio or video), `text` (a message, chat line or email), `recording-set` (several recordings that make up one call), `transfer` (who transferred whom, to whom) and `incomplete` (a call that failed before anyone spoke, with a `disposition` such as `no-answer` or `busy`). A transcript is not dialog; it is analysis.

## Analysis

An entry in `analysis` for anything derived from the conversation: transcripts, summaries, sentiment, translations. Each entry names its `type` and the `vendor` that produced it, and can name a `product` and a `schema` for its format. The list doubles as a record of which systems have processed the conversation.

## Attachment

An entry in `attachments` for material supplied alongside the conversation rather than derived from it: a document shared during a call, lead data, a signaling trace, or a lawful basis record. Each attachment has a `purpose`, a `start` time, the index of the `party` that contributed it, and a related `dialog` index.

## Inline and external content

Dialog, analysis and attachment entries hold content one of two ways. Inline content sits in `body` with an `encoding`: `none` for plain text, `base64url` for binary, `json` for a JSON value. External content is referenced by an HTTPS `url` with a `content_hash` (`sha512-` followed by the Base64url digest), so a reader can tell whether the file changed. A `mediatype` gives the content's media type.

## Extension

A separately specified addition to the core fields, named in the top-level `extensions` array. An extension a reader must understand to interpret the vCon correctly is also listed in `critical`; a reader that does not support it must reject the vCon. See [Extensions](../extensions/README.md).

## Signed and encrypted forms

A vCon exists in three forms. The unsigned form is the plain JSON object. The signed form wraps it in a JWS signature, which detects any change. The encrypted form wraps the signed form in JWE, encrypting the whole object. Signing is not automatic: core-04 says a vCon SHOULD be signed or encrypted when it leaves the security domain that built it.

## Redacted and amended versions

A signed vCon cannot change without breaking its signature, so changes produce a new vCon. A redacted vCon removes data, such as personal information, and points back to the less redacted version through `redacted`. An amended vCon adds data, such as new analysis, and points back to the prior version through `amended`. The pointer carries the prior version's `uuid` and can add a `url` with `content_hash`.

## Data projection

A flat view of selected vCon fields, such as one spreadsheet or database row per conversation. Projections are a convenience for storage and reporting. They usually drop data, so they do not replace the vCon.

## Privacy and consent

The lawful basis for processing a conversation, including any consent given, can travel inside the vCon as a [Lawful Basis](../extensions/lawful-basis.md) attachment, so systems that receive the vCon can check it before acting. The [Lifecycle](../extensions/lifecycle.md) draft records what then happens to the vCon on a SCITT transparency service. For the privacy vocabulary itself, see the [Privacy Primer](privacy-primer.md).

## Conserver

The open source server that collects conversations, builds vCons and runs them through processing chains. See the [Conserver](../conserver/README.md) section.
