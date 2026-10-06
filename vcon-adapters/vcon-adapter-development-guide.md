---
description: >-
  Design patterns for adapter authors going beyond the template, custom
  architectures, listener shapes, multi-source merging, batch processing.
---

# 🔗 vCon Adapter Development Guide

The [Quick Start From Template](quick-start-from-template.md) covers \~90% of new adapters. This page is for the other 10%, the cases where the template's shape isn't quite right, where you need to think about architecture rather than just substitute placeholders.

If you haven't read the [Quick Start From Template](quick-start-from-template.md) and the [Spec Compliance Checklist](spec-compliance-checklist.md) yet, do that first. This page assumes both.

The spec target and every field rule are on the [Spec Compliance Checklist](spec-compliance-checklist.md); field definitions are in the [vCon field reference](../vcons/field-reference.md). This page does not repeat them.

## Listener shapes

How conversation data arrives at your adapter dictates the listener shape. Four common ones, with worked examples from the ecosystem:

### Webhook receiver

Source platforms that push events to you. The adapter exposes an HTTP endpoint, validates the source's signature on the incoming webhook, builds a vCon, and delivers it downstream.

* **When to use:** SignalWire (call.ended), Twilio (call status), Slack (message events), Zoom (meeting.recording.completed), any modern SaaS with outbound webhooks.
* **Skeleton:**

```python
from datetime import datetime, timezone

from aiohttp import web
from vcon.dialog import Dialog
from vcon.party import Party

from foo_adapter.vcon_builder import add_lawful_basis, external_media_url, new_vcon

async def on_call_ended(request: web.Request) -> web.Response:
    # 1. Validate the SOURCE's signature (NOT the same as your downstream HMAC)
    if not verify_source_signature(request):
        return web.Response(status=401)

    event = await request.json()
    audio = await fetch_recording(event["recording_url"])  # bytes, for content_hash

    # 2. Build the vCon
    v = new_vcon(subject=event.get("title"))
    v.add_party(Party(tel=event["from"], role="caller"))
    v.add_party(Party(tel=event["to"], role="agent"))
    v.add_dialog(Dialog(
        start=event["started_at"],
        parties=[0, 1],
        **external_media_url(url=event["recording_url"], content=audio, mediatype="audio/wav"),
    ))
    add_lawful_basis(
        v,
        request.app["config"].lawful_basis,
        granted_at=datetime.now(timezone.utc).isoformat(),
    )

    # 3. Deliver downstream
    await request.app["delivery"].deliver(v.vcon_dict)
    return web.Response(status=200)
```

`external_media_url()` sets `url`, `mediatype` and a `sha512-<base64url>` `content_hash`. `add_lawful_basis()` records the basis your deployment configured and adds nothing, with one warning, when none is set. It never invents one.

The template's `health_server.py` already has an aiohttp app; add your routes to it rather than starting a second server.

### Polling job

Source platforms that don't push but expose a list/since API. The adapter loops on a schedule, asks "anything new since `cursor`?", processes results, advances the cursor.

* **When to use:** REST APIs without webhooks, older PBX systems, ElevenLabs (poll for completed jobs).
* **State management:** persist the cursor somewhere durable: a file, a tiny SQLite DB, Redis. Don't keep it in memory; restarts will replay or skip events.
* **Skeleton:**

```python
import asyncio

async def poll_loop(client, delivery, state_path: Path, interval: float = 300) -> None:
    cursor = load_cursor(state_path)
    while True:
        events = await client.list_completed_since(cursor)
        for event in events:
            v = build_vcon(event)
            ok = await delivery.deliver(v.vcon_dict)
            if ok:
                cursor = event["completed_at"]
        save_cursor(state_path, cursor)
        await asyncio.sleep(interval)
```

