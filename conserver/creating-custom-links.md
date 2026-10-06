---
description: How to write a conserver link, with the Vcon helper methods, the return and error rules, metrics, packaging and tests, checked against vcon-server.
---

# 🧩 Creating Custom Links

A link is a Python module with a `run` function. The conserver imports it by name from `config.yml` and calls `run` once for each vCon that reaches it. This page shows how to write one. For what the return value does to a chain, see [Concepts](concepts.md#link).

## The interface

```python
def run(vcon_uuid: str, link_name: str, opts: dict | None) -> str | None:
    ...
```

* `vcon_uuid` is the UUID of the vCon. The link loads the vCon itself.
* `link_name` is the name of the entry under `links:`, so one module can serve several entries.
* `opts` is the `options:` block from that entry. It is `None` when the entry has no `options`, so merge it into your defaults with `opts or {}`.

| Return | Effect |
| ------ | ------ |
| The same UUID | The chain continues. |
| A different UUID | The chain continues with that vCon. |
| `None` or any falsy value | The chain stops for this vCon. Nothing is stored. |
| Raise an exception | The chain stops and the UUID goes to `DLQ:<ingress_list>`. |

Return `None` to filter on purpose, and raise to report a failure that someone should replay. A link that catches an error and returns the UUID hides the failure from the dead letter queue.

## A template

```python
from lib.logging_utils import init_logger
from lib.vcon_redis import VconRedis

logger = init_logger(__name__)

default_options = {
    "threshold": 0.5,
    "tag_name": "processed_by",
}


def run(vcon_uuid, link_name, opts=None):
    options = {**default_options, **(opts or {})}

    vcon_redis = VconRedis()
    vcon = vcon_redis.get_vcon(vcon_uuid)
    if vcon is None:
        # In neither Redis nor any storage. None is the "halt" signal.
        logger.error("%s: vCon %s not found", link_name, vcon_uuid)
        return None

    vcon.add_analysis(
        type="my_analysis",
        dialog=0,
        vendor="example",
        body={"score": 0.9},
        extra={"vendor_schema": {"threshold": options["threshold"]}},
    )
    vcon.add_tag(tag_name=options["tag_name"], tag_value=link_name)

    vcon_redis.store_vcon(vcon)
    return vcon_uuid
```

Save it as `conserver/links/my_link/__init__.py` in your build, and configure it:

```yaml
links:
  my_link:
    module: links.my_link
    options:
      threshold: 0.7
```

Nothing is saved until you call `store_vcon`. The method stamps `vcon: "0.4.0"`, renames legacy fields to the current spec, and keeps the vCon's existing TTL.

## Reading a vCon

`VconRedis().get_vcon(uuid)` returns a `Vcon` object, or `None`. On a Redis miss it tries each configured storage, so a vCon that expired from Redis can still load.

| Member | Returns |
| ------ | ------- |
| `vcon.uuid`, `vcon.created_at`, `vcon.subject` | Top-level fields |
| `vcon.parties`, `vcon.dialog`, `vcon.analysis`, `vcon.attachments` | Lists of dicts |
| `vcon.to_dict()` | The whole vCon as a dict |
| `vcon.find_attachment_by_purpose("lawful_basis")` | The first attachment with that `purpose`, or `None` |
| `vcon.get_tag("priority")` | The value of the tag `priority:<value>`, or `None` |
| `vcon.tags` | The `tags` attachment itself, or `None`. It is not a dict of tags. Use `get_tag` |
| `Vcon.decoded_body(entry)` | An analysis or attachment body as a Python value, whatever its `encoding` |

Use `decoded_body` before you read a body. With `encoding: "json"` the current spec makes `body` the JSON value itself, and older producers stored a string. The helper returns a value in either case.

```python
from vcon import Vcon

for index, dialog in enumerate(vcon.dialog):
    if dialog.get("type") != "recording" or not dialog.get("url"):
        continue
    if (dialog.get("duration") or 0) < 30:
        continue
    if any(a["dialog"] == index and a["type"] == "my_analysis" for a in vcon.analysis):
        continue  # already done, so a replay does no extra work
    ...
```

## Writing results

```python
# Analysis. A dict or list body is stored as JSON, and encoding becomes "json".
vcon.add_analysis(type="summary", dialog=0, vendor="example",
                  body="The caller asked about billing.", encoding="none",
                  extra={"vendor_schema": {"model": "example-model"}})

# Tag. Stored in the attachment with purpose "tags" as "name:value".
vcon.add_tag(tag_name="category", tag_value="billing")

# Attachment. The argument is named type. The vCon stores it as purpose on store_vcon.
vcon.add_attachment(type="my_report", body={"items": 3}, encoding="json")
```

`encoding` is `none`, `json` or `base64url`. A string body with `encoding="json"` must parse as JSON, and a `base64url` body must decode, or the call raises. `extra` is merged into the analysis entry, which is where `vendor_schema` goes. When you record your options, drop credentials first with `from lib.redaction import safe_opts`.

Create new attachments with a `purpose` that names what they are. If your link builds a new vCon, carry across the attachment with `purpose: "lawful_basis"` from the source, or add one that states the basis for the new vCon. Do not invent a basis. See the [Lawful Basis extension](../extensions/lawful-basis.md).

## Filtering and routing

Return `None` to filter:

```python
def run(vcon_uuid, link_name, opts=None):
    vcon = VconRedis().get_vcon(vcon_uuid)
    if vcon is None:
        return None
    if vcon.get_tag("do_not_process") is not None:
        return None
    return vcon_uuid
```

To send a vCon to another chain, push its UUID onto that chain's ingress list with `VconQueue`:

```python
from lib.queue import VconQueue

destination = "priority_in" if vcon.get_tag("priority") == "high" else "normal_in"
VconQueue().enqueue(destination, vcon_uuid)
return None  # this chain stops here
```

`only_if` and `sampling_rate`, the options the analysis links share, are in `lib.links.filters` as `is_included(opts, vcon)` and `randomly_execute_with_sampling(opts)`. Use them to behave the same way.

## Calling external services

Keep the API key in the link's options, because the conserver does not expand `${VAR}` or read provider keys from the environment. Retry transient failures with `tenacity`, and set a timeout on every request. The conserver has no chain timeout, so a call without one can hold a worker indefinitely.

```python
import logging
import requests
from tenacity import before_sleep_log, retry, stop_after_attempt, wait_exponential

from lib.logging_utils import init_logger

logger = init_logger(__name__)


@retry(
    wait=wait_exponential(multiplier=2, min=1, max=60),
    stop=stop_after_attempt(5),
    before_sleep=before_sleep_log(logger, logging.INFO),
)
def call_service(url, payload, api_key):
    response = requests.post(
        url, json=payload, headers={"Authorization": f"Bearer {api_key}"}, timeout=30
    )
    response.raise_for_status()
    return response.json()
```

For OpenAI-compatible calls, use `lib.openai_client.get_openai_client(opts)`. It gives your link the same OpenAI, Azure and LiteLLM options as the [standard analysis links](standard-links.md#conventions).

## Metrics

Use the two functions in `lib.metrics`. The older `init_metrics`, `stats_gauge` and `stats_count` log a deprecation warning and record nothing.

```python
from lib.metrics import increment_counter, record_histogram
import time

started = time.time()
try:
    ...
    increment_counter("conserver.link.my_link.success", attributes={"link.name": link_name})
except Exception:
    increment_counter("conserver.link.my_link.failure", attributes={"link.name": link_name})
    raise
finally:
    record_histogram(
        "conserver.link.my_link.processing_time",
        time.time() - started,
        attributes={"link.name": link_name},
    )
```

Do not put the vCon UUID in `attributes`. Every distinct value makes a new time series and breaks percentile queries. The conserver already counts every link in `conserver.link.count` and times it in `conserver.link.execution_time`, and puts the UUID on the trace span. Metrics export only when `OTEL_EXPORTER_OTLP_ENDPOINT` is set. See [Production Deployment](production-deployment.md#metrics).

## Hooks around links

A link entry can carry an `after_link` block inside `options`. After the link returns or raises, the conserver calls `conserver/after_link_hook.py` with that block, the status and the vCon's party identifiers. Replace the file at image build time to add audit logging or notifications without touching your links. See [Configuring the Conserver](configuring-the-conserver.md#links).

## Packaging

The conserver imports `module:` as an ordinary Python module, so any importable module works. There is no plugin registry. A `conserver.links` entry point in `pyproject.toml` is not read.

* **Built into the image.** Put the package under `conserver/links/`, and use `module: links.my_link`.
* **Installed from PyPI or GitHub.** Name it under `imports:`. The conserver runs `pip install` when a worker starts.

```yaml
imports:
  my_link_pkg:
    module: my_link_pkg
    pip_name: git+https://github.com/example/my-link.git@v1.0.0

links:
  my_link:
    module: my_link_pkg
    options:
      threshold: 0.7
```

The package must expose `run` at its top level. Pin the version in `pip_name`, because the install happens in your worker.

## Testing

Run tests inside the image so the imports resolve:

```bash
docker compose run --rm conserver pytest conserver/links/my_link/ -v
```

Patch `VconRedis` and check what the link did to the vCon object:

```python
from unittest.mock import Mock, patch

def test_run_adds_analysis():
    vcon = Mock()
    vcon.dialog = [{"type": "recording", "url": "https://example.com/a.wav", "duration": 60}]
    vcon.analysis = []

    with patch("links.my_link.VconRedis") as redis_cls:
        redis_cls.return_value.get_vcon.return_value = vcon
        from links.my_link import run

        assert run("0192f4f0-7a52-8c3d-9a1b-2c3d4e5f6a7b", "my_link", None) == "0192f4f0-7a52-8c3d-9a1b-2c3d4e5f6a7b"

    vcon.add_analysis.assert_called_once()
    redis_cls.return_value.store_vcon.assert_called_once_with(vcon)


def test_run_halts_when_vcon_missing():
    with patch("links.my_link.VconRedis") as redis_cls:
        redis_cls.return_value.get_vcon.return_value = None
        from links.my_link import run

        assert run("missing", "my_link", {}) is None
```

## Practices

1. Merge `opts or {}` into your defaults, because `opts` can be `None`.
2. Check whether the work is already done and skip it. A dead letter replay reruns the chain, and a worker restart can too.
3. Return `None` for a deliberate filter and raise for a failure.
4. Put a timeout on every network call.
5. Keep credentials in `options`, and strip them with `safe_opts` before you log or store options.
6. Keep the vCon UUID out of metric attributes.
7. Handle `get_vcon` returning `None`.
