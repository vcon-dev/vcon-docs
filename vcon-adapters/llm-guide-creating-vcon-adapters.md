---
description: >-
  Drop-into-context guide for LLMs generating vCon adapter code. Spec target:
  draft-ietf-vcon-vcon-core-02, syntax 0.4.0. Pairs with the
  vcon-adapter-template repo.
---

# 🤖 LLM Guide: Creating vCon Adapters

This page is designed to be pasted into a model's context window when you want it to generate adapter code. It encodes the spec target, the canonical scaffold, the library API, and the legacy-field-name traps in one place.

## Ground truth

**Spec:** IETF [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/). The `vcon` syntax parameter is the string `"0.4.0"`. Any older value (`0.0.1`, `0.0.2`, `0.2.0`, `0.3.0`) is wrong.

**Canonical scaffold:** [vcon-dev/vcon-adapter-template](https://github.com/vcon-dev/vcon-adapter-template). New adapters SHOULD start from this template — it ships a `vcon_builder.py` wrapper, HMAC webhook delivery, retries/DLQ, health + Prometheus endpoints, and 14 spec-compliance smoke tests.

**Library:** [`vcon`](https://pypi.org/project/vcon/) ≥ 0.9.4. The lib's helpers are spec-correct out of the box — use them instead of writing to `vcon_dict[...]` directly.

## The one rule

> Always use the `vcon` library helpers (`add_party`, `add_dialog`, `add_attachment`, `add_analysis`, `add_tag`). Never write directly to `vcon_dict[...]` except for the four documented quirks below.

## Imports

```python
import hashlib
import json
import logging
from abc import ABC, abstractmethod
from base64 import urlsafe_b64encode
from datetime import datetime, timezone
from typing import Any

from vcon import Vcon
```

`Party` and `Dialog` model classes are no longer commonly used — pass values as kwargs to `add_party()` / `add_dialog()` instead.

## Create a vCon (handle the four quirks)

`Vcon.build_new()` has four spec-incorrect behaviors that every adapter must paper over. The template's `new_vcon()` helper does it for you:

```python
def new_vcon(
    *,
    subject: str | None = None,
    extensions: list[str] | None = None,
) -> Vcon:
    v = Vcon.build_new()
    v.vcon_dict["vcon"] = "0.4.0"                # 1. Set the syntax parameter
    if subject is not None:
        v.vcon_dict["subject"] = subject         # 2. No setter on the class
    v.vcon_dict.pop("group", None)               # 3. Drop empty placeholder
    v.vcon_dict.pop("redacted", None)            # 4. Drop empty placeholder
    if extensions:
        v.vcon_dict["extensions"] = list(extensions)
    return v
```

Always call this (or equivalent) instead of `Vcon.build_new()` directly.

## Base adapter pattern

```python
class BaseVconAdapter(ABC):
    """Base class for vCon adapters."""

    def __init__(self, config: dict[str, Any]):
        self.config = config
        self.logger = logging.getLogger(self.__class__.__name__)

    @abstractmethod
    def extract_data(self, source: Any) -> dict[str, Any]:
        """Pull raw conversation data out of the source system."""

    @abstractmethod
    def transform_to_vcon(self, raw_data: dict[str, Any]) -> Vcon:
        """Turn raw data into a vCon."""

    def validate(self, v: Vcon) -> None:
        is_valid, errors = v.is_valid()
        if not is_valid:
            raise ValueError(f"Invalid vCon: {errors}")

    def process(self, source: Any) -> Vcon:
        raw = self.extract_data(source)
        v = self.transform_to_vcon(raw)
        self.validate(v)
        return v
```

## Parties

```python
participant_map: dict[str, int] = {}
for i, p in enumerate(raw_data.get("participants", [])):
    v.add_party(
        name=p.get("name"),
        tel=p.get("phone"),
        mailto=p.get("email"),
        sip=p.get("sip_uri"),          # OK in 0.4.0
        timezone=p.get("timezone"),    # OK in 0.4.0
        role=p.get("role", "participant"),
    )
    participant_map[p["id"]] = i
```

Do NOT pass `did=` — the `did` field was removed in 0.4.0.

## Dialogs (text, recording, video)

```python
v.add_dialog(
    type="text",                         # or "recording", "video", "transfer", "incomplete"
    start=parse_timestamp(msg["timestamp"]),
    parties=[participant_map[msg["sender_id"]]],
    originator=participant_map[msg["sender_id"]],
    mimetype="text/plain",
    body=msg["content"],
)
```

## External media (recordings)

```python
def sha512_b64url(data: bytes) -> str:
    return "sha512-" + urlsafe_b64encode(hashlib.sha512(data).digest()).rstrip(b"=").decode("ascii")

v.add_dialog(
    type="recording",
    start=parse_timestamp(media["timestamp"]),
    parties=parties,
    mediatype="audio/wav",
    url=media["url"],
    content_hash=sha512_b64url(media["bytes"]),  # MUST be sha512-<base64url>
    duration=media.get("duration"),
)
```

Do NOT emit a hex `content_hash`. Always `sha512-<base64url-of-digest>` (no `=` padding).

## Analysis (transcripts, sentiment, summaries)

```python
v.add_analysis(
    type="transcript",
    dialog=0,
    vendor="openai-whisper",                                          # REQUIRED
    product="whisper-large-v3",
    body=json.dumps(wtf_document),
    encoding="json",
    schema="https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/",
)
```

Field name is `schema`, NOT `schema_version`. `vendor` is REQUIRED — the lib raises `TypeError` if you omit it.

## Attachments (metadata, signaling, consent)

Standard core attachment — uses `purpose`:

```python
v.add_attachment(
    purpose="call_metadata",     # NEVER "type" for core attachments
    body=json.dumps({"queue": "support", "skill": "billing"}),
    encoding="json",
    party=0,                     # REQUIRED — use 0 for vCon-level
    dialog=0,                    # REQUIRED — use 0 for vCon-level
)
```

The lawful\_basis extension is the **only** documented exception — it uses `type: "lawful_basis"`. See [Extensions Cookbook](extensions-cookbook.md).

## Tags

```python
v.add_tag("source", "your_platform")
v.add_tag("call_id", raw_data["call_id"])
```

Library ≥0.9.3 writes `party: 0, dialog: 0` on the tags attachment correctly.

## Extensions

```python
v = new_vcon(extensions=["sip-signaling", "wtf", "lawful_basis"])
```

Every extension used MUST appear in top-level `extensions[]`. The template includes this in `new_vcon()`.

## NEVER write these field names

| ❌ Never          | ✅ Always   | Where                               |
| ---------------- | ---------- | ----------------------------------- |
| `appended`       | `amended`  | top-level metadata                  |
| `must_support`   | `critical` | top-level metadata                  |
| `schema_version` | `schema`   | analysis                            |
| `type`           | `purpose`  | attachments (except `lawful_basis`) |
| `did`            | (removed)  | party                               |
| `0.2.0`, `0.3.0` | `"0.4.0"`  | `vcon` syntax param                 |

## Timestamps

Always ISO-8601 with timezone:

```python
def parse_timestamp(ts: Any) -> str:
    if isinstance(ts, datetime):
        if ts.tzinfo is None:
            ts = ts.replace(tzinfo=timezone.utc)
        return ts.isoformat()
    if isinstance(ts, str):
        from dateutil import parser
        return parser.parse(ts).isoformat()
    if isinstance(ts, (int, float)):
        return datetime.fromtimestamp(ts, timezone.utc).isoformat()
    raise ValueError(f"Unparseable timestamp: {ts!r}")
```

Never emit naive datetimes.

## Dialog types

| Type           | Use for                                                  |
| -------------- | -------------------------------------------------------- |
| `"text"`       | Messages, chat, IVR prompts, individual transcript turns |
| `"recording"`  | Audio recordings                                         |
| `"video"`      | Video recordings / calls                                 |
| `"transfer"`   | Call transfers — see `add_transfer_dialog`               |
| `"incomplete"` | Failed/abandoned calls — see `add_incomplete_dialog`     |

## MIME types (mediatype)

* Text: `"text/plain"`, `"text/html"`
* Audio: `"audio/wav"`, `"audio/mp3"`, `"audio/ogg"`, `"audio/x-wav"`
* Video: `"video/mp4"`, `"video/webm"`
* Email: `"message/rfc822"`

The field name is `mediatype`, not `mimetype`, in the spec (the library accepts both as kwarg names).

## Validation

Always end `transform_to_vcon` with library validation:

```python
is_valid, errors = v.is_valid()
if not is_valid:
    raise ValueError(f"Invalid vCon: {errors}")
```

For stronger checking, copy the smoke tests from [`vcon-adapter-template/tests/test_vcon_builder.py`](https://github.com/vcon-dev/vcon-adapter-template/blob/main/tests/test_vcon_builder.py).

## Testing pattern

```python
def test_adapter_produces_compliant_vcon():
    adapter = MyAdapter({})
    v = adapter.process(sample_input)

    # Library validation
    is_valid, errors = v.is_valid()
    assert is_valid, errors

    # Spec compliance
    assert v.vcon_dict["vcon"] == "0.4.0"
    assert "group" not in v.vcon_dict
    assert "redacted" not in v.vcon_dict
    serialized = json.dumps(v.vcon_dict)
    assert "appended" not in serialized        # legacy field name
    assert "must_support" not in serialized    # legacy field name
    assert "schema_version" not in serialized  # legacy field name

    # Structural sanity
    assert len(v.vcon_dict["parties"]) > 0
    assert len(v.vcon_dict["dialog"]) > 0
```

## Delivery (downstream)

For HTTP webhook delivery, sign the body with HMAC-SHA256 (`X-Hub-Signature-256: sha256=<hex>`), key the request off the vCon `uuid` as `Idempotency-Key`, retry with exponential backoff, persist failures to a dead-letter queue. The template's [`webhook_delivery.py`](https://github.com/vcon-dev/vcon-adapter-template/blob/main/src/__ADAPTER_PACKAGE__/webhook_delivery.py) is the reference implementation. See [Operational Patterns](operational-patterns.md).

## Key considerations

1. Use the library helpers; never hand-roll `vcon_dict[...]` (except the four `new_vcon` quirks).
2. Set `vcon` syntax to `"0.4.0"`.
3. Drop empty `group: []` and `redacted: {}` from `build_new()`.
4. Attachments use `purpose` — except `lawful_basis`, which uses `type`.
5. Analysis uses `schema`, never `schema_version`. `vendor` is required.
6. Transcripts live in `analysis[]`, not `attachments[]`.
7. `content_hash` is `sha512-<base64url>`, never hex.
8. List every extension you use in top-level `extensions[]`.
9. Timestamps are ISO-8601 with timezone.
10. Validate before returning.

When generating adapter code, ground every decision on the [Spec Compliance Checklist](spec-compliance-checklist.md) and the [Extensions Cookbook](extensions-cookbook.md). If you're unsure, prefer the shape used by [`vcon-adapter-template`](https://github.com/vcon-dev/vcon-adapter-template).
