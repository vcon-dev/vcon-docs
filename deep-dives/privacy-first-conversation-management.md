---
description: >-
  Explains why the legal grounds for processing a conversation belong inside
  the vCon, and how a pipeline checks them before transcribing, analyzing or
  training on it.
---

# Privacy-First Conversation Management

> **Spec note:** This paper was first written against an early `draft-vcon-consent` document. That work is now [`draft-howe-vcon-lawful-basis-02`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/). For exact field definitions see the [Lawful Basis](../extensions/lawful-basis.md) page and the [field reference](../vcons/field-reference.md).

## Summary

Voice and chat conversations are some of the richest personal data a business holds and some of the hardest to govern. Collecting consent is the easy part. Keeping it meaningful as the data moves between systems, gets reused for new purposes, and outlives the conversation is the hard part.

The vCon Lawful Basis extension puts the legal grounds for processing inside the conversation container: the basis, the purposes it covers, its expiration, and optionally the proof of how it was established. Any system that receives the vCon can read that record and gate its own processing on it.

## The problem with separate consent records

Conventional consent management fails in four recurring ways:

1. Consent records live in a different database from the data they govern, so checking compliance means reconstructing state across systems.
2. Permissions are often all-or-nothing, so "yes to transcription, no to model training" cannot be expressed.
3. Audit trails are scattered, so proving compliance becomes an investigation instead of a query.
4. Newer uses such as model training and inference were never modeled in older consent schemas.

A single contact center call may pass through recording, transcription, sentiment analysis and later use as training data. Each step can fall under a different rule set, such as HIPAA, the GDPR or the CCPA. Knowing which permission applies at which step, when it expires, and how it was given is the operational problem this extension addresses.

## The lawful basis attachment

A vCon is a JSON container for one conversation: parties, dialog, analysis and attachments. The extension adds one attachment with `purpose: "lawful_basis"`:

```json
{
  "purpose": "lawful_basis",
  "start": "2025-01-02T12:15:30Z",
  "party": 0,
  "dialog": 0,
  "mediatype": "application/json",
  "encoding": "json",
  "body": {
    "lawful_basis": "consent",
    "expiration": "2026-01-02T12:00:00Z",
    "purpose_grants": [
      { "purpose": "recording", "granted": true, "granted_at": "2025-01-02T12:15:30Z" },
      { "purpose": "transcription", "granted": true, "granted_at": "2025-01-02T12:15:30Z" },
      { "purpose": "ai_training", "granted": false, "granted_at": "2025-01-02T12:15:30Z" }
    ],
    "proof_mechanisms": [
      {
        "proof_type": "verbal_confirmation",
        "timestamp": "2025-01-02T12:15:30Z",
        "proof_data": { "dialog_reference": 0, "confirmation_text": "Yes, I consent to recording this call" }
      }
    ]
  }
}
```

The vCon SHOULD also list `lawful_basis` in its top-level `extensions` array. The extension is Compatible, so the draft does not require it in `critical`. List it there only if readers that cannot evaluate the basis must reject the vCon.

## What the extension provides

**Per-purpose grants.** Each entry in `purpose_grants` names one processing purpose and records whether it was granted, with a timestamp. Granting `transcription` while denying `ai_training` is one attachment. Purpose names are strings; the draft's examples use `recording`, `transcription` and `analysis`.

**The six GDPR bases.** `lawful_basis` is one of `consent`, `contract`, `legal_obligation`, `vital_interests`, `public_task` or `legitimate_interests`. Consent comes from the data subject; the other five are justifications the data controller asserts.

**Expiration and revalidation.** `expiration` is a timestamp or `null`. Readers MUST refuse to process after expiration, allowing for clock skew. `status_interval`, such as `"30d"`, sets how often an open-ended basis is rechecked.

**Proof.** Optional `proof_mechanisms` entries record how the basis was established, with `proof_type` of `verbal_confirmation`, `signed_document`, `cryptographic_signature` (COSE) or `external_system`. An optional body-level `content_hash` lets a reader detect changes to the record, and an optional `registry` points at a SCITT transparency service that holds attestations.

