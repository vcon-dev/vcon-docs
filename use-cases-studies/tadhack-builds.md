---
description: Lists what developers built on vCon at the 2025 and 2026 TADHack hackathons, which vCon features each build used, and the public data they built on.
---

# TADHack Builds

TADHack is the developer hackathon run alongside TADSummit. Two vCon-focused editions, in 2025 and March 2026, produced public, recorded builds. They are prototypes, not production deployments, and are listed here because each one is on the public record with a demo.

## TADHack 2025 and its dataset

Hackers were given [vcon-dev/tadhack-2025](https://github.com/vcon-dev/tadhack-2025), 43 synthetic customer-service calls at a fictional yacht brokerage, generated with [vcon\_faker](../tools/vcon-faker.md). Each call carries audio by URL with a `content_hash`, transcript, diarization and summary as `analysis` entries, and a [`lawful_basis`](../extensions/lawful-basis.md) attachment whose proof records that it is synthetic demo data with no real data subject. Every party is marked `"validation": "synthetic"`.

Builds described on the TADSummit blog ([meet the developers, 2025-06-20](https://blog.tadsummit.com/?p=6374)):

* **vCase**, the overall winner per the [vCon Progress Report](https://blog.tadsummit.com/2025/08/20/vcon-progress-report/): a case platform for support organizations in Nigeria that combines voice transcripts, SMS and email into one thread per case.
* **Supernova**: a dashboard that turns vCon files into insights, with an AI assistant for natural-language questions.
* **Reflecta**: an undergraduate project from Valencia College.
* A build using vCons to compare customer feedback across departments.

## VCONIC TADHack 2026

Held 2026-03-07 to 2026-03-08 with sixteen submissions ([list of hacks](https://blog.tadhack.com/2026/03/08/vconic-tadhack-the-hacks/), [event page](https://blog.tadhack.com/2025/12/19/vconic-tadhack/)). Most connected to the [vCon MCP server](../mcp-server/what-is-the-vcon-mcp-server.md). The full review, with winners, is [VCONIC TADHack 2026: Hackathon Review](../helps-and-hacks/vconic-tadhack-2026-hackathon-review.md). A selection, by the vCon feature each one exercised:

| Build | What it did | vCon feature |
| --- | --- | --- |
| Apparitions (senior class winner) | Location-based audio experiences; each point of interest is a vCon with coordinates and media in attachments | [`attachments`](../vcons/field-reference.md#attachment-object) as typed data |
| ConvoSense | Pulls conversations from a voice-AI platform and a support-chat platform and converts them to vCon | One format across vendors |
| Patanisha | Phone, SMS and email for each support case unified into one timeline; loaded TADHack 2025 data; used a conserver for storage and processing | Multi-channel [`dialog`](../vcons/field-reference.md#dialog-object), [conserver](../conserver/README.md) |
| vChat | Messaging app that imports WhatsApp, SMS and email threads into vCons, with export and optional redaction | Import and export, redaction |
| ConsentMate | Consent dashboard tracking active, expired and expiring consent per call | [Lawful Basis](../extensions/lawful-basis.md) |
| 911 First Response | Emergency call turned into a vCon whose analysis carries transcript, summary and a structured dispatch action | [`analysis`](../vcons/field-reference.md#analysis-object) |
| ConvoLens | Financial-services conversation intelligence across messaging, social and call channels, queried through MCP | `analysis`, MCP |
