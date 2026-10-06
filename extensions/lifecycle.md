---
description: >-
  Explains how to record what happened to a vCon (creation, sharing, consent,
  redaction, deletion) on a SCITT transparency service, so you can prove it
  later.
---

# 🔄 Lifecycle Extension

**Draft:** [`draft-howe-vcon-lifecycle-01`](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/) · **Extension token:** none

## Purpose of the extension

A vCon is a snapshot of one conversation. Regulators and auditors usually ask a different question: what happened to that conversation afterwards, when, and who held it. The lifecycle draft answers that by recording "moments that matter" as operations on a [SCITT](../deep-dives/scitt-supply-chain-integrity-transparency-and-trust.md) transparency service. Each operation gets a SCITT receipt, which is the durable proof that it was recorded.

The draft states that these events "are not intended to be stored within the vCon itself" (Section 5). It defines no vCon parameters and no extension token, and its IANA section reads "This document has no IANA actions" (Section 8). So there is nothing to add to `extensions` or `critical`, and no field to look for in the vCon. The draft does not specify how an event identifies its vCon.

## Lifecycle events

The full list from Section 5:

| Event | Meaning |
| --- | --- |
| `vcon_created` | The vCon was first recorded. Recordings or attachments may be added later. |
| `vcon_enhanced` | The vCon was amended or added to, for example a transcription correction or a consent change. |
| `vcon_sent` | The vCon was sent to an external party. Records what was sent, to whom and when. |
| `vcon_received` | The vCon was received from an external party. |
| `vcon_consent_accepted` | One or more parties consented to the vCon being recorded and shared for its intended purpose. |
| `vcon_consent_revoked` | One or more parties revoked consent for one or more purposes. |
| `vcon_party_redacted` | A party's information was redacted. The vCon can remain usable. |
| `vcon_deleted` | The vCon was deleted, because of revocation or because it is no longer needed. A Data Controller that deletes a vCon must tell every recipient to delete it or take over as Data Controller. |
| `vcon_expired` | The vCon expired under license terms or compliance rules. This triggers deletion and notice to everyone it was shared with. |
| `vcon_rcvr_purged` | A receiving entity no longer needs the vCon, deletes it, and notifies the sender. |

## Flow

Section 4 walks through three roles. A Data Originator (the infrastructure that captured the call) creates the vCon and sends it to a Data Controller. The Controller records `vcon_received`, adds transcription and licensing, and sends it on to Data Processors. A Processor validates the Controller's SCITT receipt, records consent, enhances the vCon, and gives the data subject a way to review and revoke consent.

When a data subject revokes consent, the Controller records the revocation on its transparency service and passes the request to every Processor that has not yet acknowledged it. Each Processor deletes the data. The events stay on the transparency service; that record is the audit trail.

## With Lawful Basis

[Lawful Basis](lawful-basis.md) records, inside the vCon, which processing is allowed. Lifecycle records, outside the vCon, what was done and when. A lawful basis attachment can also name a SCITT registry in its `registry` field.

## See also

* [vCon Lifecycle Management using SCITT](../deep-dives/vcon-lifecycle-management-using-scitt.md) for the longer walkthrough.
* [SCITT: Supply Chain Integrity, Transparency and Trust](../deep-dives/scitt-supply-chain-integrity-transparency-and-trust.md) for background on SCITT.
* [Field reference](../vcons/field-reference.md) for the extension tokens that do exist.
