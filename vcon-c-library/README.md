---
description: >-
  The C implementation of vCon (libvcon), a peer to the Python and JavaScript
  libraries for native, embedded, and systems code.
icon: plug
---

# vCon-C Library

`libvcon` is the C implementation of the vCon specification, a peer to the
[Python `vcon` library](../vcon-library/) and [`vcon-js`](../vcon-js-library/).
All three target [`draft-ietf-vcon-vcon-core`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/)
(syntax `"0.4.0"`) and produce byte-compatible vCons: build with one, verify or
consume with another.

> **Version:** `libvcon` **0.1.0** · Single header `vcon.h` · Requires OpenSSL 3;
> libcurl and ffmpeg optional · [GitHub: vcon-dev/vcon-c](https://github.com/vcon-dev/vcon-c).

## When to use vcon-c

* **vcon-c** for C/C++ services, embedded targets, telephony/media daemons, or
  any native code that reads or writes vCons where a libc + OpenSSL is the only
  safe dependency. It is the natural fit alongside the C components of the
  conserver ecosystem (for example the vfun transcription daemon).
* **Python `vcon`** for conserver links, data pipelines, and ML preprocessing.
* **vcon-js** for Node, edge functions, and browser code.

## Parity with the Python library

`libvcon` 0.1.0 is built for behavioral parity with `vcon-lib` 0.9.6, proven by
golden fixtures generated from the Python library and checked into the C repo.
Coverage:

* ✅ Core tree, JSON I/O, `property_handling` (default/strict/meta)
* ✅ Validation with the same error strings as Python `is_valid`
* ✅ uuid8 (domain + time), strict RFC 3339, base64url
* ✅ Builders: party, dialog (incl. transfer/incomplete + inline data),
  attachment, analysis, tags, extension/critical lists
* ✅ Content hashing (sha512/sha256 + legacy), OpenSSL-backed
* ✅ JWS RS256 sign/verify, cross-verified with Python both directions
* ✅ Extensions: lawful_basis and wtf transcription
* ✅ Optional HTTP (libcurl) and media metadata (ffprobe + header parsing)

### Deliberate divergences

`libvcon` matches Python's observable behavior except where Python is
demonstrably wrong:

* **`created_at` is preserved verbatim** on parse (Python reformats via dateutil,
  turning `Z` into `+00:00` non-idempotently).
* **The lawful-basis permission check works.** Python's
  `check_lawful_basis_permission` returns `False` for every input;
  `vcon_check_lawful_basis_permission` evaluates the grant per
  `draft-howe-vcon-lawful-basis`.
* **Strict RFC 3339** timestamps rather than lenient dateutil parsing.

## Quickstart

```c
#include <vcon.h>

vcon_t *v = vcon_new(NULL);              /* uuid8 + created_at filled in */

cJSON *alice = cJSON_CreateObject();
cJSON_AddStringToObject(alice, "name", "Alice");
vcon_add_party(v, alice);                /* takes ownership */

int parties[] = {0};
cJSON *d = vcon_dialog_new("text", "2026-01-15T10:30:00Z", parties, 1);
cJSON_AddStringToObject(d, "mediatype", "text/plain");
cJSON_AddStringToObject(d, "body", "Hello");
vcon_add_dialog(v, d);

char *json = vcon_to_json(v);            /* caller frees */
printf("%s\n", json);
free(json);
vcon_free(v);
```

Build and install:

```bash
cmake -S . -B build && cmake --build build
ctest --test-dir build --output-on-failure
sudo cmake --install build              # libvcon.a, vcon.h, libvcon.pc
```

Options: `-DVCON_WITH_CURL=OFF` drops HTTP; `-DVCON_WITH_MEDIA=ON` adds
image/PDF/ffprobe metadata.

## See also

* [Python vCon Library](../vcon-library/) — the reference implementation
* [vCon-JS Library](../vcon-js-library/) — the TypeScript peer
* [Extensions](../extensions/) — shapes for lawful_basis, wtf, and others
