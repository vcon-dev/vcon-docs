---
icon: clipboard-check
description: The must/never list for every adapter PR. Mirrors the smoke tests in vcon-adapter-template.
---

# ✅ Spec Compliance Checklist

This is the gate every adapter PR (and every hand-written vCon construction) should pass. It mirrors the 14 smoke tests in [`vcon-adapter-template/tests/test_vcon_builder.py`](https://github.com/vcon-dev/vcon-adapter-template/blob/main/tests/test_vcon_builder.py) and the [`CONTRIBUTING.md`](https://github.com/vcon-dev/vcon-adapter-template/blob/main/CONTRIBUTING.md) in the same repo. If you scaffolded from the template, `pytest` runs all of this for you.

**Spec target:** IETF [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), syntax `"0.4.0"`.

## The one-line rule

> **Always use the [`vcon`](../vcon-library/README.md) library helpers (`add_party`, `add_dialog`, `add_attachment`, `add_analysis`, `add_tag`). Never write directly to `vcon_dict[...]` except for the four documented quirks below.**

Recent versions of the `vcon` Python library (≥0.9.4) emit spec-correct output for every helper. Hand-rolling a dict bypasses that and is the single most common source of compliance drift in real adapters.

## Top-level vCon

- [ ] `vcon` syntax parameter is exactly the string `"0.4.0"` — not `0.0.1`, `0.0.2`, `0.2.0`, `0.3.0`, or any number
- [ ] `uuid` is a v4 UUID string
- [ ] All timestamps are ISO-8601 with a timezone (`Z` or explicit offset)
- [ ] No empty `group: []` or empty `redacted: {}` left over from `Vcon.build_new()` — drop them; the spec reserves these fields for actual use
- [ ] `subject` is written via `v.vcon_dict["subject"]` (the lib has no setter)

The template's `new_vcon()` helper handles all five of these — call it instead of `Vcon.build_new()` directly.

## Analysis objects

- [ ] Constructed via `Vcon.add_analysis(type, dialog, vendor, body, encoding, schema, product, ...)`
- [ ] Field name is **`schema`** — never `schema_version`
- [ ] `vendor` is REQUIRED on every analysis (the lib enforces this as a kwarg)
- [ ] `body` is always a string. For JSON bodies, pair it with `encoding="json"`
- [ ] Transcripts live in `analysis[]`, not `attachments[]`
- [ ] Transcript analysis has `schema=<WTF draft URL>`, `encoding="json"`, `vendor="<provider>"`, `product="<model>"`

See the [Extensions Cookbook](extensions-cookbook.md) for full transcript examples.

## Attachment objects

- [ ] Constructed via `Vcon.add_attachment(purpose, body, encoding, party, dialog, ...)`
- [ ] Field name is **`purpose`** — never `type` (in core; `lawful_basis` is a documented exception, see below)
- [ ] `party` AND `dialog` indices are passed. Use `0, 0` for vCon-level attachments not tied to a specific party or dialog
- [ ] JSON-bodied attachments use `encoding="json"` with a JSON-string `body`

## Tags

- [ ] Use `Vcon.add_tag(name, value)` directly. Library ≥0.9.3 writes `party`/`dialog` on the tags attachment correctly — no backfill needed.

## External media

- [ ] Both `url` AND `content_hash` are present on the dialog
- [ ] `content_hash` is formatted as `sha512-<base64url-of-digest>` — not hex, not base64 (with `+/=`), not SHA-256
- [ ] `mediatype` is set (e.g. `audio/wav`, `video/mp4`, `text/plain`)

The template provides `sha512_b64url(data)` and `external_media_url(url, content, mediatype)` helpers in [`vcon_builder.py`](https://github.com/vcon-dev/vcon-adapter-template/blob/main/src/__ADAPTER_PACKAGE__/vcon_builder.py).

## Legacy field-name traps

These names appear in older vcon-mcp code and older draft revisions. **Never** emit them in a new vCon — the receiver may reject the vCon or, worse, silently misroute it.

| ❌ Never write | ✅ Always write | Why |
|---|---|---|
| `appended` | `amended` | Legacy vcon-mcp column name |
| `must_support` | `critical` | Legacy vcon-mcp column name |
| `schema_version` | `schema` | Older draft field name; current spec is `schema` |
| `type` (on attachments) | `purpose` | Core spec uses `purpose`; only `lawful_basis` extension uses `type` |
| `did` (on parties) | (removed) | The `did` field was removed in `0.4.0` |

The template's smoke test `test_no_legacy_field_names_in_serialized_vcon` greps the serialized vCon for `appended` and `must_support` and fails the build if either appears.

## Extensions

- [ ] Every extension used is listed in top-level `extensions[]`
- [ ] Extension names match the spec exactly (e.g. `"sip-signaling"`, `"lawful_basis"`, `"wtf"` or `"wtf_transcription"`, `"agent_session"`, `"lifecycle"`)

## Lawful basis (if recording consent is tracked)

The `lawful_basis` attachment is the **single documented exception** to the "use `purpose` on attachments" rule:

- [ ] Attachment uses `type: "lawful_basis"` (not `purpose`)
- [ ] `"lawful_basis"` is added to top-level `extensions[]`
- [ ] For synthetic test data: `lawful_basis: "legitimate_interests"` + `proof_mechanism` of type `external_system`

See the [Extensions Cookbook](extensions-cookbook.md) and the [Lawful Basis page](../extensions/lawful-basis.md) for full examples.

## Synthetic test data

If you're generating synthetic vCons for testing or training:

- [ ] Each synthetic party is marked with `validation: "synthetic"`
- [ ] A `purpose: "synthetic_data_consent"` attachment is present, OR a `lawful_basis` attachment documents the synthetic origin via an `external_system` proof mechanism

## The 14 smoke tests, by name

These are the canonical compliance gates. From [`test_vcon_builder.py`](https://github.com/vcon-dev/vcon-adapter-template/blob/main/tests/test_vcon_builder.py):

1. `test_syntax_is_0_4_0`
2. `test_build_new_strips_group_and_redacted`
3. `test_subject_is_written_via_vcon_dict`
4. `test_extensions_listed_at_top_level`
5. `test_lib_add_attachment_uses_purpose_with_party_and_dialog`
6. `test_lib_add_analysis_uses_schema_not_schema_version`
7. `test_lib_add_analysis_requires_vendor`
8. `test_lib_add_tag_writes_party_and_dialog`
9. `test_content_hash_format`
10. `test_external_media_dialog_has_url_and_content_hash`
11. `test_no_legacy_field_names_in_serialized_vcon[appended]`
12. `test_no_legacy_field_names_in_serialized_vcon[must_support]`
13. (Plus delivery-layer tests for HMAC signature format and DLQ behavior)

Run them with `pytest` from any adapter scaffolded from the template. If you're not using the template, copy the test file — it's spec-version-pinned and short.

## When you discover compliance drift

If you find a vCon in the wild — your archive, a partner's payload, a test fixture — that violates this list, do *not* round-trip it through `Vcon.from_dict(...)` and pretend it's fine. Open an issue against the producing adapter, file the offending field, and either:

- Add a migration link in your conserver to repair the shape, OR
- Reject the vCon at ingress

The spec is what the spec says. Tolerating drift is how the ecosystem fragments.
