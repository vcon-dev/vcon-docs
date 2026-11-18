---
description: Where to start with the Python vCon library, version 1.0.3 on core-05.
icon: plug
---

# vCon Library

## vCon Python Library

> **Current version:** `vcon` **1.0.3** · `pip install vcon` · [GitHub: vcon-dev/vcon-lib](https://github.com/vcon-dev/vcon-lib) · Targets [`draft-ietf-vcon-vcon-core-05`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), syntax `"0.5.0"` · Reads and migrates `"0.4.0"` · Python 3.12 or later.

### About the library

The `vcon` package builds, reads, validates, signs and serializes vCons in Python. A vCon is a JSON container for one conversation: its parties, sessions, dialogs (text, recordings, images), events such as joins and transfers, attachments, and analysis such as transcripts or sentiment. The library covers the core-05 object model and two extensions, [Lawful Basis](../extensions/lawful-basis.md) and [WTF transcription](../extensions/wtf-transcription.md).

Version 1.0.0 is a breaking release. Code written for 0.10.0 and core-04 needs changes: dialogs belong to sessions, `party_history` and transfer dialogs became events, and attachments need a `dialog` or `session`. Version 1.0.3 fixes the WTF transcription analysis entry, `add_image()` scope, image content hashes, and logging set up on import. The [quickstart](quickstart.md#release-notes) lists the changes.

### Features

* Create vCons and add parties, sessions, dialogs, events, attachments, analysis and tags
* Migrate 0.4.0 vCons to 0.5.0 on load, with a report of every inferred value
* Content hashes in the spec form (`sha512-<unpadded base64url>`)
* Sign and verify with JWS (`alg: "none"` is refused)
* Lawful basis attachments and permission checks
* WTF transcription helpers, provider adapters (Whisper, Deepgram, AssemblyAI) and SRT or WebVTT export
* Validate objects, files and JSON strings against the core-05 rules, with separate SHOULD-level warnings
* Generate UUID8 identifiers

### Pages in this section

* [Quickstart](quickstart.md): install, build a vCon, load a 0.4.0 vCon, release notes
* [Library API reference](library-api-reference.md): classes, methods, constants
* [Guide for LLMs](vcon-library-guide-for-llms.md): a compact reference for models that write code against the library
* [Generated API docs](https://vcon-dev.github.io/vcon-lib/): Sphinx pages built from the source
* [Changelog](https://github.com/vcon-dev/vcon-lib/blob/main/CHANGELOG.md) and [migration guide](https://github.com/vcon-dev/vcon-lib/blob/main/MIGRATION_GUIDE.md) on GitHub

Field rules live in the [field reference](../vcons/field-reference.md). That page describes core-04 (`"0.4.0"`), which the conserver still writes; the library writes core-05 (`"0.5.0"`).

### IETF vCon Working Group

The vCon (Virtual Conversation) format is being developed as an open standard through the Internet Engineering Task Force (IETF). The vCon Working Group is focused on creating a standardized format for representing digital conversations across various platforms and use cases.

#### Participating in the Working Group

1. **Join the Mailing List**: Subscribe to the vCon working group mailing list at [vcon@ietf.org](mailto:vcon@ietf.org)
2. **Review Documents**:
   * Working group documents and drafts can be found at: https://datatracker.ietf.org/wg/vcon/documents/
   * The current Internet-Draft can be found at: https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/
3. **Attend Meetings**:
   * The working group meets virtually during IETF meetings
   * Meeting schedules and connection details are announced on the mailing list
   * Past meeting materials and recordings are available on the IETF datatracker
4. **Contribute**:
   * Submit comments and suggestions on the mailing list
   * Propose changes through GitHub pull requests
   * Participate in working group discussions
   * Help with implementations and interoperability testing

For more information about the IETF standardization process and how to participate, visit: https://www.ietf.org/about/participate/