**Data subject rights.** The draft requires implementations to support access, rectification, erasure, portability and withdrawal of a lawful basis (Section 11.3). It cites GDPR Article 7, CCPA Section 1798.135 and the HIPAA Privacy Rule as the requirements it addresses, and leaves compliance in each jurisdiction to the implementer.

## Checking the basis before processing

The draft's processing rules (Section 6) say: skip expired bases, evaluate every grant for the requested purpose, apply the most restrictive, and deny when there is no grant or the grant is `false`. A direct implementation:

```python
import json
from datetime import datetime, timezone
from vcon import Vcon


def allowed(v: Vcon, purpose: str) -> bool:
    """True only if an unexpired lawful basis grants `purpose` and none denies it."""
    now = datetime.now(timezone.utc)
    decisions = []
    for att in v.vcon_dict.get("attachments", []):
        if att.get("purpose") != "lawful_basis":
            continue
        body = att["body"]
        if isinstance(body, str):  # accept older stringified bodies
            body = json.loads(body)
        exp = body.get("expiration")
        if exp and datetime.fromisoformat(exp.replace("Z", "+00:00")) <= now:
            continue
        decisions += [g["granted"] for g in body["purpose_grants"] if g["purpose"] == purpose]
    return bool(decisions) and all(decisions)
```

A pipeline step then gates on it. `transcribe()` stands in for your own speech-to-text call returning a WTF document:

```python
if allowed(v, "transcription"):
    v.add_analysis(
        type="wtf_transcription",
        dialog=0,
        vendor="openai",
        product="whisper-large-v3",
        mediatype="application/json",
        encoding="json",
        body=transcribe(v),
    )
    v.add_extension("wtf_transcription")
```

A training-data filter is one line: `training_set = [v for v in vcons if allowed(v, "ai_training")]`. Each included example then traces back to the grant that allowed it.

The `vcon` library also provides `Vcon.add_lawful_basis_attachment()` to build the attachment and `Vcon.check_lawful_basis_permission(purpose, party_index)` to test a grant. See [Lawful Basis](../extensions/lawful-basis.md#python).

## Pairing with Lifecycle

Lawful Basis records what is allowed, inside the vCon. The [Lifecycle](../extensions/lifecycle.md) draft records what happened, outside it, as events such as `vcon_consent_accepted`, `vcon_consent_revoked` and `vcon_deleted` on a SCITT transparency service. Together they answer what was permitted and what was done.

## Security considerations

From the draft's Section 10:

* Lawful basis attachments MUST be integrity protected with vCon signing. The core draft does not sign anything automatically, so sign the vCon before it leaves your domain.
* Implementations MUST prevent forgery: verify cryptographic proofs, check content hashes of external proof documents, and log validation.
* Implementations MUST prevent replay of a lawful basis from one vCon into another, by binding it to the vCon, validating timestamps, and checking that `party` and `dialog` indexes refer to the right content.
* Attachments with sensitive content SHOULD be encrypted outside secure environments, for example with TLS 1.2 or later in transit.
* Systems SHOULD keep an immutable log of when a basis was given, checked, revoked or expired. A SCITT registry can serve this role.

## Example scenarios

These are illustrations, not descriptions of specific deployments.

**Multi-channel service.** Each chat, call and email becomes a vCon with its own lawful basis attachment. Reporting and analysis steps call `allowed()` for their purpose, so the same pipeline behaves correctly on every channel.

**Telemedicine.** Consent is captured before recording, with separate grants for the clinical record, quality review and de-identified research. Expiration drives deletion, and revocation is recorded as a lifecycle event.

**Training data.** Historical recordings are filtered so only vCons with an unexpired `ai_training` grant reach the training set.

## References

* IETF VCON Working Group: [https://datatracker.ietf.org/wg/vcon/about/](https://datatracker.ietf.org/wg/vcon/about/)
* [`draft-howe-vcon-lawful-basis-02`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/)
* [`draft-howe-vcon-lifecycle-01`](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/)
* SCITT Working Group: [https://datatracker.ietf.org/wg/scitt/about/](https://datatracker.ietf.org/wg/scitt/about/)
* GDPR: [https://gdpr.eu/](https://gdpr.eu/)
* CCPA: [https://oag.ca.gov/privacy/ccpa](https://oag.ca.gov/privacy/ccpa)

Questions about the specification go to the VCON working group list, vcon@ietf.org.
