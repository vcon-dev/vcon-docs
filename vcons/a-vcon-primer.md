---
description: >-
  Explains what a vCon is, why conversations need their own file format, what
  sits inside one, and where vCon work stands at the IETF, for readers new to
  the standard.
---

# 💬 A vCon Primer

_By Thomas McCarthy-Howe, CTO, Strolid._

A vCon (virtual conversation) is a portable, verifiable container for one conversation. It is a JSON object that holds who took part, what was said, what has been learned from it since, and the files that explain why it happened. The format is being standardized in the [IETF VCON working group](https://datatracker.ietf.org/group/vcon/about/) as [`draft-ietf-vcon-vcon-core`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), currently revision 04 with syntax `"vcon": "0.4.0"`.

## What a vCon is

A vCon is to a conversation what a PDF is to a document or a [vCard](https://datatracker.ietf.org/doc/html/rfc6350) is to a business card. The same shape works for a phone call, a chat session, an email thread, a video meeting, or a conversation between a person and an AI agent. A simple example is the last call you had with a customer service agent: the vCon identifies the people on the call, carries the recording or transcript, holds analysis such as a summary, and attaches supporting documents.

A vCon can be built after the conversation ends or updated while it is still in progress. It can be stored on disk, attached to an email, or sent across a network. When it leaves the security domain that built it, the core draft says it SHOULD be signed (JWS) or encrypted (JWE). A signed vCon is tamper-evident: any change after signing invalidates the signature. Signing is a recommendation, so an unsigned vCon is still a valid vCon.

## Why conversations need a file

Conversations carry decisions, negotiations, commitments and complaints, yet they have rarely existed as one coherent digital object. A recording lives in one platform, the transcript in another, consent records somewhere else, and AI summaries get copied into a CRM. Every tool sees a fragment.

That was tolerable while people reviewed a handful of calls. It stops being tolerable when models and agents summarize, classify and act on every conversation. A model reasoning over a summary of a transcript that has been detached from its recording and its consent record is working from a partial copy, and it fills the gaps by inference.

Other kinds of content crossed this line long ago. Documents got portable formats, calendars got iCalendar, contacts got vCard, and ecosystems formed around each. vCon proposes the same for conversations: one object that keeps the media, the participants, the analysis and the related records together as the conversation moves between systems. The [Conserver](../conserver/conserver-introduction.md) is open source infrastructure that builds, enriches, stores and forwards those objects.

Three pressures make this more urgent now.

* **AI agents talk to customers.** There is no shared record of what an agent said, on whose behalf, or under what authority. A vCon can hold that record, including the agent as a party.
* **Synthetic media is harder to spot.** A fabricated recording fed into an AI pipeline is the conversational analog of malware injected into a software supply chain. A signed vCon lets a recipient check who produced it and whether it changed.
* **Permissions do not travel today.** Consent usually lives in a privacy policy or a separate database, apart from the conversation it covers. The [Lawful Basis extension](../extensions/lawful-basis.md) puts the legal grounds for processing inside the vCon, scoped by purpose and time, so any system that receives it can check before acting.

## Inside a vCon

<div align="right"><figure><img src="../.gitbook/assets/Conserver Pictures (8).jpg" alt=""><figcaption><p>The insides of a vCon</p></figcaption></figure></div>

Core-04 defines four arrays inside the top-level object, alongside `uuid`, `created_at` and the `vcon` syntax version. The [field reference](field-reference.md) lists every field.

**Parties** identify who took part, human or bot, by `tel`, `sip`, `mailto`, `did`, name and organization. A `validation` field records how identity was checked.

**Dialog** holds the conversation itself: recordings, video, text messages. Media can be inline, for a vCon that ships as one file, or referenced by URL with a `content_hash`, when the recordings live on their own storage.

**Analysis** holds what has been derived from the dialog: transcripts, summaries, sentiment, model outputs. Each entry names the dialog it was derived from and the vendor, and optionally the product, that produced it.

**Attachments** carry the context the conversation depended on, such as a sales lead, a CRM record or an inbound form. Extensions use attachments too. A lawful basis record is an attachment with `purpose: "lawful_basis"`.

Consent is not a core component. It is defined by the Lawful Basis extension, one of several [extensions](../extensions/README.md) that build on the core container.

A vCon can also be redacted or amended. A redacted vCon removes data for a given audience and points back to the original, so a recipient can tell it was derived without seeing what was removed. An amended vCon adds to a prior version and references it.

## The hard part

Conversations are among the most valuable and the most sensitive data a business holds. Renewals, complaints, objections and agent errors show up in conversation long before they reach structured data. Voices and faces are biometric identifiers a person cannot change, and conversations routinely include health, financial and family details, plus named third parties who never agreed to be discussed.

vCon gives a business tools for holding both sides:

* **Lawful basis in the file.** With the Lawful Basis extension, each downstream system can see what processing is permitted and refuse to act outside that scope.
* **Redaction as a defined operation.** A redacted vCon can go to one audience while the unredacted original stays controlled.
* **Signatures.** A signed vCon shows who signed it and whether it has changed since.
* **A lifecycle record.** The [Lifecycle extension](../extensions/lifecycle.md) records events such as creation, sharing, consent revocation and deletion on a [SCITT](../deep-dives/scitt-supply-chain-integrity-transparency-and-trust.md) transparency service. The log is append-only and tamper-evident, and an auditor can verify receipts without trusting the operator's database.

None of this removes the tension between value and sensitivity. It makes the handling auditable in software instead of in policy documents.

## Data subject rights in practice

When conversations live in vCons, the rights in the GDPR and similar laws map onto concrete operations. The [Lawful Basis draft](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/) is the document that requires implementations to support them; the core draft does not.

* **Access.** A query against structured objects instead of a search across systems.
* **Rectification.** A new analysis entry or an amended vCon, recorded alongside the original.
* **Erasure.** Deletion across every store that holds the vCon, with a `vcon_deleted` event recorded under the Lifecycle extension.
* **Restriction and objection.** The purposes granted in the lawful basis record, which downstream systems can read before processing.
* **Portability.** The vCon itself, in an open format.
* **Automated decision making.** The analysis entries name the vendors and products that processed the conversation.

The [Privacy Primer](privacy-primer.md) covers the vocabulary. In GDPR terms a "natural person" is an individual human being, as opposed to a legal person such as a company.

## Where vCon is running

The [TADSummit vCon Progress Report](https://blog.tadsummit.com/2025/08/20/vcon-progress-report/) (August 2025) describes an alliance between Strolid and Frontline Group and a vCon pilot with 211 contact centers, the U.S. social services help line. Frontline's own announcement is [here](https://frontline.group/frontline-group-launches-vcon-pilot/). Other public talks and articles are collected under [Talks, Articles and Press](../talks-articles-press/).

## Standards status

As of 2026-10-06, datatracker lists four VCON working group documents, all in the "WG Document" state: [`draft-ietf-vcon-vcon-core-04`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), [`draft-ietf-vcon-overview-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-overview/), [`draft-ietf-vcon-privacy-primer-01`](https://datatracker.ietf.org/doc/draft-ietf-vcon-privacy-primer/) and [`draft-ietf-vcon-cc-extension-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-cc-extension/). The extensions on this site (lawful basis, lifecycle, WTF transcription, agent session, SIP signaling) are individual drafts.

No IPR disclosures had been filed against `draft-ietf-vcon-vcon-core` as of the same date, according to the [datatracker IPR search](https://datatracker.ietf.org/ipr/search/?draft=draft-ietf-vcon-vcon-core&submit=draft). Anyone can read the drafts, join the [mailing list](https://datatracker.ietf.org/group/vcon/about/) and raise an objection in writing.

## What to read next

* [Concepts](concepts.md) for the vocabulary
* [Field Reference](field-reference.md) for every field and draft revision
* [Privacy Primer](privacy-primer.md) for lawful basis and personal data
* [The Journey of a vCon](../conserver/vcon-conveyor-infographic.md) for a vCon moving through a pipeline
* [Conserver Quick Start](../conserver/conserver-quick-start.md) to run a pipeline on your machine
