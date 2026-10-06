---
description: Helps you decide whether to run a conserver by setting out what it changes in operations, with the code or public source behind each claim.
---

# 💲 Operational Benefits of Conservers

A conserver is infrastructure for conversation records. It takes in vCons, enriches them through a chain of links, signs and registers them, and stores or forwards them. Each benefit below is tied either to what the [vcon-server](https://github.com/vcon-dev/vcon-server) code does or to a public source.

## Public deployments

The [vCon Progress Report (TADSummit, August 2025)](https://blog.tadsummit.com/2025/08/20/vcon-progress-report/) is the public record:

* The Frontline Group and Strolid are working together to embed vCons in the contact center, with Strolid's conserver platform described as the infrastructure to scale that approach.
* A pilot with 211 contact centers across the U.S., the social services help line, aims at "transforming crisis calls into real-time intelligence."
* Frontline is scaling vCon use cases internally, including outcome reporting through SCITT-verified dashboards.

The [VCONIC TADHack](https://blog.tadhack.com/2026/03/08/vconic-tadhack-the-hacks/) in March 2026 produced sixteen hacks.

## AI chosen per step

A chain is a list of independent links, so the transcription engine, the redaction step, the summarizer and the tagger are each a separate configuration entry. Switching transcription vendor is a change to one link's `vendor` option, and a chain can mix hosted and local models. Configuration is re-read by every worker on each loop, so a change takes effect without a redeploy. See [Standard Links](standard-links.md) and [Inside the Conserver](inside-the-conserver.md).

## Provenance on a transparency ledger

The `scitt` link signs a statement about the vCon and registers it on a SCITT transparency service, storing the COSE receipt in the vCon. Run it at creation and again after enrichment and the vCon carries proof of both states. See [Standard Links](standard-links.md) and [Lifecycle](../extensions/lifecycle.md). Steve Lasker's [TADSummit Innovators episode](https://blog.tadsummit.com/2024/08/20/steve-lasker/) and his TADSummit 2024 keynote ([session page](https://blog.tadsummit.com/2024/10/29/the-rise-and-rise-of-vcon/)) make the case that combining SCITT and vCon delivers AI governance for conversations at scale.

Thomas Wieberneit's [The vCon Reality Check](http://blog.aheadcrm.co.nz/2026/04/the-vcon-reality-check-moving-beyond.html) makes the related compliance point: a vCon identifies the container holding an interaction, "providing an auditable trail for consent and compliance."

## Many tenants, many conservers

One conserver can run separate chains, with separate ingress lists, storages and partner keys, for separate customers. Partners submit through `POST /vcon/external-ingress` with a key that only works for their own list. Conservers in different networks or jurisdictions pass work between them with [followers](concepts.md#follower), which move the vCon itself rather than sharing a database.

## Failures you can replay

Failures land in queues you can inspect. A chain that raises puts the UUID on `DLQ:<ingress_list>`; a storage that fails puts it on `DLQ:storage:<name>` and the other storages still get the write. Both can be replayed through the API, and the vCon's Redis copy is kept for seven days by default so there is something to replay. See [Concepts](concepts.md#dead-letter-queues).

## One record, many stores

A chain can write the same vCon to Postgres for reporting, S3 for retention, a search index and the vCon MCP server in one pass. Every store receives the same JSON, so no reader depends on another reader's format. See [Storage](storage.md).

## Beyond customer experience

The record format is not specific to contact centers. Matthew Smith's [vCon and UNS](https://blog.tadsummit.com/2025/12/17/matthew-smith-vcon-and-uns/) talk applies vCon to operator conversations in manufacturing and process industries, describing how it "transforms unstructured conversations into contextualized, queryable, and governed operational knowledge."

## Further reading

* [Concepts](concepts.md): how a chain runs
* [Day in the Life of a vCon](day-in-the-life-of-a-vcon.md): one conversation end to end
* [Articles and Press](../talks-articles-press/articles-and-press.md), [Podcasts](../talks-articles-press/podcasts.md) and [Conference Keynotes](../talks-articles-press/conference-keynotes.md): the wider public record, including Jeff Pulver's Telecom Reseller podcasts on vCon