[`signalwire_adapter`](https://github.com/vcon-dev/signalwire_adapter) is a polling-based reference. The repo is archived and predates the template, so copy its polling shape, not its vCon construction.

### File watcher

Source produces files on a disk or in an object store. The adapter watches the location and ingests new files.

* **When to use:** legacy systems that drop CDRs as XML/CSV, audio recorders that write WAV files, a softswitch writing to S3.
* **Local disk:** [`watchdog`](https://pypi.org/project/watchdog/) is the standard library.
* **S3:** poll the bucket with `list_objects_v2` and a marker, or wire up SQS notifications.
* **Critical:** track which files you've already processed. A `.processed` sibling file, a manifest, or a small DB. Re-ingesting the same file produces a _new_ UUID and a duplicate vCon downstream.

[`vcon-audio-adapter`](https://github.com/vcon-dev/vcon-audio-adapter) demonstrates the directory-watch pattern.

### Batch CLI

One-shot: read a manifest of records, build vCons, deliver, exit. Run from cron, a Kubernetes Job, a GitHub Action, an ad-hoc terminal.

* **When to use:** backfilling historical data, processing closed datasets (IETF meeting recordings, archive imports), CI-time generation of test fixtures.
* **Pattern:** add a `click` or `argparse` command to `cli.py` that reads the input, loops, and uses the same `WebhookDelivery` for output.

[`ietf2vcon`](https://github.com/vcon-dev/ietf2vcon) is a batch-CLI reference.

## Multi-source adapters

Sometimes one logical conversation lives across two sources, e.g. recording URLs in one system and transcripts in another. Two approaches:

### Stitch in the adapter

Wait until both sides arrive, build a single vCon, deliver. Use this when latency between sides is short (seconds to minutes) and missed-correlation rate is low.

```python
class CallCorrelator:
    def __init__(self):
        self.pending: dict[str, dict] = {}  # call_id → partial event

    def on_event(self, event: dict) -> Vcon | None:
        cid = event["call_id"]
        existing = self.pending.get(cid, {})
        merged = {**existing, **event}
        if "recording_url" in merged and "transcript" in merged:
            self.pending.pop(cid, None)
            return build_vcon(merged)
        self.pending[cid] = merged
        return None
```

Persist `self.pending` across restarts (Redis, SQLite). Set a TTL, after some maximum delay, deliver the partial vCon with whatever you have rather than dropping it.

### Stitch downstream

Deliver one vCon per source, mark each with a correlation tag, let the conserver merge them via a [link](../conserver/standard-links.md).

```python
v.add_tag("correlation_id", event["call_id"])
v.add_tag("source_role", "recording")  # or "transcript"
```

This is simpler, your adapter stays stateless, and shifts the correlation cost to a place that already does pipeline work. Prefer this unless you have a specific reason not to.

## Choosing where data lives in the vCon

A constant question for adapter authors: "where does X go?" The shortest answer:

| What                                   | Where                                         | Why                                 |
| -------------------------------------- | --------------------------------------------- | ----------------------------------- |
| Audio/video recordings                 | `dialog[]` (`type: "recording"` or `"video"`) | The recording _is_ the conversation |
| Text messages, chat lines, IVR prompts | `dialog[]` (`type: "text"`)                   | Each utterance is a dialog turn     |
| Failed calls (no-answer, busy)         | `dialog[]` (`type: "incomplete"`)             | Still a conversation event          |
| Transcripts (WTF or otherwise)         | `analysis[]`                                  | Derived FROM the recording          |
| Sentiment, summaries, intent labels    | `analysis[]`                                  | Derived FROM the conversation       |
| SIP Call-ID and dialog tags            | `dialog[]` fields `sip_call_id`, `sip_from_tag`, `sip_to_tag` | Correlation keys on the call leg |
| SIP messages, traces, SDP, STIR data   | `attachments[]` (`sip-invite`, `sip-message-trace`, `sip-sdp`, `stir-passport-extended`, and the rest of the draft's purposes) | Signaling metadata about the call   |
| Recording consent, GDPR basis          | `attachments[]` (`purpose: "lawful_basis"`, built by `add_lawful_basis()`) | Legal metadata about the recording  |
| CRM ticket IDs, source row IDs         | `tags` attachment via `add_tag()`             | Lookup keys for joins               |
| Agent name, queue name                 | `party.role`, `party.name`                    | Party metadata                      |
| AI agent identity and tool trace       | `parties[].meta.agent_session`, an `agent_trace` analysis | See the Agent Session draft |

When in doubt: **derived data → `analysis[]`; supplied metadata → `attachments[]`; the conversation itself → `dialog[]`**.

## When NOT to use the template

The template is opinionated. If your situation conflicts with its opinions, fork the structure but keep the [Spec Compliance Checklist](spec-compliance-checklist.md):

* **Non-HTTP delivery.** Kafka, NATS, gRPC, files on a shared volume, an S3 bucket. The `webhook_delivery.py` shape doesn't fit, but the principles: sign the bytes, idempotency key off the UUID, retry with backoff, persist failures: still apply.
* **Streaming / partial vCons.** If you need to emit progressively-updated vCons as a long call unfolds, the [Lifecycle extension](../extensions/lifecycle.md) is the right pattern, not webhook fan-out.
* **Embedded inside another service.** Sometimes "the adapter" is a function called inside a larger app. Just import `vcon_builder.py` and skip the CLI/health-server scaffolding.
* **Non-Python.** Use [`vcon-js`](../vcon-js-library/) for Node, hand-port to other languages with the same checklist. The spec is language-neutral; the templates are not.

## Validating what you built

The library has a validator:

```python
is_valid, errors = v.is_valid()
if not is_valid:
    raise ValueError(f"Generated invalid vCon: {errors}")
```

The template's `tests/test_spec_compliance.py` is a stronger gate. It checks a sample vCon against the vendored official JSON schema and enforces rules the schema alone does not. Copy it and the schema into a non-template adapter. The authoritative spec is the draft itself; when a page here disagrees with [`draft-ietf-vcon-vcon-core-04`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) or an extension draft, the draft wins and the page is a bug.

## Common drift

Older adapters in the ecosystem repeat the same mistakes: hand-rolled dicts that bypass the library, legacy field names, hex `content_hash`, timestamps without a timezone, empty placeholder objects, transcripts in `attachments[]`, and no lawful basis. Each has a rule on the [checklist](spec-compliance-checklist.md). Another is sloppy file watching: re-ingesting a file creates a new UUID and a duplicate vCon downstream, so track what you have processed.

## Further reading

* [Spec Compliance Checklist](spec-compliance-checklist.md): the gate every PR should pass
* [Operational Patterns](operational-patterns.md): production delivery concerns
* [Extensions Cookbook](extensions-cookbook.md): WTF, lawful basis, SIP signaling, agent session
* [vCon Library (Python)](../vcon-library/): full library API reference
* [vcon-speckit](https://github.com/vcon-dev/vcon-speckit) for spec-compliance notes when you write vCon JSON by hand
