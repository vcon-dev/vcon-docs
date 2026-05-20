---
description: A Technical Whitepaper on Standardized Lawful Basis in Virtualized Conversations
---

# Privacy-First Conversation Management

> **Spec note:** This whitepaper was originally written against an early `draft-vcon-consent` document. That work has been superseded by [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/), which formalizes the model described here. For the current spec surface and JSON shapes, see [Lawful Basis](../extensions/lawful-basis.md) and [Lifecycle](../extensions/lifecycle.md). The rationale and patterns in this whitepaper remain valid.

## Executive Summary

Voice and chat conversations are some of the richest personal data a business handles, and also some of the hardest to govern. The real problem is not collecting consent. It is keeping that consent meaningful as the data moves between systems, gets re-used for new purposes, and outlives the conversation it came from.

The vCon [Lawful Basis extension](../extensions/lawful-basis.md) addresses this by embedding the legal grounds for processing directly inside the conversation container. The lawful basis, the purposes it covers, an expiration, and a cryptographic proof all travel with the data. The result is a machine-checkable record of why processing is allowed, available wherever the conversation goes.

## The Problem: Consent in the AI Era

Traditional consent management has four recurring failure modes:

1. Consent records sit in a separate database from the data they govern, so compliance checks require reconstruction across systems.
2. Permissions are binary, which makes nuanced cases (yes to transcription, no to model training) impossible to express.
3. Audit trails are scattered, so proving compliance is an investigation rather than a query.
4. AI-era use cases such as model training, inference, and text-and-data-mining were never modeled in older consent schemas.

Consider a healthcare contact center. A single patient call may pass through call recording, transcription, sentiment analysis, and eventual use as training data for a future assistant. Each step lives under a different rule (HIPAA for healthcare data, GDPR for EU residents, CCPA for Californians, sectoral rules for AI training). Tracking which permissions apply at which step, when they expire, and how to prove they were given is the operational problem that the Lawful Basis extension exists to solve.

## The vCon Solution

A vCon (Virtualized Conversation) is a standardized JSON container that packages everything related to a single conversation: the parties, the dialog content (audio, text, video), any analysis (transcripts, sentiment, summaries), and a list of typed attachments.

The Lawful Basis extension adds a structured attachment to that list. The attachment declares the legal grounds for processing the vCon, the specific purposes those grounds cover, when the basis expires, and how the basis was established. Because the attachment lives inside the vCon, consent travels with the data and cannot be separated from it.

A minimal attachment looks like this:

```json
{
  "type": "lawful_basis",
  "encoding": "json",
  "party": 0,
  "dialog": 0,
  "body": {
    "lawful_basis": "consent",
    "expiration": "2026-01-02T12:00:00Z",
    "purpose_grants": [
      { "purpose": "recording",     "granted": true, "granted_at": "2025-01-02T12:15:30Z" },
      { "purpose": "transcription", "granted": true, "granted_at": "2025-01-02T12:15:30Z" },
      { "purpose": "analysis",      "granted": true, "granted_at": "2025-01-02T12:15:30Z" }
    ],
    "proof_mechanisms": [
      {
        "mechanism_type": "audio_recording",
        "dialog_index": 0,
        "description": "Verbal consent captured at start of recording"
      }
    ]
  }
}
```

The top-level vCon must also declare the extension:

```json
{
  "vcon": "0.4.0",
  "extensions": ["lawful_basis"],
  "must_understand": ["lawful_basis"]
}
```

Placing `lawful_basis` in `must_understand` is the safer default. A consumer that does not understand the lawful-basis model should refuse the vCon rather than silently lose the consent record.

## Key Features

### Granular, purpose-based permissions

Each entry in `purpose_grants[]` names a specific processing purpose and records whether it was granted. Common purposes include `recording`, `transcription`, `analysis`, `storage`, `redistribution`, and AI-oriented purposes such as model training and inference. Granting `transcription` while denying `ai_training` is a single attachment, not a separate document.

### Six GDPR-aligned bases

The `lawful_basis` field takes one of the six GDPR bases: `consent`, `contract`, `legal_obligation`, `vital_interests`, `public_task`, or `legitimate_interests`. The first five require an expiration. `legitimate_interests` may set `expiration: null` for ongoing grounds. Aligning to GDPR's six bases makes the extension legible to regulators while remaining usable under CCPA, HIPAA, and similar regimes.

### Cryptographic verification

Each attachment carries at least one entry in `proof_mechanisms[]` describing how the basis was established. The mechanism can be an audio segment in the vCon itself, a reference to an external system of record, or a signed document. When paired with the [Lifecycle extension](../extensions/lifecycle.md), consent acceptance and revocation events are also recorded on a SCITT transparency ledger, producing a tamper-evident, third-party-verifiable audit trail.

### Temporal management

Expiration timestamps invalidate consent automatically. Revalidation intervals can require periodic refresh for sensitive purposes. Clock-skew handling and indefinite-but-revalidated grants are first-class concerns in the spec rather than ad-hoc patches in application code.

