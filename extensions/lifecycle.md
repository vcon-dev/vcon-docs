---
description: Append-only audit ledger anchored in SCITT for vCon creation, transmission, consent, and deletion events.
---

# 🔄 Lifecycle Extension

**Draft:** [`draft-howe-vcon-lifecycle`](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/) · **Extension name:** `"lifecycle"`

## What it is

A vCon is a snapshot. The question regulators and auditors usually want to answer — "what happened to this conversation, when, and who saw it?" — needs more than a snapshot. The Lifecycle extension answers that question by anchoring vCon events in an external [SCITT (Supply Chain Integrity, Transparency, and Trust)](../deep-dives/scitt-supply-chain-integrity-transparency-and-trust.md) ledger.

Unlike most extensions, **Lifecycle adds no per-vCon fields**. The vCon itself stays small. The lifecycle data lives on an append-only, cryptographically verifiable SCITT ledger that the vCon and its downstream consumers reference.

## When to use it

- GDPR compliance: demonstrate the chain of consent acceptance, transfers, and (eventually) deletion in response to a Right to Erasure request.
- CCPA / state-level privacy laws with similar audit requirements.
- Internal compliance for regulated industries (healthcare, financial services) where conversation handling has to be auditable end-to-end.
- Multi-party processing pipelines (call recording → transcription → analysis → CRM) where each handoff needs an independent, tamper-evident record.

## Spec surface

Lifecycle defines a vocabulary of event types that get recorded on a SCITT ledger. The events the draft enumerates include:

| Event | When to record it |
|-------|-------------------|
| `vcon_created` | New vCon assembled and signed |
| `vcon_enhanced` | An analysis or attachment was added |
| `vcon_sent` | The vCon (or a derived projection) was transmitted to another party |
| `vcon_received` | The vCon was received from another party |
| `vcon_consent_accepted` | Consent (lawful basis) was recorded for the conversation |
| `vcon_consent_revoked` | A party revoked consent — triggers downstream cleanup |
| `vcon_deleted` | The vCon was deleted in response to a Right to Erasure request |

Each event entry on the SCITT ledger references the vCon's UUID, includes a timestamp, and is signed by the party producing it. The SCITT receipt that comes back from the ledger is the durable proof that the event happened.

To signal that a vCon participates in this lifecycle scheme, declare the extension:

```json
{
  "vcon": "0.4.0",
  "extensions": ["lifecycle"]
}
```

You generally do **not** put `"lifecycle"` in `must_understand[]` — consumers that don't speak SCITT can still process the vCon, they just won't validate the audit trail.

## Operational pattern

A typical flow:

1. **Create.** Build the vCon, sign it, post a `vcon_created` entry to the SCITT ledger. Store the receipt alongside the vCon.
2. **Consent.** When you add a [lawful basis attachment](lawful-basis.md), post a `vcon_consent_accepted` entry.
3. **Process.** Each time you add analysis or share the vCon, post `vcon_enhanced` / `vcon_sent` / `vcon_received` entries.
4. **Revocation.** If a data subject revokes consent, post `vcon_consent_revoked`. Downstream consumers watching the ledger trigger their own deletion workflows.
5. **Deletion.** Post `vcon_deleted` after the vCon (and any derived data) has been removed. The deletion event itself remains on the ledger forever — that's the audit trail.

## See also

- [vCon Lifecycle Management using SCITT](../deep-dives/vcon-lifecycle-management-using-scitt.md) — the long-form rationale and walkthrough.
- [SCITT: Supply Chain Integrity, Transparency and Trust](../deep-dives/scitt-supply-chain-integrity-transparency-and-trust.md) — what SCITT is and why it fits here.
- [Lawful Basis](lawful-basis.md) — the consent record that lifecycle events reference.
