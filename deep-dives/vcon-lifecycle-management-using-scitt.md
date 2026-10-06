---
description: >-
  A whitepaper on why a vCon's lawful basis and lifecycle events belong in the
  file and in a SCITT log, and how the two together answer an auditor's
  questions about a conversation.
---

# vCon Lifecycle Management using SCITT

{% hint style="info" %}
**Whitepaper, revised 2026-10-06.** This page gives the rationale. The normative text is in [`draft-howe-vcon-lifecycle`](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/) and [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/), both individual drafts. For field names and event tokens, see the [Lifecycle](../extensions/lifecycle.md) and [Lawful Basis](../extensions/lawful-basis.md) extension pages.
{% endhint %}

## The problem

A single customer call can be recorded by a telephony platform, transcribed by one vendor, scored for sentiment by another, and used to train a model by a third. Each step needs a legal basis, and that basis can change: a person can withdraw consent, a retention period can end, a purpose can be refused.

In most systems the consent record sits in one database and the conversation data sits in several others. When someone asks for access, correction or deletion, the organization has to work out which systems hold a copy and prove afterwards that each one acted. Neither is easy when the records that would answer those questions can be edited by the same people being audited.

## Two records, kept apart

The vCon approach splits the problem in two.

**The vCon carries its own permissions.** Under the Lawful Basis extension, a vCon includes an attachment with `purpose: "lawful_basis"`. Its body states the basis (`consent`, `contract`, `legal_obligation`, `vital_interests`, `public_task` or `legitimate_interests`), an `expiration` (or `null`), and `purpose_grants`, each naming a purpose, whether it was granted, and when. Optional fields add a revalidation interval (`status_interval`), proof of how the basis was established (`proof_mechanisms`), and a pointer to an external `registry` such as a SCITT service. A system that receives the vCon reads this before transcribing, analyzing or training on it. The permissions travel with the data instead of living in a separate database.

**A SCITT log records what happened.** The Lifecycle draft defines events such as `vcon_created`, `vcon_enhanced`, `vcon_sent`, `vcon_received`, `vcon_consent_accepted`, `vcon_consent_revoked`, `vcon_party_redacted`, `vcon_deleted` and `vcon_expired`. Each is registered as a Signed Statement on a [SCITT](scitt-supply-chain-integrity-transparency-and-trust.md) Transparency Service, which returns a Receipt. The draft says these events are not stored in the vCon itself.

The log is append-only and tamper-evident. Once a statement is registered, removing or changing it without detection would require breaking the signatures and hash structure that the Receipts commit to. A Transparency Service can have a single operator; the protection comes from the verifiable log and the Receipts, which a relying party checks without trusting the operator's database. The log can hold hashes of vCons rather than their contents, so it can stay up after the personal data it describes has been deleted.

## How the pieces answer common requests

**"Did this person agree to AI training?"** Read the `purpose_grants` in the vCon's lawful basis attachment. If a registry is named, check the latest consent events for that vCon in the log.

**"Who has a copy?"** Every `vcon_sent` and `vcon_received` event is in the log with a Receipt, so the list of recipients does not depend on each system's own records.

**"Delete it everywhere."** The Lifecycle draft says a Data Controller that deletes a vCon must tell every recipient to delete it or take on Data Controller responsibilities. Each deletion is recorded as `vcon_deleted`, and a recipient that no longer needs a copy can record `vcon_rcvr_purged`.

**"Prove it to the regulator."** Hand over the Receipts. The auditor verifies them against the Transparency Service's public key.

**Data subject rights.** The Lawful Basis draft, not the core draft, requires implementations to support access, rectification, erasure, portability and withdrawal of a lawful basis.

## Limits

* Both drafts are individual submissions, not working group documents, and may change.
* A Receipt proves that a statement was registered. It does not prove that a recipient actually deleted the data. The log makes the claim attributable and permanent; it does not enforce it.
* Systems still have to read and honor the lawful basis attachment. A vCon that arrives without one tells the recipient nothing about what it may do.

## In the Conserver

The Conserver's [`scitt` link](../conserver/standard-links.md) registers Signed Statements for a vCon on a SCRAPI service such as [scittles](https://github.com/vcon-dev/scittles) and can store the Receipt on the vCon. Running one instance before transcription and one after records `vcon_created` and `vcon_enhanced`. The `datatrails` link records events on the DataTrails service.