### Regulatory compliance

The extension is designed to satisfy the operational requirements of the major privacy regimes without forcing a separate compliance stack:

* **GDPR.** Right of access (read the attachment), right of rectification (update or replace it), right to be forgotten (revoke via Lifecycle), right to portability (export the vCon).
* **CCPA.** Right to know, right to delete, opt-out of sale, non-discrimination, all expressible in `purpose_grants` and Lifecycle events.
* **HIPAA.** Per-purpose authorization, audit trail, breach support, all attached to the conversation itself.

## Technical Implementation

### Real-time verification at processing time

Processing pipelines read the attachment and gate behavior on the relevant purpose:

```python
from vcon import Vcon

def process_conversation(v: Vcon) -> Vcon:
    basis = next(
        (a for a in v.vcon_dict.get("attachments", [])
         if a.get("type") == "lawful_basis"),
        None,
    )
    if basis is None:
        return v  # no basis declared; do not process

    body = basis["body"]
    grants = {g["purpose"]: g["granted"] for g in body["purpose_grants"]}

    if grants.get("transcription"):
        v.add_analysis(transcribe_audio(v.dialog[0]))

    if not grants.get("ai_training", False):
        v.add_processing_restriction("no_ai_training")

    return v
```

### Building the attachment

The Python `vcon` library provides `add_lawful_basis_attachment()`, but appending the dict directly is often simpler:

```python
v.vcon_dict["attachments"].append({
    "type": "lawful_basis",
    "encoding": "json",
    "party": 0,
    "dialog": 0,
    "body": {
        "lawful_basis": "consent",
        "expiration": "2026-01-02T12:00:00Z",
        "purpose_grants": [
            {"purpose": "recording",     "granted": True, "granted_at": "2025-01-02T12:15:30Z"},
            {"purpose": "transcription", "granted": True, "granted_at": "2025-01-02T12:15:30Z"},
        ],
        "proof_mechanisms": [
            {"mechanism_type": "audio_recording", "dialog_index": 0,
             "description": "Verbal consent captured at start of recording"},
        ],
    },
})
v.add_extension("lawful_basis")
```

### Pairing with Lifecycle

Lawful Basis declares the legal grounds. The [Lifecycle extension](../extensions/lifecycle.md) records the events: when consent was accepted, modified, or revoked, each anchored on a SCITT transparency service. Used together, the two answer both "what was allowed" and "what happened, and when, provably." Most production deployments use both.

### Security considerations

Signatures use COSE (CBOR Object Signing and Encryption) with certificate-chain validation for signing authority. External proof references include content-integrity hashes. Consent ledger traffic uses TLS 1.2 or later with certificate pinning for critical services. Access to consent data follows least privilege and is audit-logged.

## Use Cases

### Multi-channel customer service

A telco handles inquiries across voice, chat, and email. Each interaction is a vCon with its own Lawful Basis attachment. When a chat escalates to a voice call, the basis travels with the case file. AI analysis and reporting gate themselves on `purpose_grants`, so the same data pipeline behaves correctly across channels with no parallel compliance system.

### Healthcare telemedicine

A telemedicine platform records video consultations with patients in multiple jurisdictions. Consent is captured before recording starts, with separate grants for clinical record, quality review, and de-identified research. Expiration drives automatic deletion. The same vCon satisfies HIPAA for US patients and GDPR for EU patients without forking the data pipeline.

### AI training data

A vendor builds conversational AI from historical customer-service recordings. Lawful Basis attachments mark which conversations carry training permission. A filter in the training pipeline reads `purpose_grants` and includes only conversations where `ai_training` is granted and the basis is unexpired. The resulting model has an auditable provenance trail: a regulator can trace any training example back to the consenting interaction.

## Conclusion

Lawful Basis turns privacy compliance from a paperwork exercise into a property of the data itself. The legal grounds, the purposes they cover, the expiration, and the proof all live inside the vCon, where any downstream system can read them, gate on them, and verify them. Combined with Lifecycle's SCITT-anchored event log, the result is a privacy posture that scales with AI-era data use rather than collapsing under it.

***

#### References and Further Reading

* IETF vCon Working Group: [https://datatracker.ietf.org/wg/vcon/](https://datatracker.ietf.org/wg/vcon/)
* `draft-howe-vcon-lawful-basis`: [https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/)
* SCITT Working Group: [https://datatracker.ietf.org/wg/scitt/](https://datatracker.ietf.org/wg/scitt/)
* Lawful Basis extension (this docs site): [extensions/lawful-basis.md](../extensions/lawful-basis.md)
* Lifecycle extension (this docs site): [extensions/lifecycle.md](../extensions/lifecycle.md)
* GDPR Compliance Guide: [https://gdpr.eu/](https://gdpr.eu/)
* CCPA Resource Center: [https://oag.ca.gov/privacy/ccpa](https://oag.ca.gov/privacy/ccpa)

_For questions about implementation or to contribute to the specification, contact the vCon working group at vcon@ietf.org_
