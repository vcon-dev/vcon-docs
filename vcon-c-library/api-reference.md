---
description: Every libvcon 0.1.0 function group, the return codes, the ownership rules, the Python-to-C map, and what is not ported.
---

# 🔌 API Reference

Reference for **`libvcon` 0.1.0**. The authoritative declarations are in [`include/vcon.h`](https://github.com/vcon-dev/vcon-c/blob/main/include/vcon.h). For a first program see the [overview](README.md). Baseline: Python `vcon-lib` 0.9.6 and `draft-ietf-vcon-vcon-core-02`, syntax `"0.4.0"` (`VCON_SYNTAX_VERSION`). The C library has not yet tracked the 0.10 changes; see [Parity baseline](README.md#parity-baseline). Field rules: [field reference](../vcons/field-reference.md).

## Build options

| Option | Default | Effect |
| --- | --- | --- |
| `-DVCON_WITH_CURL=ON/OFF` | ON | Builds `vcon_load_url` and `vcon_post_url` through libcurl. OFF removes them and the libcurl dependency. |
| `-DVCON_WITH_MEDIA=ON/OFF` | OFF | Builds `vcon_image_dimensions`, `vcon_pdf_page_count`, and `vcon_probe_media`. |

OpenSSL 3 is always required (hashing and JWS). Link with `-lvcon -lcjson`; the installed `libvcon.pc` carries both.

## Return codes

Functions that return `int` use these unless noted. Zero is success and negative is failure.

| Code | Value | Meaning |
| --- | --- | --- |
| `VCON_OK` | 0 | Success |
| `VCON_ERR` | -1 | Generic failure |
| `VCON_ERR_PARSE` | -2 | JSON did not parse |
| `VCON_ERR_ARG` | -3 | Bad argument |
| `VCON_ERR_IO` | -4 | File read or write failed, or ffprobe unavailable |

Exceptions: `vcon_add_*` return the new index (0 or more) or a negative code. `vcon_is_valid`, `vcon_validate_*`, `vcon_validate_extensions`, `vcon_verify_content_hash`, and `vcon_check_lawful_basis_permission` return 1 or 0. `vcon_verify` returns 1, 0, or `VCON_ERR_ARG`. `vcon_post_url` returns the HTTP status (100 or more) or a negative code. Constructors and loaders return `NULL` on failure.

## Ownership rules

* Functions that return `vcon_t *` or `char *` transfer ownership. Free with `vcon_free` and `free`.
* `cJSON *` pointers from accessors and finders (`vcon_parties_get`, `vcon_root`, `vcon_find_attachment_by_purpose`, and so on) are borrowed from the vCon's tree. Do not free them. They are invalid after `vcon_free` or after you mutate the array they live in.
* Every `vcon_add_*` that takes a `cJSON *` takes ownership of it, including on the lawful basis and WTF builders (`purpose_grants`, `proof_mechanisms`, `transcript`, `segments`, `metadata`). Do not free it or add it to a second tree.
* `vcon_set_redacted` and `vcon_set_amended` take ownership of the object. Setting one clears the other.
* `vcon_get_tag` returns an owned string. Free it.
* `vcon_errors_t` owns its strings. Call `vcon_errors_free` after every validation call.
* `vcon_generate_key_pair` returns two owned PEM strings.
* `vcon_dialog_new` returns an object you own until you pass it to `vcon_add_dialog`.

## Lifecycle and I/O

| Function | Description |
| --- | --- |
| `vcon_new(created_at_or_NULL)` | New vCon with a uuid8 and empty arrays. |
| `vcon_from_json(text, mode)` | Parse JSON. Preserves a valid RFC 3339 `created_at` verbatim. `NULL` on failure. |
| `vcon_load(source, mode)` | A string starting with `{` is JSON, otherwise a file path. |
| `vcon_load_file(path, mode)` | Load from a file. |
| `vcon_load_url(url, mode)` | GET and parse. Needs `VCON_WITH_CURL`. |
| `vcon_to_json(v)` | Owned JSON in insertion order. |
| `vcon_to_json_canonical(v)` | Owned JSON with keys sorted recursively. |
| `vcon_save_file(v, path)` | Write to a file. |
| `vcon_post_url(v, url, headers, n)` | POST as `application/json` with extra `"Name: value"` headers. Needs `VCON_WITH_CURL`. |
| `vcon_root(v)` | Borrowed root `cJSON`. |
| `vcon_free(v)` | Free the vCon. |

`vcon_prop_mode_t` is `VCON_PROP_DEFAULT` (keep non-standard properties), `VCON_PROP_STRICT` (drop them), or `VCON_PROP_META` (move them into `meta`).

## Properties and arrays

* Getters return borrowed strings, `NULL` when absent: `vcon_get_uuid`, `vcon_get_vcon_version`, `vcon_get_subject`, `vcon_get_created_at`, `vcon_get_updated_at`.
* Setters: `vcon_set_subject`, `vcon_set_updated_at(v, rfc3339_or_NULL)`, `vcon_set_redacted`, `vcon_set_amended`.
* Arrays: `vcon_parties_count`, `vcon_dialog_count`, `vcon_attachments_count`, `vcon_analysis_count`, and `vcon_parties_get`, `vcon_dialog_get`, `vcon_attachments_get`, `vcon_analysis_get` (borrowed).

## Builders

| Function | Description |
| --- | --- |
| `vcon_add_party(v, party)` | Append a party. Returns its index. |
| `vcon_find_party_index(v, by, val)` | Index of the first party whose string field `by` equals `val`, or -1. |
| `vcon_dialog_new(type, start, parties, n)` | Build a dialog object. Add optional fields, then call `vcon_add_dialog`. |
| `vcon_add_dialog(v, dialog)` | Append a dialog. Adds `meta:{}` and `metadata:{}` if missing, as vcon-lib 0.9.6 did. |
| `vcon_add_transfer_dialog(v, start, transfer_data, parties, n)` | Append a transfer dialog. |
| `vcon_add_incomplete_dialog(v, start, disposition, parties, n)` | Append an incomplete dialog. |
| `vcon_dialog_add_inline_data(dialog, body, filename, mediatype)` | Set `body`, `mediatype`, `filename`, `encoding: "base64url"`, and a sha512 `content_hash` over the given body string. The body is stored as you pass it. Pass text that is already base64url if you declare that encoding. |
| `vcon_add_attachment(v, attachment)` | Append an attachment object you built. Set `purpose`, `start`, `party`, `dialog`. |
| `vcon_find_attachment_by_purpose(v, purpose)` | Borrowed first match, or `NULL`. |
| `vcon_add_analysis(v, analysis)` | Append an analysis object. Set `type`, `dialog`, `vendor`. |
| `vcon_find_analysis_by_type(v, type)` | Borrowed first match, or `NULL`. |
| `vcon_add_tag(v, name, value)` | Find or create the single `purpose: "tags"` attachment and append `"name:value"` to its list body. |
| `vcon_get_tag(v, name)` | Owned value string, or `NULL`. |
| `vcon_add_extension`, `vcon_remove_extension` | Edit `extensions[]`. |
| `vcon_add_critical`, `vcon_remove_critical` | Edit `critical[]`. |

## Content hash

| Function | Description |
| --- | --- |
| `vcon_compute_content_hash(data, len, alg_or_NULL)` | Owned `sha512-<base64url, no padding>` (default) or `sha256-...`. `NULL` for an unknown algorithm. |
| `vcon_parse_content_hash_algorithm(hash)` | `"sha512"`, `"sha256"`, or `NULL` for a legacy unprefixed hash. |
| `vcon_verify_content_hash(data, len, stored)` | 1 if the data matches. |

## Extensions

| Function | Description |
| --- | --- |
| `vcon_add_lawful_basis_attachment(v, basis, expiration, grants, proofs_or_NULL, party, dialog)` | Add a `purpose: "lawful_basis"` attachment with `encoding: "json"` and a JSON string body, and add `lawful_basis` to `extensions`. Pass -1 for `party` or `dialog` to omit. Does not set `start`. |
| `vcon_find_lawful_basis_attachments(v, party_or_-1, out, cap)` | Fill borrowed pointers. Returns the total match count, which may exceed `cap`. |
| `vcon_check_lawful_basis_permission(v, purpose, party_or_-1)` | 1 if a lawful basis attachment grants `purpose` (`granted: true`) and is unexpired. Follows `draft-howe-vcon-lawful-basis`. vcon-lib 0.9.x returns `False` for every input; Python 0.10.0 returns `True` for a granted grant. |
| `vcon_add_wtf_transcription_attachment(v, transcript, segments, metadata, party, dialog)` | Add a `purpose: "wtf_transcription"` attachment. Adds `wtf_transcription` to `extensions`. |
| `vcon_add_wtf_transcription_analysis(v, transcript, segments, metadata, dialog)` | Add an analysis entry with `type: "transcription"`, the WTF draft URL as `schema`, and `vendor` and `product` from `metadata.provider` and `metadata.model`. |
| `vcon_find_wtf_attachments(v, party_or_-1, out, cap)` | Fill borrowed pointers for `wtf_transcription` attachments. |
| `vcon_validate_extensions(v, errs)` | 1 if every name in `extensions[]` is known (`lawful_basis`, `wtf_transcription`) and its attachments parse. |

See [Lawful Basis](../extensions/lawful-basis.md) and [WTF Transcription](../extensions/wtf-transcription.md) for the shapes. The lawful basis builder writes proof entries as you pass them. The current draft names the proof fields `proof_type`, `timestamp`, and `proof_data`, so pass those.

## Validation

`vcon_is_valid(v, &errs)`, `vcon_validate_json(text, &errs)`, and `vcon_validate_file(path, &errs)` return 1 when valid and fill `errs` with the same messages as vcon-lib's `is_valid` otherwise. Free with `vcon_errors_free`.

## Signing

`vcon_sign(v, priv_pem)` stores a JWS (RS256) in the vCon in place. `vcon_verify(v, pub_pem)` checks it. `vcon_generate_key_pair(&priv, &pub)` makes a 2048-bit RSA pair (PKCS#8 private, SPKI public). Signing and verification are cross-checked with the Python library.

## Media (`VCON_WITH_MEDIA`)

`vcon_image_dimensions` (PNG, JPEG, GIF header parse), `vcon_pdf_page_count` (page-object scan), and `vcon_probe_media` (ffprobe for duration, width, height).

## uuid8

`vcon_uuid8_time(custom_low64)` and `vcon_uuid8_domain_name(domain)` return owned 36-character strings.

## Python to C map

| Python (`vcon-lib`) | C (`libvcon`) |
| --- | --- |
| `Vcon.build_new(created_at)` | `vcon_new` |
| `build_from_json`, `load`, `load_from_file`, `load_from_url` | `vcon_from_json`, `vcon_load`, `vcon_load_file`, `vcon_load_url` |
| `to_json`, `dumps`, `to_dict`, `save_to_file`, `post_to_url` | `vcon_to_json` (and `vcon_to_json_canonical`), `vcon_root`, `vcon_save_file`, `vcon_post_url` |
| properties `uuid`, `vcon`, `subject`, `created_at`, `updated_at`, `redacted`, `amended` | `vcon_get_*` and `vcon_set_*` (`redacted` and `amended` mutually exclusive) |
| `parties`, `dialog`, `attachments`, `analysis` | `vcon_<array>_count` and `vcon_<array>_get` |
| `add_party`, `find_party_index` | `vcon_add_party`, `vcon_find_party_index` |
| `add_dialog`, `add_transfer_dialog`, `add_incomplete_dialog` | same names with `vcon_` prefix. `vcon_dialog_new` builds the base object |
| `Dialog.add_inline_data` | `vcon_dialog_add_inline_data` |
| `add_attachment`, `find_attachment_by_purpose` | `vcon_add_attachment`, `vcon_find_attachment_by_purpose` |
| `add_analysis`, `find_analysis_by_type` | `vcon_add_analysis`, `vcon_find_analysis_by_type` |
| `tags`, `get_tag`, `add_tag` | `vcon_get_tag`, `vcon_add_tag` |
| `get/add/remove_extensions`, `get/add/remove_critical` | `vcon_add_extension`, `vcon_remove_extension`, `vcon_add_critical`, `vcon_remove_critical` |
| `compute_content_hash`, `parse_content_hash_algorithm`, `verify_content_hash` | `vcon_compute_content_hash`, `vcon_parse_content_hash_algorithm`, `vcon_verify_content_hash` |
| `is_valid`, `validate_json`, `validate_file` | `vcon_is_valid`, `vcon_validate_json`, `vcon_validate_file` |
| `sign`, `verify`, `generate_key_pair` | `vcon_sign`, `vcon_verify`, `vcon_generate_key_pair` |
| `uuid8_time`, `uuid8_domain_name` | `vcon_uuid8_time`, `vcon_uuid8_domain_name` |
| `add_lawful_basis_attachment`, `find_lawful_basis_attachments`, `check_lawful_basis_permission` | same names with `vcon_` prefix |
| `add_wtf_transcription_attachment`, `add_wtf_transcription_analysis`, `find_wtf_attachments` | same names with `vcon_` prefix |
| `validate_extensions` | `vcon_validate_extensions` |

## Not ported

* pydash dotted-path lookup
* `datetime` object overloads (C takes RFC 3339 strings)
* Python `logger` output
* The extension registry class hierarchy, replaced by direct builders and a known-name check
* `add_external_data` fetch-and-hash, and the heavy ffmpeg operations (transcode, thumbnail, streaming). The curl and ffprobe building blocks are present; these are added when needed.

## Known limitation

cJSON stores numbers as doubles, so an integer above 2^53 in `meta` loses precision.

## See also

* [Overview and quickstart](README.md)
* [`vcon.h`](https://github.com/vcon-dev/vcon-c/blob/main/include/vcon.h)
* [Python library API](../vcon-library/library-api-reference.md)
