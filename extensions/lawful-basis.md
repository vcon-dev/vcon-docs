---
description: >-
  Shows how to record the legal grounds for processing a conversation inside
  the vCon, so every downstream system can check what it is allowed to do.
---

# ⚖️ Lawful Basis Extension

**Draft:** [`draft-howe-vcon-lawful-basis-02`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/) · **Extension token:** `lawful_basis`

## Purpose of the extension

Privacy law such as the GDPR requires a documented lawful basis before personal data is processed. The Lawful Basis extension puts that record inside the vCon as one attachment. It says which basis applies, which processing purposes are granted or denied, when the basis expires, and optionally how it was proven. Because the record travels with the conversation, any system that receives the vCon can check it before acting.

The extension is Compatible (draft Section 4.1). A reader that does not support it can ignore the attachment and still process the rest of the vCon.

## Attachment shape

This vCon is a reduction of the draft's examples in Sections 4.3 and 5.3:

```json
{
  "vcon": "0.4.0",
  "uuid": "01a1125f-3397-860d-832a-bc92ac6830cd",
  "created_at": "2025-01-02T12:00:00Z",
  "extensions": ["lawful_basis"],
  "parties": [
    { "tel": "+12025550100", "name": "Alice" },
    { "tel": "+12025550199", "name": "Bob" }
  ],
  "dialog": [
    {
      "type": "recording",
      "start": "2025-01-02T12:14:07Z",
      "parties": [0, 1],
      "mediatype": "audio/x-wav",
      "url": "https://example.com/recordings/call-1.wav",
      "content_hash": "sha512-GLy6IPaIUM1GqzZqfIPZlWjaDsNgNvZM0iCONNThnH0a75fhUM6cYzLZ5GynSURREvZwmOh54-2lRRieyj82UQ"
    }
  ],
  "attachments": [
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
          { "purpose": "sentiment_analysis", "granted": false, "granted_at": "2025-01-02T12:15:30Z" }
        ],
        "proof_mechanisms": [
          {
            "proof_type": "verbal_confirmation",
            "timestamp": "2025-01-02T12:15:30Z",
            "proof_data": {
              "dialog_reference": 0,
              "time_offset": "00:01:23",
              "confirmation_text": "Yes, I consent to recording this call"
            }
          }
        ]
      }
    }
  ]
}
```

The `content_hash` above is a placeholder copied from the core draft's examples, not the hash of a real file. The draft's own attachment examples omit `mediatype`; it is included here because core-04 Section 4.4.5 requires it for inline attachments.

## Attachment fields

From draft Section 5.1:

* `purpose` MUST be `"lawful_basis"`. Not `type`.
* `encoding` MUST be `"json"`, and `body` is the JSON object itself.
* `party` is the index of the data subject's party. Use `0` when no specific party applies.
* `dialog` is the related dialog index. Use `0` when no specific dialog applies.
* `start` SHOULD give the time the basis was recorded.

## Body fields

**Required** (Section 5.2.1):

* `lawful_basis`: one of `consent`, `contract`, `legal_obligation`, `vital_interests`, `public_task`, `legitimate_interests`.
* `expiration`: RFC 3339 timestamp, or `null` for no fixed expiry. A `null` expiration is still subject to revalidation through `status_interval`.
* `purpose_grants`: array of grants. Each grant MUST have `purpose` (string), `granted` (boolean, `false` records a denial) and `granted_at` (timestamp). `conditions` is an optional array of strings.

**Optional** (Section 5.2.2):

* `proof_mechanisms`: array of proofs, each with `proof_type`, `timestamp` and `proof_data` (an object whose contents depend on the type). Defined types: `verbal_confirmation`, `signed_document`, `cryptographic_signature`, `external_system`.
* `terms_of_service`: URL.
* `status_interval`: revalidation interval such as `"30d"`.
* `content_hash`: an object with `algorithm` (`sha-256`, `sha-3-256` or `blake2b-256`), `canonicalization` (`jcs`) and a hex `value`, computed over the body. This is separate from the core `content_hash` string used for external files.
* `registry`: an object with `type` (`scitt`) and `url`, pointing at a transparency service that holds attestations.
* `metadata`: implementation-specific data.

## Declaring the extension

vCons with a lawful basis attachment SHOULD list `lawful_basis` in `extensions` (Section 4.3). The draft says the token does not need to go in `critical` (Section 4.1). You may still list it in `critical` if your application must stop readers that cannot evaluate the basis; core-04 allows that, and such readers will then reject the vCon.

## Processing rules

Section 6 tells readers what to do before acting on a vCon:

* Reject processing when `expiration` has passed, allowing a small clock skew (five minutes is the recommended maximum).
* Check that `party` and `dialog` indexes exist.
* For a requested purpose, evaluate every applicable grant, apply the most restrictive, and deny when no grant exists or the grant is `false`.
* Verify `content_hash` when present, and SHOULD verify proof mechanisms.

## Python

The [`vcon`](https://pypi.org/project/vcon/) library (0.10.0) builds this attachment and fills in `purpose`, `encoding`, `party`, `dialog`, `mediatype` and `start`. `Vcon.build_new()` already sets `"vcon": "0.4.0"`, and the helper adds `lawful_basis` to `extensions`. Proof mechanisms must be `ProofMechanism` objects, not dicts.

```python
from vcon import Vcon
from vcon.party import Party
from vcon.extensions.lawful_basis.attachment import ProofMechanism, ProofType

v = Vcon.build_new()
v.add_party(Party(tel="+12025550100", name="Alice"))

v.add_lawful_basis_attachment(
    lawful_basis="consent",
    expiration="2027-10-06T14:00:00Z",
    purpose_grants=[
        {"purpose": "recording", "granted": True, "granted_at": "2026-10-06T14:00:00Z"},
        {"purpose": "transcription", "granted": True, "granted_at": "2026-10-06T14:00:00Z"},
    ],
    proof_mechanisms=[
        ProofMechanism(
            proof_type=ProofType.VERBAL_CONFIRMATION,
            timestamp="2026-10-06T14:00:05Z",
            proof_data={"dialog_reference": 0, "confirmation_text": "Yes, you can record this call"},
        )
    ],
    party_index=0,
    dialog_index=0,
)

v.check_lawful_basis_permission("transcription", 0)  # True
v.check_lawful_basis_permission("analysis", 0)       # False: no grant
```

## Synthetic data

Generated conversations have no real data subject. One workable pattern is `legitimate_interests` with `expiration: null`, an `external_system` proof that names the generator, and `validation: "synthetic"` on each party. This is a suggestion, not draft text. Never write a `consent` basis for a person who did not give it.

```json
{
  "lawful_basis": "legitimate_interests",
  "expiration": null,
  "purpose_grants": [
    { "purpose": "analysis", "granted": true, "granted_at": "2026-10-06T14:00:00Z" },
    { "purpose": "redistribution", "granted": true, "granted_at": "2026-10-06T14:00:00Z" }
  ],
  "proof_mechanisms": [
    {
      "proof_type": "external_system",
      "timestamp": "2026-10-06T14:00:00Z",
      "proof_data": { "system": "vcon-faker", "note": "Synthetic conversation; no real data subject" }
    }
  ]
}
```

## See also

* [Field reference](../vcons/field-reference.md) for core attachment fields.
* [Lifecycle](lifecycle.md) for recording consent and revocation events on a SCITT transparency service.
* [Privacy-First Conversation Management](../deep-dives/privacy-first-conversation-management.md) for the design rationale.
