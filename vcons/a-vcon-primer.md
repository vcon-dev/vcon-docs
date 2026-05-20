---
description: Thomas McCarthy-Howe, CTO, Strolid.
---

# 💬 A vCon Primer

A [vCon](https://datatracker.ietf.org/group/vcon/about/) is a portable, verifiable container for a conversation.  It is currently on the standards track at the premier Internet standards organization, the [IETF](https://www.ietf.org).   The three core specifications are in preparation for working group last call, after over two years of work from dozens of leading engineers, privacy advocates, regulators and operators. \
\
This page explains what one is, why the idea matters now, where vCons are already in production, what they look like inside, and what they let you do that other formats do not. If you read one piece of vCon documentation, read this.

## What a vCon is

A vCon (virtual conversation) is to a conversation what a PDF is to a document, or a vCard is to a business card. Think of it as a sealed folder for a conversation, carrying who was on the call (parties), what was said (the dialog, as recording or transcript), what they agreed to (consent), the notes added since (analysis), and a stamped record of every set of hands it has passed through (a tamper evident history). Inside the cover it is a signed JSON object. The same shape works for a phone call, a chat session, a video meeting, or a human to agent conversation.

The name traces to a casual remark by Brian Galvin, past CTO of both Genesys and Nuance, asking why there was no vCard equivalent for conversations. vCon is the answer. The technical definition lives in the IETF VCON working group, with the spec target [`draft-ietf-vcon-vcon-core`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) and syntax parameter `"vcon": "0.4.0"` (1).

Like PDF and vCard, vCon is open and carries no intellectual property encumbrance. Data formats cannot be patented in most jurisdictions, and vCon was designed that way on purpose.

## Why this matters now

The original use case was contact center recording. Since 2024 the stakes have widened. Four forces are pushing the same direction at once.

**Agentic AI is moving into production.** Agents are starting to talk to customers and to each other. There is no shared record of what an agent said, on whose behalf, or under what authority. Without that record, there is nothing for a regulator, a customer, or a downstream system to verify against.

**Authentic and synthetic are getting harder to tell apart.** A deepfake injected into an AI pipeline is the conversational analog of malware injected into a software supply chain. vCon pairs with SCITT, the IETF effort for Supply Chain Integrity, Transparency and Trust, so creation, sharing, analysis, and deletion are recorded in an append only ledger that cannot be altered after the fact.

**Consent does not travel today.** Consent typically lives in a privacy policy, a recording disclosure, or a screenshot, separate from the conversation it covers. vCon carries consent inside the file itself, scoped by purpose and time. That is the difference between reporting on a privacy policy and enforcing one.

**The silo model is doubling down.** Proprietary contact center, recording, and AI stacks are extending deeper, on architectures that do not interoperate. Without an open container, every enterprise rebuilds the same data prison in a new color every five years.

vCon is built in the open at the IETF, the standards body responsible for TCP/IP, DNS, HTTP, TLS, and SIP. The process is rough consensus, running code, and a public record. Anyone can read the drafts, join the mailing list, and challenge a design decision in writing.

## Where vCon is running today

vCon is past the pilot stage.

The [BPO that incubated the technology](https://www.strolid.com) runs roughly a quarter million vCon formatted conversations per month through its production pipeline, and that volume has roughly doubled over the past year.

A large financial institution is live with millions of vCon productions per day on a path to a million per hour. That deployment is also the first production instance of real-time vCons, where applications follow a conversation as it happens rather than waiting for the recording.

A [prototype is running at a United Way 211 center](https://frontline.group/frontline-group-launches-vcon-pilot/) where the system listens for context that should change routing. A food-banking question and a sexual-abuse disclosure should not sit in the same queue, and they should not have to wait for tomorrow's manager review to be distinguished.

Dozens of [companies are actively building with vCons today](https://www.pulver.com/members). Telecom, contact center, and CPaaS vendors are leaning in first, which is the usual pattern for an open standard. SIP gave service providers recording. vCon gives them a portable answer for what to do with the recording next.

## Inside a vCon

<div align="right"><figure><img src="../.gitbook/assets/Conserver Pictures (8).jpg" alt=""><figcaption><p>The insides of a vCon</p></figcaption></figure></div>

A vCon has five things inside it.

**Dialogues** are the recorded media: audio, video, text, messaging. A vCon can be packed (media inline) for emailing or shipping as one file, or unpacked (media by reference) when the recordings are large enough to live on their own storage.

**Parties** identify who was in the conversation, and who verified their identity. Identity verification is a first class concept, not an afterthought.

**Consent** is carried inside the file, scoped by purpose and duration. When consent is withdrawn or expires, the systems holding the vCon can act on it without consulting an external policy.

**Analysis** holds commentary derived from the dialog: transcription, sentiment, redaction, summarization, model outputs. It is stored as JSON, attachable in layers, and tied to the dialog it refers to.

**Attachments** carry the context the conversation depended on. A sales lead, a CRM record, an inbound form, an authentication challenge, anything that explains why the conversation happened in the first place.

## The hard part

Conversations are simultaneously the most valuable and the most sensitive data a business holds. The value is obvious: every renewal, complaint, sales objection, support edge case, agent error, and customer insight lives in conversation long before it shows up in structured data. Modern ML, agentic AI, compliance audits, and revenue operations all want this material in volume. The sensitivity is just as obvious. Voices and faces are biometric identifiers a customer cannot change. The disclosures inside a conversation routinely include health, finances, family circumstances, and named third parties who never consented to be in the room at all. Treating one side without the other is the trap. Lock the conversations down and the business loses the most important signal it produces. Open them up and the next breach is catastrophic and unrecoverable.

vCon is built to hold both sides at once:

* **Consent** rides inside the file, scoped by purpose and duration, so each downstream system can see what it is allowed to do and refuse to act outside that scope.&#x20;
* **Redaction** is a first class operation, recorded in the analysis layer, so a redacted projection can be produced for one audience while the unredacted original remains controlled and inspectable for another.&#x20;
* **Provenance and integrity** are cryptographic, not procedural, so any version of a vCon can be traced back to who signed it, what was changed, and when.&#x20;
* **SCITT**, the IETF Supply Chain Integrity, Transparency and Trust ledger, records every lifecycle event (creation, sharing, analysis, deletion) in an append only log that downstream auditors can verify on their own without trusting the operator.&#x20;

**None of these mechanisms eliminate the tension**. **They make it manageable, auditable, and verifiable in software, which is the difference between a privacy policy and a privacy posture**.

## What it gets you

* **Privacy and deletion you can actually execute.** vCons make "what data did we capture, and where is it" answerable in a structured way. GDPR style deletion stops being a project and starts being an API call.
* **Audit and provenance.** Every lifecycle event (creation, sharing, analysis, deletion) can be recorded in a SCITT ledger that downstream auditors can verify on their own. The conversation, and what was done to it, are inspectable separately.
* **ML lifecycle hygiene.** When a customer revokes consent or asks for deletion, you need to know which models trained on which conversations. vCons make that traceable, which makes the retraining cost bounded.
* **Consent that travels.** Consent moves with the file. Any system that opens a vCon can see the scope and the expiry, and refuse to act outside them, without phoning home.
* **An interoperable ecosystem.** Because the format is open, independent vendors can supply redaction, validation, transcription, and analytics tools that all read and write the same object. The buyer is not locked into a single stack.

## GDPR, in practice

Every data subject right GDPR grants becomes operable rather than aspirational when conversations live in vCons:

* The **right to be informed** is met by the disclosure stored at capture.&#x20;
* The **right of access** becomes a query against a structured object instead of a hunt across systems.&#x20;
* **Rectification** lands as an additional analysis entry with provenance rather than an overwrite.
* **Erasure** can be issued by the Conserver across every storage location holding the vCon, with the deletion event recorded in SCITT.&#x20;
* **Restriction of processing** follows the consent scope inside the file, which downstream systems can read directly and refuse to act outside of.&#x20;
* **Portability** is the format's defining trait, so a subject access request can return the conversations themselves rather than a flat export.&#x20;
* **The right to object** travels with the file, since revocation of consent propagates rather than waiting on a separate policy.&#x20;
* And for the **rights around automated decision making**, the analysis layer records every model that touched the conversation, so the subject can be told which decisions used their data.

## What to read next

* [vCons are...](vcons-are....md) for the one paragraph mental model
* [Concepts](concepts.md) for the deeper vocabulary
* [Privacy Primer](privacy-primer.md) for the lawful basis and PII story
* [Conserver Quick Start](../conserver/conserver-quick-start.md) to run a vCon pipeline on your machine
* [IETF VCON working group](https://datatracker.ietf.org/group/vcon/about/) for the primary spec record

## Foot Notes

1. The IETF VCON working group page is at [datatracker.ietf.org/group/vcon/about/](https://datatracker.ietf.org/group/vcon/about/). The core spec target is [`draft-ietf-vcon-vcon-core`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) with syntax parameter `"vcon": "0.4.0"`.
2. In GDPR terminology, a "natural person" is an individual human being, as opposed to a legal person such as a corporation.
