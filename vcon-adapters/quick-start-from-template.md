---
description: Scaffold a new vCon adapter from the official template repo and get a first vCon delivered.
---

# 🚀 Quick Start From Template

The [vcon-adapter-template](https://github.com/vcon-dev/vcon-adapter-template) repo is a GitHub template repository. This page takes you from a source platform that produces conversations to a running adapter delivering vCons over a signed webhook or straight to a conserver. The rules your code must follow are on the [Spec Compliance Checklist](spec-compliance-checklist.md), and the template's tests enforce them.

## Prerequisites

* Python 3.12+
* [`uv`](https://docs.astral.sh/uv/) (recommended) or `pip`
* `gh` CLI, or the GitHub web UI for step 1
* A source platform that exposes conversation data: webhook events, a pollable API, files on disk, or an S3 bucket

## Step 1. Create your repo

Pick three names that refer to the same adapter:

| Name | Form | Example |
| --- | --- | --- |
| Adapter name | kebab-case | `foo` |
| Python package | snake\_case | `foo_adapter` |
| Source platform | human-readable | `Foo` |

```bash
gh repo create vcon-dev/vcon-foo-adapter \
  --template vcon-dev/vcon-adapter-template \
  --public \
  --clone

cd vcon-foo-adapter
```

## Step 2. Replace the placeholders

```bash
ADAPTER_NAME=foo
ADAPTER_PACKAGE=foo_adapter
SOURCE_PLATFORM=Foo

mv "src/__ADAPTER_PACKAGE__" "src/${ADAPTER_PACKAGE}"

find . -type f \( -name "*.py" -o -name "*.toml" -o -name "*.yaml" -o -name "*.yml" -o -name "*.md" -o -name "Dockerfile" \) \
  -not -path "./.git/*" \
  -exec sed -i.bak \
    -e "s/__ADAPTER_PACKAGE__/${ADAPTER_PACKAGE}/g" \
    -e "s/__ADAPTER_NAME__/${ADAPTER_NAME}/g" \
    -e "s/__SOURCE_PLATFORM__/${SOURCE_PLATFORM}/g" \
    {} \;
find . -name "*.bak" -delete
```

In `README.md`, delete the "What this is" section that explains the template. Keep `USAGE.md` until you have read it: it holds the lawful basis setup and the `finalize_vcon()` and `json_body()` notes.

## Step 3. Install and run the tests

```bash
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"
pytest
```

The whole test suite should pass before you change anything. A failure at this point means placeholder residue or a Python version mismatch.

## Step 4. Wire the source listener

Open `src/<your_package>/cli.py` and find `# TODO: wire your source-platform listener`. Pick the shape that fits your source (webhook receiver, polling job, file watcher, batch CLI); the [Development Guide](vcon-adapter-development-guide.md) has a skeleton for each.

The per-event work looks like this:

```python
from datetime import datetime, timezone

from vcon.dialog import Dialog
from vcon.party import Party

from foo_adapter.vcon_builder import add_lawful_basis, external_media_url, new_vcon

async def handle_event(event: dict, audio: bytes, config, delivery) -> None:
    v = new_vcon(subject=event.get("title"))
    v.add_party(Party(tel=event["caller"], role="caller"))
    v.add_party(Party(tel=event["agent"], role="agent"))

    media = external_media_url(
        url=event["recording_url"],
        content=audio,
        mediatype="audio/wav",
    )
    v.add_dialog(
        Dialog(
            start=event["started_at"],
            parties=[0, 1],
            **media,  # type, url, mediatype, content_hash (sha512-<base64url>)
        )
    )

    add_lawful_basis(
        v,
        config.lawful_basis,
        granted_at=datetime.now(timezone.utc).isoformat(),
    )
    await delivery.deliver(v.vcon_dict)
```

`external_media_url()` computes `content_hash` with `sha512_b64url()`. Replace `datetime.now()` with the time your source says the basis was established when it tells you. If `LAWFUL_BASIS` is unset, `add_lawful_basis()` logs one warning and adds nothing; it never invents a basis. See the [Extensions Cookbook](extensions-cookbook.md#lawful-basis) for what the attachment looks like.

`deliver()` calls `finalize_vcon()` for you. If you serialize a vCon anywhere else, such as straight to local storage, call `finalize_vcon(v.vcon_dict)` first.

## Step 5. Configure

Copy `config.example.yaml` to `config.yaml`. The file substitutes `${ENV_VAR}` at startup, so secrets stay out of it.

| Env var | Purpose |
| --- | --- |
| `<PACKAGE>_API_KEY` | Credentials for your source platform |
| `VCON_WEBHOOK_URL` | Where to POST vCons (`delivery.mode: webhook`) |
| `VCON_WEBHOOK_HMAC_SECRET` | Shared secret for body signing (`delivery.mode: webhook`) |
| `CONSERVER_URL` | Base URL of a vcon-server (`delivery.mode: conserver`) |
| `CONSERVER_API_TOKEN` | Sent as `x-conserver-api-token` (`delivery.mode: conserver`) |
| `LAWFUL_BASIS` | One of `consent`, `contract`, `legal_obligation`, `vital_interests`, `public_task`, `legitimate_interests`. Unset means no attachment |
| `LAWFUL_BASIS_PURPOSE` | Comma-separated purposes, default `recording` |
| `LAWFUL_BASIS_JURISDICTION` | Optional, for example `US-MA` |
| `LAWFUL_BASIS_EXPIRATION` | Optional ISO 8601 timestamp |
| `LAWFUL_BASIS_PROOF_MECHANISM` | Optional, for example `external_system` |
| `LAWFUL_BASIS_PROOF_DESCRIPTION` | Optional free text for the proof |

Set `delivery.mode` to `webhook` (the default) or `conserver`. Conserver mode POSTs to `{CONSERVER_URL}/vcon` with the token header and an `ingress_lists` query parameter per configured list. Both modes share the retry and dead-letter code ([Operational Patterns](operational-patterns.md)). Each `LAWFUL_BASIS*` variable overrides the matching `vcon.lawful_basis` YAML field.

## Step 6. Run it

```bash
python -m foo_adapter     # or: docker compose up
curl localhost:8000/healthz   # {"status":"ok"}
curl localhost:8000/metrics   # Prometheus exposition
```

## Step 7. Verify a real vCon

Trigger or simulate an event and check:

* The log shows `delivered url=... uuid=...`
* `vcons_delivered_total` increments
* A webhook receiver sees `Idempotency-Key: <uuid>` and `X-Hub-Signature-256: sha256=...`; a conserver sees the `x-conserver-api-token` header
* With the receiver down, vCons land in `dlq/` after retries run out
* The delivered vCon has a `purpose: "lawful_basis"` attachment, or you saw the one-time warning that says why not

## Step 8. Publish

Push the repo with `git push -u origin main`. The template ships one workflow, `test.yml` (lint, type check, tests on Python 3.12 and 3.13). It has no PyPI publish step, so adding one is up to you.

## What to read next

* [Spec Compliance Checklist](spec-compliance-checklist.md)
* [Operational Patterns](operational-patterns.md)
* [Extensions Cookbook](extensions-cookbook.md)
* [vCon Adapter Development Guide](vcon-adapter-development-guide.md)
