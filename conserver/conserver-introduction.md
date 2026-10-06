---
description: Explains what a conserver is for and why conversations need their own processing server, with the public sources behind each claim.
---

# 🚀 Conserver

A conserver is the server that receives vCons after a conversation ends, processes them, and delivers the results to the systems that use them. An adapter on a phone system, contact center queue or chat platform turns each conversation into a vCon and posts it to the conserver. The conserver runs it through a configured chain of links (transcription, redaction, summarization, tagging, routing, ledger registration) and writes the finished record to databases, object stores, search indexes or the [vCon MCP server](../mcp-server/README.md). The reference implementation is open source under the MIT license at [vcon-dev/vcon-server](https://github.com/vcon-dev/vcon-server).

<figure><img src="../.gitbook/assets/Conserver Pictures (7).jpg" alt="Conversation sources feed the conserver, which extracts, transforms and serves vCons to business systems"><figcaption><p>Sources on the left, business systems on the right, the conserver extracting, transforming and serving vCons in between</p></figcaption></figure>

## Why conversations need their own server

A recorded conversation is a large media file plus who spoke, when, under what consent, and what every later process derived from it. A conserver keeps all of that in one vCon and adds to it at each step, so the transcript, the summary and the consent record stay attached to the call they describe. The vCon carries its own [lawful basis](../extensions/lawful-basis.md) and its [lifecycle](../extensions/lifecycle.md) events, which is what lets a deletion request or a consent withdrawal be answered from the record itself.

Industry commentary makes the same point. CRM analyst Thomas Wieberneit, in [The vCon Reality Check](http://blog.aheadcrm.co.nz/2026/04/the-vcon-reality-check-moving-beyond.html), argues for putting conversational data infrastructure ahead of AI marketing, and notes that a vCon makes it possible to pinpoint the container holding an interaction, "providing an auditable trail for consent and compliance." Steve Lasker's TADSummit 2024 keynote ([session page](https://blog.tadsummit.com/2024/10/29/the-rise-and-rise-of-vcon/)) makes the case that combining SCITT and vCon delivers AI governance for conversations at scale. The conserver's `scitt` link and storage are where that happens.

## Public vCon deployments

The [vCon Progress Report (TADSummit, August 2025)](https://blog.tadsummit.com/2025/08/20/vcon-progress-report/) describes a company serving car dealerships and a contact center company working together to embed vCons in the contact center, and a pilot with 211 social services contact centers across the U.S. For more public material see [Talks, Articles and Press](../talks-articles-press/README.md).

## Related pages

- [Concepts](concepts.md) defines links, chains, storages and tracers, and how a chain runs.
- [Quick Start](conserver-quick-start.md) gets one running.
- [Operational Benefits](operational-benefits-of-conservers.md) covers what running one buys you.
