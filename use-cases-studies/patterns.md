---
description: Describes the problems vCon is designed to solve that have no public deployment yet, and points each one to the vCon feature and the public build that demonstrates it.
---

# Patterns

These are patterns, not case studies. Each describes a problem the vCon format was designed for and the part of the spec that addresses it. None is backed by a public production deployment yet. Where a public prototype exists, it is linked; for documented deployments see [Use Cases / Studies](README.md).

## Moving conversation history between providers

When a business changes contact-center or UC provider, or merges with another company, its recorded conversations usually sit in the old provider's format. A vCon is one self-contained JSON record per conversation, with the media inline or linked by `url` and [`content_hash`](../vcons/field-reference.md#dialog-object), so a migration becomes an export of vCons from one system and an import into the other.

Shown in prototype by ConvoSense and vChat at [VCONIC TADHack 2026](tadhack-builds.md), which converted conversations from several platforms and messaging apps into vCon.

## Privacy and data-subject rights

Recorded conversations are personal data, and often regulated data. Under HIPAA, health information is defined as information "whether oral or recorded in any form or medium" ([45 CFR 160.103](https://www.law.cornell.edu/cfr/text/45/160.103)), so a recorded call about a patient's care is covered just as a written record is. Privacy enforcement can be expensive: the FTC imposed a [$5 billion penalty on Facebook in July 2019](https://www.ftc.gov/news-events/news/press-releases/2019/07/ftc-imposes-5-billion-penalty-sweeping-new-privacy-restrictions-facebook).

vCon gives a conversation a place to record why it may be held and what was done to it:

* The [Lawful Basis extension](../extensions/lawful-basis.md) attaches the basis for processing, the purposes granted, and an expiration.
* A [`redacted`](../vcons/field-reference.md#top-level-object) vCon points to the less-redacted version it was derived from, so a redacted copy can be shared while the original stays controlled.
* The [Lifecycle extension](../extensions/lifecycle.md) records consent revocation, deletion and expiry on a SCITT transparency service, giving an audit trail of what was deleted and who was told.
* The [Privacy Primer](../vcons/privacy-primer.md) sets out the privacy scope of vCon.

Shown in prototype by ConsentMate at [VCONIC TADHack 2026](tadhack-builds.md). All three [public corpora](public-corpora.md) carry a `lawful_basis` attachment.

## Sharing conversations with partners

A conversation often needs to reach another organization: an outsourcer, an analytics vendor, a regulator. Because a vCon is a complete record in a standard format, the recipient needs no integration with the platform it came from. A conserver accepts partner submissions on [`/vcon/external-ingress`](../conserver/operational-benefits-of-conservers.md#many-tenants-many-conservers) with a key scoped to that partner, and the [signed form](../vcons/field-reference.md#signed-and-encrypted-forms) lets the recipient check the record was not altered.

## Comparing transcription engines

Each [`analysis`](../vcons/field-reference.md#analysis-object) entry names the `vendor` and optionally the `product` that produced it, and a vCon can hold several transcripts of the same dialog. That makes a set of vCons a test set: run the same recordings through several engines and compare the results entry by entry. The [IETF corpus](public-corpora.md#ietf-meeting-sessions) records which of three engines produced each transcript in the `vendor` field. For generating synthetic test data, see [vcon\_faker](../tools/vcon-faker.md).

## Certifying a conversation

A party with first-hand knowledge of a call, such as the network that carried it, can sign a statement about the vCon and register it on a SCITT transparency service. The conserver [`scitt` link](../conserver/standard-links.md#scitt) does this and stores the receipt in the vCon. See [SCITT and vCon](../deep-dives/scitt-supply-chain-integrity-transparency-and-trust.md). The [211 pilot](contact-center-211-pilot.md) sources mention SCITT-verified outcome reporting.

## Analysis across many conversations

Call recordings are usually reviewed one at a time. Once each conversation is a vCon with transcripts, summaries and tags in `analysis`, a whole corpus can be searched and aggregated by any tool that reads the format, such as the [vCon MCP server](../mcp-server/what-is-the-vcon-mcp-server.md). Most [TADHack 2026 builds](tadhack-builds.md) took this route.
