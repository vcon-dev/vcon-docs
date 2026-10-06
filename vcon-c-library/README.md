---
description: >-
  What libvcon 0.1.0 covers, which Python library version it matches, and how
  to build and use it from C.
icon: plug
---

# vCon-C Library

`libvcon` is the C implementation of the vCon specification, a peer to the [Python `vcon` library](../vcon-library/) and [`vcon-js`](../vcon-js-library/). Version 0.1.0 reads and writes the vCon JSON structure from C and C++ with libc, OpenSSL 3, and a vendored cJSON.

> **Version:** `libvcon` **0.1.0** · Single header [`vcon.h`](https://github.com/vcon-dev/vcon-c/blob/main/include/vcon.h) · Requires OpenSSL 3. libcurl and ffprobe are optional · Source: [vcon-dev/vcon-c](https://github.com/vcon-dev/vcon-c).

## Parity baseline

`libvcon` 0.1.0 matches Python `vcon-lib` **0.9.6** and `draft-ietf-vcon-vcon-core-02` (syntax `"0.4.0"`). Parity is behavioral, proven by golden fixtures generated from vcon-lib 0.9.6.

`libvcon` has not yet tracked the 0.10 changes in `vcon-lib`, which targets core-04. These differ from what the C library writes today:

* Empty `meta` and `metadata` placeholders. `vcon_add_dialog` adds `meta:{}` and `metadata:{}` as 0.9.6 did. Newer Python drops empty `meta`.
* The tags attachment body shape. `vcon_add_tag` writes the body as a list of `"name:value"` strings. [vcon-js](../vcon-js-library/) writes an object.
* base64url padding rules.

The three libraries read each other's output, but do not rely on identical serialized bytes. Field rules are in the [field reference](../vcons/field-reference.md).

## When to use vcon-c

* **vcon-c** for C and C++ services, embedded targets, telephony and media daemons, or native code where libc and OpenSSL are the only safe dependencies.
* **Python `vcon`** for conserver links, data pipelines, and ML preprocessing.
* **vcon-js** for Node, edge functions, and browser code.

## What is covered

* Core tree, JSON I/O, and `property_handling` (default, strict, meta)
* Validation with the same error strings as Python `is_valid`
* uuid8 (domain and time), strict RFC 3339, base64url
* Builders: party, dialog (including transfer, incomplete, inline data), attachment, analysis, tags, extension and critical lists
* Content hashing (sha512, sha256, legacy unprefixed), through OpenSSL
* JWS RS256 sign and verify, cross-verified with Python in both directions
* Extensions: lawful basis and WTF transcription builders and finders
* Optional HTTP through libcurl and media metadata through ffprobe and header parsing

The [API reference](api-reference.md) has the function list, return codes, ownership rules, and the Python-to-C map.

### Deliberate divergences from vcon-lib 0.9.6

* **`created_at` is preserved verbatim** on parse. Python reformats it through dateutil, turning `Z` into `+00:00` non-idempotently.
* **The lawful basis permission check evaluates the grant.** In vcon-lib 0.9.x, `check_lawful_basis_permission` returns `False` for every input. `vcon_check_lawful_basis_permission` follows `draft-howe-vcon-lawful-basis`: the grant is present, `granted` is true, and the basis is unexpired. Python 0.10.0 returns `True` for a granted grant.
* **Strict RFC 3339** timestamps rather than lenient dateutil parsing. A non-array `parties`, `dialog`, or `analysis` is a parse error.

## Build

```bash
cmake -S . -B build && cmake --build build
ctest --test-dir build --output-on-failure
sudo cmake --install build              # libvcon.a, vcon.h, libvcon.pc
```

Options: `-DVCON_WITH_CURL=OFF` drops HTTP (`vcon_load_url`, `vcon_post_url`). `-DVCON_WITH_MEDIA=ON` adds image, PDF, and ffprobe metadata. The default build compiles the worked example to `./build/call_recording`.

## Quickstart

This builds a vCon with one party, one text dialog, and a lawful basis attachment, checks it, and prints it. It was built and run against the 0.1.0 source.

```c
#include <stdio.h>
#include <stdlib.h>
#include <vcon.h>

int main(void) {
    vcon_t *v = vcon_new(NULL);                    /* uuid8 and created_at filled in */

    cJSON *alice = cJSON_CreateObject();
    cJSON_AddStringToObject(alice, "name", "Alice");
    int alice_idx = vcon_add_party(v, alice);      /* takes ownership, returns index */

    int parties[] = { alice_idx };
    cJSON *d = vcon_dialog_new("text", "2026-01-15T10:30:00Z", parties, 1);
    cJSON_AddStringToObject(d, "mediatype", "text/plain");
    cJSON_AddStringToObject(d, "body", "Hello");
    cJSON_AddStringToObject(d, "encoding", "none");
    vcon_add_dialog(v, d);

    /* Illustrative values only, not a real consent record. */
    cJSON *grants = cJSON_Parse(
        "[{\"purpose\":\"recording\",\"granted\":true,"
        "\"granted_at\":\"2026-01-15T10:30:00Z\"}]");
    int rc = vcon_add_lawful_basis_attachment(v, "consent", "2027-01-15T10:30:00Z",
                                              grants, NULL, alice_idx, 0);
    if (rc != VCON_OK) { fprintf(stderr, "lawful basis failed: %d\n", rc); return 1; }

    vcon_errors_t errs = {0};
    if (!vcon_is_valid(v, &errs))
        for (size_t i = 0; i < errs.count; i++) fprintf(stderr, "invalid: %s\n", errs.items[i]);
    vcon_errors_free(&errs);

    printf("recording permitted: %d\n",
           vcon_check_lawful_basis_permission(v, "recording", -1));

    char *json = vcon_to_json(v);                  /* caller frees */
    printf("%s\n", json);
    free(json);
    vcon_free(v);
    return 0;
}
```

The lawful basis attachment carries `purpose: "lawful_basis"`, `encoding: "json"`, and a body that is a JSON string. The builder adds `lawful_basis` to `extensions`. It does not set `start`, which the core schema requires on every attachment. To include one, build the attachment object yourself and add it with `vcon_add_attachment`.

For a full walkthrough with a recording dialog, content hash, transcript and summary analysis, tags, JWS signing, and serialization, read [`examples/call_recording.c`](https://github.com/vcon-dev/vcon-c/blob/main/examples/call_recording.c).

## See also

* [API Reference](api-reference.md)
* [Python vCon Library](../vcon-library/)
* [vCon-JS Library](../vcon-js-library/)
* [Extensions](../extensions/)
