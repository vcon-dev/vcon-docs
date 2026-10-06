---
description: >-
  A short digest to paste into a model's context so it generates spec-correct
  vCon adapter code. The rules live on the Spec Compliance Checklist.
---

# 🤖 LLM Guide: Creating vCon Adapters

Paste this page into a model's context to generate adapter code. It holds the ground truth and a pointer list. The full rules are on the [Spec Compliance Checklist](spec-compliance-checklist.md), which is the single source.

## Ground truth

* Spec: `draft-ietf-vcon-vcon-core-04`. The `vcon` syntax parameter is `"0.4.0"`.
* Library: `vcon` (PyPI) 0.10.0 or later. Start from [vcon-adapter-template](https://github.com/vcon-dev/vcon-adapter-template) and use its `src/<package>/vcon_builder.py` helpers: `new_vcon()`, `add_lawful_basis()`, `external_media_url()`, `sha512_b64url()`, `finalize_vcon()`, `json_body()`.
* `add_party()` and `add_dialog()` take `Party` and `Dialog` objects (`from vcon.party import Party`, `from vcon.dialog import Dialog`). `add_attachment()`, `add_analysis()` and `add_tag()` take keyword arguments.
* With `encoding="json"`, `body` is the JSON value itself, never a `json.dumps()` string. Readers accept both; use `json_body()`.
* Attachments use `purpose`, never `type`, and carry `start`, `party`, `dialog`. A body carries `mediatype` and `encoding`.
* Analysis uses `schema`, never `schema_version`, and requires `vendor`. Transcripts go in `analysis[]` with `type: "wtf_transcription"`.
* External media needs both `url` and `content_hash`, formatted `sha512-<base64url digest>`.
* Field names: `critical` (not `must_support`), `amended` (not `appended`), `mediatype` (not `mimetype`). Do not emit `did` on parties.
* Timestamps are ISO 8601 with a timezone. List every extension used in `extensions[]`.
* Build through helpers. Hand-set only `subject` and `extensions`.

## Lawful basis

Every vCon the adapter emits gets a `purpose: "lawful_basis"` attachment built by the template helper:

```python
from datetime import datetime, timezone
from foo_adapter.vcon_builder import add_lawful_basis

add_lawful_basis(v, config.lawful_basis, granted_at=datetime.now(timezone.utc).isoformat())
```

`config.lawful_basis` is a `LawfulBasisConfig`, loaded from `LAWFUL_BASIS`, `LAWFUL_BASIS_PURPOSE`, `LAWFUL_BASIS_JURISDICTION`, `LAWFUL_BASIS_EXPIRATION` and `LAWFUL_BASIS_PROOF_MECHANISM`, or from the `vcon.lawful_basis` YAML block. Never default a basis in generated code. Do not write a literal `"consent"` or any other value as a fallback. If the operator has not configured one, the helper logs a warning and adds nothing, and that is the correct behavior. Do not build the attachment by hand and do not use the library's `add_lawful_basis_attachment()`.

## Delivery

Use the template's `WebhookDelivery` (HMAC `X-Hub-Signature-256`, `Idempotency-Key` equal to the vCon `uuid`, backoff, dead-letter queue) or `ConserverDelivery` (`POST {CONSERVER_URL}/vcon` with `x-conserver-api-token` and `ingress_lists`). Both call `finalize_vcon()`. JWS content signing is not built in. See [Operational Patterns](operational-patterns.md).

## Where to look

* Rules and tests: [Spec Compliance Checklist](spec-compliance-checklist.md)
* Field definitions: [vCon field reference](../vcons/field-reference.md)
* Scaffold steps and env vars: [Quick Start From Template](quick-start-from-template.md)
* Worked extension recipes (WTF, lawful basis, SIP, agent session): [Extensions Cookbook](extensions-cookbook.md)
* Listener shapes and where data lives in a vCon: [Development Guide](vcon-adapter-development-guide.md)
* Authority for any disagreement: the drafts on [datatracker](https://datatracker.ietf.org/)
