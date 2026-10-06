---
description: >-
  The single rule list for adapter code review. Every other adapters page links
  here instead of repeating field rules.
---

# ✅ Spec Compliance Checklist

This page is the one place the adapter rules are written down. The [Extensions Cookbook](extensions-cookbook.md), the [Development Guide](vcon-adapter-development-guide.md) and the [LLM Guide](llm-guide-creating-vcon-adapters.md) link here rather than repeating them. Field-by-field definitions live in the [vCon field reference](../vcons/field-reference.md). The template's test suite in [`tests/`](https://github.com/vcon-dev/vcon-adapter-template/tree/main/tests) enforces most of this, and `pytest` runs it in any adapter scaffolded from the template.

## Spec target

Adapters here target IETF [`draft-ietf-vcon-vcon-core-04`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), with the `vcon` syntax parameter set to `"0.4.0"` and the [`vcon`](../vcon-library/README.md) Python library at 0.10.0 or later. Code or examples citing `0.2.0`, `0.3.0`, `-02` or `-03` are out of date.

With `encoding: "json"`, an attachment or analysis `body` is the JSON value itself (an object or array), not a `json.dumps()` string. Stringified JSON is still accepted, so readers must handle both, and the template's `json_body()` does. Writers emit the value form. Where an extension draft's own example shows a stringified body, write the value form anyway.

## The one rule

Build through the `vcon` library helpers (`add_party`, `add_dialog`, `add_attachment`, `add_analysis`, `add_tag`) and the template's `new_vcon()`, `add_lawful_basis()` and `finalize_vcon()`. Do not write to `vcon_dict[...]` by hand except for what those helpers cannot do: `subject` (the library has no setter) and the `extensions` list. `new_vcon()` also drops the empty `group` and `redacted` that older library releases leave behind, and `finalize_vcon()` strips empty `meta` and `metadata` placeholders. Current library releases already omit `group` and `redacted`, so on a pinned `vcon>=0.10.0` those pops are no-ops.

## Top-level vCon

* [ ] `vcon` is exactly `"0.4.0"`
* [ ] `uuid` is a UUID string, preferably version 8 built from a domain you control (core-04 §4.1.2)
* [ ] Every timestamp is ISO 8601 with a timezone (`Z` or an offset)
* [ ] No empty `group`, `redacted`, `meta` or `metadata` anywhere
* [ ] Every extension used is listed in `extensions[]`, spelled as the draft spells it: `lawful_basis`, `sip-signaling`, `wtf_transcription`, `agent_session`. The lifecycle draft defines no token, so lifecycle events need no `extensions` entry
* [ ] An extension is listed in `critical` only when a consumer must refuse the vCon without it

## Analysis

* [ ] `vendor` is present (the library raises without it)
* [ ] The schema pointer is `schema`, never `schema_version`
* [ ] A JSON body is the parsed value with `encoding: "json"`
* [ ] Transcripts live in `analysis[]`, never `attachments[]`, with `type: "wtf_transcription"`, `vendor`, `product` and the WTF draft URL in `schema`

## Attachments

* [ ] The field is `purpose`, never `type`, including for `lawful_basis`. Readers should still accept legacy `type` on old vCons
* [ ] `start`, `party` and `dialog` are all present. Use `0` and `0` when the attachment is not tied to a party or dialog
* [ ] A body carries `mediatype` and `encoding`
* [ ] Inline binary is `base64url`, never plain base64 or hex
* [ ] Tags go through `add_tag(name, value)`

## External media

* [ ] A dialog that points at media has both `url` and `content_hash`
* [ ] `content_hash` is `sha512-<base64url digest>`, not hex, not padded base64, not SHA-256
* [ ] `mediatype` is set

The template provides `sha512_b64url(data)` and `external_media_url(url=, content=, mediatype=)` in [`vcon_builder.py`](https://github.com/vcon-dev/vcon-adapter-template/blob/main/src/__ADAPTER_PACKAGE__/vcon_builder.py).

## Legacy names

Never write the left column. A receiver may reject the vCon or misroute it.

| Never write | Write | Note |
| --- | --- | --- |
| `appended` | `amended` | legacy vcon-mcp column |
| `must_support`, `must_understand` | `critical` | legacy vcon-mcp column |
| `schema_version` | `schema` | older draft field |
| `type` on an attachment | `purpose` | applies to `lawful_basis` too |
| `mimetype` | `mediatype` | the spec field is `mediatype` |
| `did` on a party | none | removed in 0.4.0 |

## Lawful basis

* [ ] Every vCon an adapter emits carries a `purpose: "lawful_basis"` attachment, built by `add_lawful_basis()` from `LawfulBasisConfig`, not by hand
* [ ] The adapter never defaults a basis in code. When `LAWFUL_BASIS` and the YAML block are unset, the helper logs one warning per process and adds nothing. Set it to what your deployment actually relies on, or accept the warning
* [ ] `lawful_basis` is in `extensions[]` (the helper adds it)
* [ ] The body follows [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/): `lawful_basis`, `expiration` (an ISO 8601 timestamp or `null`), and `purpose_grants[]` with `purpose`, `granted` and `granted_at`. `proof_mechanisms[]` entries carry `proof_type`, `timestamp` and `proof_data`

See the [Extensions Cookbook](extensions-cookbook.md) for the helper call and the [Lawful Basis page](../extensions/lawful-basis.md) for the model.

## Synthetic test data

* [ ] Each synthetic party carries `validation: "synthetic"`
* [ ] If you choose to attach a lawful basis to synthetic data, document the origin with an `external_system` proof mechanism. Do not forge a consent record and do not default a basis for the data

## The template's tests

The template's `tests/` directory has three modules:

* `test_vcon_builder.py` covers the builder helpers, `LawfulBasisConfig`, `add_lawful_basis()`, `finalize_vcon()` and `json_body()`, plus a check that serialized output never contains `appended` or `must_support`
* `test_spec_compliance.py` validates a sample vCon against the vendored official JSON schema and checks the rules the schema does not, such as no `mimetype`, a `purpose`, `start`, `party` and `dialog` on every attachment, and no double-encoded JSON bodies. Its `assert_spec_compliant()` is written to be copied into another repo's tests
* `test_webhook_delivery.py` covers the HMAC signature format, dead-letter writes, and `ConserverDelivery` headers, ingress lists and retries

Outside the template, copy `test_spec_compliance.py` and the vendored schema.

## When you find drift

If a vCon in your archive, a partner payload or a fixture breaks this list, do not round-trip it through `Vcon.from_dict(...)` and call it fixed. Open an issue against the producing adapter naming the field, then either repair it with a conserver link or reject it at ingress.
