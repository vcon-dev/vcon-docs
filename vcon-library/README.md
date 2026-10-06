---
description: Where to start with the Python vCon library, version 0.10.0 on core-04.
icon: plug
---

# vCon Library

## vCon Python Library

> **Current version:** `vcon` **0.10.0** · `pip install vcon` · [GitHub: vcon-dev/vcon-lib](https://github.com/vcon-dev/vcon-lib) · Targets [`draft-ietf-vcon-vcon-core-04`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), syntax `"0.4.0"` · Python 3.12 or later.

### About the library

The `vcon` package builds, reads, validates, signs and serializes vCons in Python. A vCon is a JSON container for one conversation: its parties, dialogs (text, audio, video, images), attachments, and analysis such as transcripts or sentiment. The library covers the core-04 object model and two extensions, [Lawful Basis](../extensions/lawful-basis.md) and [WTF transcription](../extensions/wtf-transcription.md).

### Features

* Create vCons and add parties, dialogs, attachments, analysis and tags
* Content hashes in the spec form (`sha512-<unpadded base64url>`)
* Sign and verify with JWS
* Lawful basis attachments and permission checks
* WTF transcription helpers, provider adapters (Whisper, Deepgram, AssemblyAI) and SRT or WebVTT export
* Validate objects, files and JSON strings
* Generate UUID8 identifiers

### Pages in this section

* [Quickstart](quickstart.md): install, build a vCon, release notes
* [Library API reference](library-api-reference.md): classes, methods, constants
* [Guide for LLMs](vcon-library-guide-for-llms.md): a compact reference for models that write code against the library

Field rules live in the [field reference](../vcons/field-reference.md).

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
