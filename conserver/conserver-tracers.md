---
description: What a conserver tracer is, exactly when it runs, how its errors are handled, and how to configure the JLINC tracer or write your own.
---

# 🩻 Conserver Tracers

A tracer is a module that the conserver calls around each link so it can record what happened to a vCon. Tracers are for audit trails. A tracer receives vCon UUIDs, not vCon data, and a well-behaved one reads the vCon from Redis and changes nothing. One tracer ships with vcon-server: `jlinc`. DataTrails and SCITT are available as [links](standard-links.md#audit-links), and SCITT also as a [storage](storage.md#scitt-transparency), not as tracers.

## Tracers and links

| | Link | Tracer |
| --- | ---- | ------ |
| Declared in | `links:`, then named by a chain | `tracers:`, which applies to every chain |
| Return value | A UUID to continue, or a falsy value to stop | Ignored |
| Runs | Once per link, in chain order | Before the first link, and after each link |
| An exception | Sends the vCon to the dead letter queue | Logged and ignored, unless `dlq_vcon_on_error` is `true` |

A tracer runs inline in the worker. The chain waits for it. A slow tracer slows every vCon, so a tracer that calls a network service adds its latency to each link.

## When tracers run

For each vCon, in each chain, the conserver calls every configured tracer:

1. Once before the first link, with `link_index` of `-1`.
2. After each link that returns, with that link's index (`0`, `1`, `2`, ...). This includes a link that returns a falsy value and stops the chain.

A link that raises gets no call. There is no call when the chain finishes. A chain of three links produces four calls per tracer.

## Configuration

Tracers are defined at the top level of `config.yml`. They are not part of a chain. A `tracers:` key inside a chain, which `example_config.yml` shows in a comment, is not read.

```yaml
tracers:
  jlinc:
    module: tracers.jlinc
    options:
      data_store_api_url: http://jlinc-server:9090
      data_store_api_key: "your-data-store-key"
      archive_api_url: http://jlinc-server:9090
      archive_api_key: "your-archive-key"
      system_prefix: VCONProd
```

Give every tracer an `options:` block, even an empty one. When a tracer raises and `options` is missing, the conserver cannot check `dlq_vcon_on_error` and fails the chain. Values are literal. The conserver does not expand `${VAR}`. Like links, a tracer entry can carry a `pip_name` to install its package.

## Error handling

The conserver wraps each tracer call in a `try`. If the tracer raises, the error is logged with the tracer name, and the chain continues. If the tracer's options set `dlq_vcon_on_error: true`, the conserver re-raises the error instead. The exception stops the chain, and the vCon goes to the ingress `DLQ:<list>` like any failed link. See [Concepts](concepts.md#dead-letter-queues).

A tracer that returns `False` has not raised, so nothing happens. The return value is never read.

## The JLINC tracer

The `jlinc` tracer sends an event to a JLINC data store for each transition. The data store signs the event, which gives you an audit trail of which vCon passed between which steps.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `data_store_api_url`, `data_store_api_key` | JLINC data store URL and key | `http://jlinc-server:9090`, empty |
| `archive_api_url`, `archive_api_key` | JLINC archive URL and key, sent with each event | `http://jlinc-server:9090`, empty |
| `system_prefix` | Prefix for the entity names | `VCONTest` |
| `agreement_id` | JLINC agreement id. The all-zero value is the general auditing agreement | `00000000-0000-0000-0000-000000000000` |
| `hash_event_data` | Send the SHA-256 hash of the vCon JSON instead of the vCon | `true` |
| `dlq_vcon_on_error` | Dead-letter the vCon when the tracer raises. See below | `false` |

On the first call it asks the data store for its domain and creates a system entity named `<prefix>-system@<domain>`. Each link gets an entity named `<prefix>-<link name>@<domain>`, created on first use and cached per process. For each call it sends an event between two of these entities, with the in and out vCon UUIDs as metadata.

With `hash_event_data: false` the whole vCon goes to the JLINC server, so leave it `true` unless the data store is inside your trust boundary.

The tracer catches errors from the JLINC API and logs a warning, so a JLINC outage does not stop a chain, and `dlq_vcon_on_error: true` does not change that. That option matters for tracers that let errors escape.

## Writing a tracer

A tracer module has a `run` function with this signature:

```python
def run(
    in_vcon_uuid: str,   # the UUID going into this step
    out_vcon_uuid: str,  # the UUID coming out of it
    tracer_name: str,    # the name from tracers: in config.yml
    links: list[str],    # every link name in the chain, in order
    link_index: int,     # -1 before the first link, else the link just run
    opts: dict,          # the tracer's options
) -> bool:
    ...
```

The conserver does not read the return value, but return `True` on success and `False` when you skip, as `jlinc` does.

```python
from lib.logging_utils import init_logger
from lib.vcon_redis import VconRedis

logger = init_logger(__name__)

default_options = {"dlq_vcon_on_error": False}


def run(in_vcon_uuid, out_vcon_uuid, tracer_name, links, link_index, opts=default_options):
    vcon = VconRedis().get_vcon(out_vcon_uuid)
    if vcon is None:
        logger.error("Tracer %s: vCon %s not found", tracer_name, out_vcon_uuid)
        return False

    step = "start" if link_index < 0 else links[link_index]
    logger.info(
        "Tracer %s: vCon %s after %s, %d analysis entries",
        tracer_name, out_vcon_uuid, step, len(vcon.analysis),
    )
    return True
```

Put the module where the worker can import it, as for a [custom link](creating-custom-links.md#packaging). Keep it fast, since it blocks the chain. Never write to the vCon. If an external system is unreliable, catch its errors inside `run`, as the JLINC tracer does, unless you want a failure to dead-letter the vCon.

## Reading tracer activity

The conserver logs each call at debug level with `tracer_name`, `tracer_module_name` and `tracer_processing_time` as structured fields. A tracer error appears at error level as `Error in tracer <name> (module: <module>) for vCon <uuid>`. Tracers add no metrics of their own.
