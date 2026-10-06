---
description: >-
  Explains what SCITT is, which IETF documents define it, and how vCon
  systems use a SCITT transparency service to prove what happened to a
  conversation.
---

# SCITT: Supply Chain Integrity, Transparency and Trust

SCITT is an IETF architecture for publishing signed statements about an artifact to an append-only, tamper-evident log, and getting back a receipt that anyone can verify later. It was designed for software supply chains. vCon uses it to record what happened to a conversation: when it was created, shared, consented to, redacted or deleted.

## The SCITT documents

* **Architecture.** [RFC 9943](https://www.rfc-editor.org/rfc/rfc9943.html), "An Architecture for Trustworthy and Transparent Digital Supply Chains", published June 2026 from [`draft-ietf-scitt-architecture`](https://datatracker.ietf.org/doc/draft-ietf-scitt-architecture/).
* **Receipts.** [RFC 9942](https://www.rfc-editor.org/rfc/rfc9942.html), "CBOR Object Signing and Encryption (COSE) Receipts", published June 2026 from [`draft-ietf-cose-merkle-tree-proofs`](https://datatracker.ietf.org/doc/draft-ietf-cose-merkle-tree-proofs/).
* **REST API.** [`draft-ietf-scitt-scrapi`](https://datatracker.ietf.org/doc/draft-ietf-scitt-scrapi/), the SCITT Reference API (SCRAPI) used to register statements and fetch receipts.
* **Signatures.** COSE, [RFC 9052](https://www.rfc-editor.org/rfc/rfc9052.html).

## The terms

**Issuer.** The organization, device or service that signs a statement.

**Signed Statement.** A statement about an artifact, signed by its Issuer with COSE. The payload can be the artifact or, more often, a hash of it.

**Transparency Service.** The service that checks a Signed Statement against its registration policy, appends it to a verifiable data structure, and returns a Receipt. RFC 9943 does not require a distributed network. A Transparency Service can be run by a single operator; the log's structure, not the number of operators, is what makes later tampering detectable.

**Receipt.** A COSE object, defined by RFC 9942, that proves a Signed Statement is included in the log. A Signed Statement with its Receipt is a Transparent Statement.

**Relying Party.** Anyone who later verifies the Transparent Statement, such as an auditor or a downstream recipient of the vCon. Verification needs the Receipt and the service's public key, not access to the operator's database.

## How vCon uses SCITT

A vCon is the record of a conversation. SCITT is the record of what was done with it. The vCon itself does not need to be published: the Signed Statement normally carries a hash of the vCon, so the log proves which version existed at what time without exposing its contents.

* **[Lifecycle extension](../extensions/lifecycle.md)** ([`draft-howe-vcon-lifecycle`](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/)) names the events to record, such as `vcon_created`, `vcon_sent`, `vcon_consent_revoked` and `vcon_deleted`, as operations on a SCITT Transparency Service. The events live in the log, not in the vCon.
* **[Lawful Basis extension](../extensions/lawful-basis.md)** ([`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/)) can point to a SCITT Transparency Service implementing SCRAPI as the registry for lawful basis attestations.
* **Conserver links.** The [`scitt` link](../conserver/standard-links.md) hashes a vCon, signs a statement with COSE, registers it on a SCRAPI service such as [scittles](https://github.com/vcon-dev/scittles), verifies the returned receipt and can store it on the vCon as an analysis entry of type `scitt_receipt`. The `datatrails` link records vCon events on the DataTrails service. Both are listed on [Standard Links](../conserver/standard-links.md).

For the reasoning behind this design, see [vCon Lifecycle Management using SCITT](vcon-lifecycle-management-using-scitt.md).
