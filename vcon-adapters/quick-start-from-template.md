---
description: Scaffold a new vCon adapter in five minutes from the official template repo.
---

# 🚀 Quick Start From Template

The [vcon-adapter-template](https://github.com/vcon-dev/vcon-adapter-template) repo is a GitHub template repository. This page is the shortest path from "I have a source platform that produces conversations" to "I have a running, spec-compliant adapter delivering vCons over a signed webhook."

Spec target: [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), syntax `"0.4.0"`.

## Prerequisites

* Python 3.12+
* [`uv`](https://docs.astral.sh/uv/) (recommended) or `pip`
* `gh` CLI (or use the GitHub web UI for step 1)
* A source platform that exposes conversation data — webhook events, an API to poll, files on a disk, an S3 bucket, anything

## Step 1 — Create your repo from the template

Pick three names up front, all referring to the same adapter:

| Name                | Form           | Example              |
| ------------------- | -------------- | -------------------- |
| **Adapter name**    | kebab-case     | `signalwire`         |
| **Python package**  | snake\_case    | `signalwire_adapter` |
| **Source platform** | human-readable | `SignalWire`         |

Then create the repo:

```bash
gh repo create vcon-dev/vcon-foo-adapter \
  --template vcon-dev/vcon-adapter-template \
  --public \
  --clone

cd vcon-foo-adapter
```

## Step 2 — Run find-and-replace on placeholders

The template uses three placeholder tokens. Substitute them in one pass:

```bash
ADAPTER_NAME=foo
ADAPTER_PACKAGE=foo_adapter
SOURCE_PLATFORM=Foo

# Rename the package directory
mv "src/__ADAPTER_PACKAGE__" "src/${ADAPTER_PACKAGE}"

# Substitute placeholders in every relevant file
find . -type f \( -name "*.py" -o -name "*.toml" -o -name "*.yaml" -o -name "*.yml" -o -name "*.md" -o -name "Dockerfile" \) \
  -not -path "./.git/*" \
  -exec sed -i.bak \
    -e "s/__ADAPTER_PACKAGE__/${ADAPTER_PACKAGE}/g" \
    -e "s/__ADAPTER_NAME__/${ADAPTER_NAME}/g" \
    -e "s/__SOURCE_PLATFORM__/${SOURCE_PLATFORM}/g" \
    {} \;
find . -name "*.bak" -delete
```

Then delete the scaffolding boilerplate the template ships with:

```bash
rm USAGE.md
# In README.md, delete the "## What this is" section explaining the template
```

## Step 3 — Install and verify the smoke tests pass

```bash
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"
pytest
```

You should see all 14 spec-compliance smoke tests pass. If anything fails, you have placeholder residue or a Python version mismatch — fix before touching anything else.

## Step 4 — Wire the source-platform listener

Open `src/<your_package>/cli.py` and find the `# TODO: wire your source-platform listener` comment. Replace it with whatever fits your source:

* **Webhook receiver:** add `aiohttp.web` routes that consume incoming events
* **Polling job:** use `asyncio.create_task` to run a loop with `asyncio.sleep(interval)`
* **File watcher:** import `watchdog` and observe a directory
* **Batch CLI:** read input files, build vCons in a loop, deliver, exit

The shape of the per-event work is the same regardless:

```python
from foo_adapter.vcon_builder import new_vcon
from foo_adapter.webhook_delivery import WebhookDelivery

async def handle_event(event: dict, delivery: WebhookDelivery) -> None:
    v = new_vcon(subject=event.get("title"))
    v.add_party(tel=event["caller"], role="caller")
    v.add_party(tel=event["agent"], role="agent")
    v.add_dialog(
        type="recording",
        start=event["started_at"],
        parties=[0, 1],
        url=event["recording_url"],
        mediatype="audio/wav",
    )
    await delivery.deliver(v.vcon_dict)
```

`new_vcon()` handles the four [`Vcon.build_new()` quirks](spec-compliance-checklist.md) (syntax param, dropped empty `group`/`redacted`, `subject` setter, extensions list) for you. Use the library's `add_party` / `add_dialog` / `add_attachment` / `add_analysis` / `add_tag` helpers for everything else — they're spec-correct out of the box.

## Step 5 — Configure

Copy `config.example.yaml` to `config.yaml` and set the values. Required env vars:

| Env var                    | Purpose                              |
| -------------------------- | ------------------------------------ |
| `<PACKAGE>_API_KEY`        | Credentials for your source platform |
| `VCON_WEBHOOK_URL`         | Where to POST vCons                  |
| `VCON_WEBHOOK_HMAC_SECRET` | Shared secret for body signing       |

The config file uses `${ENV_VAR}` substitution at startup, so you never commit secrets.

## Step 6 — Run it

```bash
python -m foo_adapter
```

Or via Docker:

```bash
docker compose up
```

Then in another terminal:

```bash
curl localhost:8000/healthz   # → {"status":"ok"}
curl localhost:8000/metrics   # → Prometheus exposition
```

## Step 7 — Verify a real vCon end-to-end

Trigger an event from your source platform (or simulate one) and watch:

* Logs show `delivered url=... uuid=...` from `webhook_delivery.py`
* The `vcons_delivered_total` Prometheus counter increments
* The receiving end sees `Idempotency-Key: <uuid>` and `X-Hub-Signature-256: sha256=…` headers
* If you forcibly take the receiver down, vCons should land in `dlq/` after retries exhaust

## Step 8 — Publish

1. Push your repo: `git push -u origin main`
2. (Optional) Add to the [vcon super repo](https://github.com/vcon-dev/vcon) as a submodule
3. (Optional) Publish to PyPI — CI will do this on tag push if `PYPI_API_TOKEN` is set

## What to read next

* [Spec Compliance Checklist](spec-compliance-checklist.md) — the must/never list every PR needs to pass
* [Operational Patterns](operational-patterns.md) — what the template's delivery layer is actually doing under the hood
* [Extensions Cookbook](extensions-cookbook.md) — attaching transcripts, consent records, and SIP signaling
* [vCon Adapter Development Guide](vcon-adapter-development-guide.md) — when the template's shape isn't enough
