---
description: How to create custom processing links
icon: puzzle-piece
---

# Creating Custom Links

Links are the modular processing units of the Conserver. This guide explains how to create your own custom links to extend the system's functionality.

## Link Interface

Every link must implement a `run` function with this signature:

```python
def run(
    vcon_uuid: str,
    link_name: str,
    opts: dict = default_options
) -> str | None:
    """
    Process a vCon through this link.

    Args:
        vcon_uuid: UUID of the vCon to process
        link_name: Name of this link instance in config
        opts: Configuration options merged with defaults

    Returns:
        str: vCon UUID to continue chain (usually same as input)
        None: Stop chain processing for this vCon
    """
```

## Basic Link Template

```python
from lib.logging_utils import init_logger
from lib.vcon_redis import VconRedis

# Initialize logger
logger = init_logger(__name__)

# Default options - these can be overridden in config
default_options = {
    "enabled": True,
    "api_key": "",
    "timeout": 30,
}


def run(vcon_uuid, link_name, opts=default_options):
    """Main entry point for the link."""

    # Merge config options with defaults
    merged_opts = default_options.copy()
    merged_opts.update(opts)
    opts = merged_opts

    logger.info(f"Starting {link_name} for vCon: {vcon_uuid}")

    # Check if link is enabled
    if not opts.get("enabled", True):
        logger.info(f"Link {link_name} is disabled, skipping")
        return vcon_uuid

    try:
        # Get vCon from Redis
        vcon_redis = VconRedis()
        vcon = vcon_redis.get_vcon(vcon_uuid)

        if not vcon:
            logger.error(f"vCon not found: {vcon_uuid}")
            return None  # Stop chain processing

        # ========================================
        # YOUR PROCESSING LOGIC HERE
        # ========================================

        # Example: Add an analysis
        vcon.add_analysis(
            type="my_analysis",
            dialog=0,
            vendor="my_company",
            body={"result": "processed"},
        )

        # Example: Add a tag
        vcon.add_tag(tag_name="processed_by", tag_value=link_name)

        # ========================================
        # END PROCESSING LOGIC
        # ========================================

        # Save updated vCon back to Redis
        vcon_redis.store_vcon(vcon)

        logger.info(f"Completed {link_name} for vCon: {vcon_uuid}")
        return vcon_uuid  # Continue chain

    except Exception as e:
        logger.error(f"Error in {link_name}: {e}", exc_info=True)
        raise  # Re-raise to move vCon to DLQ
```

## Working with vCon Objects

### Reading vCon Data

```python
# Get the vCon object
vcon_redis = VconRedis()
vcon = vcon_redis.get_vcon(vcon_uuid)

# Access vCon properties
print(vcon.uuid)        # vCon UUID
print(vcon.created_at)  # Creation timestamp
print(vcon.parties)     # List of parties
print(vcon.dialog)      # List of dialog entries
print(vcon.analysis)    # List of analysis results
print(vcon.attachments) # List of attachments
print(vcon.tags)        # Dictionary of tags
```

### Processing Dialogs

```python
for index, dialog in enumerate(vcon.dialog):
    # Check dialog type
    if dialog["type"] != "recording":
        logger.info(f"Skipping non-recording dialog {index}")
        continue

    # Check for URL
    if not dialog.get("url"):
        logger.info(f"Dialog {index} has no URL")
        continue

    # Get duration
    duration = dialog.get("duration", 0)
    if duration < 30:
        logger.info(f"Skipping short dialog {index}")
        continue

    # Process the dialog
    result = process_audio(dialog["url"])

    # Add analysis result
    vcon.add_analysis(
        type="my_analysis",
        dialog=index,
        vendor="my_vendor",
        body=result,
    )
```

### Checking Existing Analysis

```python
def get_analysis_for_type(vcon, dialog_index, analysis_type):
    """Check if analysis already exists for a dialog."""
    for analysis in vcon.analysis:
        if (analysis.get("dialog") == dialog_index and
            analysis.get("type") == analysis_type):
            return analysis
    return None

# Skip if already processed
if get_analysis_for_type(vcon, index, "my_analysis"):
    logger.info(f"Dialog {index} already has my_analysis")
    continue
```

### Adding Results

```python
# Add analysis
vcon.add_analysis(
    type="summary",           # Analysis type
    dialog=0,                 # Dialog index
    vendor="openai",          # Vendor name
    body="Summary text...",   # Analysis content
    encoding="none",          # Encoding (none, json, base64)
    extra={                   # Additional metadata
        "model": "gpt-4",
        "prompt": "Summarize this",
    },
)

# Add tag
vcon.add_tag(tag_name="category", tag_value="sales")

# Add attachment
vcon.add_attachment(
    type="application/json",
    body={"key": "value"},
    encoding="json",
)
```

## Filtering and Routing

### Stopping Chain Processing

Return `None` to stop processing for this vCon:

```python
def run(vcon_uuid, link_name, opts=default_options):
    vcon_redis = VconRedis()
    vcon = vcon_redis.get_vcon(vcon_uuid)

    # Filter condition
    if "do_not_process" in vcon.tags:
        logger.info(f"Filtering out vCon {vcon_uuid}")
        return None  # Stop chain processing

    # Continue processing
    return vcon_uuid
```

### Routing to Different Queues

```python
import redis

def run(vcon_uuid, link_name, opts=default_options):
    vcon_redis = VconRedis()
    vcon = vcon_redis.get_vcon(vcon_uuid)

    # Determine destination based on tags
    if vcon.tags.get("priority") == "high":
        destination = "priority_queue"
    else:
        destination = "normal_queue"

    # Add to destination queue
    r = redis.Redis.from_url(os.getenv("REDIS_URL"))
    r.rpush(destination, vcon_uuid)

    logger.info(f"Routed {vcon_uuid} to {destination}")

    return None  # Stop current chain (routed elsewhere)
```

## External API Integration

### With Retry Logic

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    before_sleep_log,
)

@retry(
    wait=wait_exponential(multiplier=2, min=1, max=60),
    stop=stop_after_attempt(5),
    before_sleep=before_sleep_log(logger, logging.INFO),
)
def call_external_api(url, data, api_key):
    """Call external API with retries."""
    response = requests.post(
        url,
        json=data,
        headers={"Authorization": f"Bearer {api_key}"},
        timeout=30,
    )
    response.raise_for_status()
    return response.json()


def run(vcon_uuid, link_name, opts=default_options):
    vcon_redis = VconRedis()
    vcon = vcon_redis.get_vcon(vcon_uuid)

    try:
        result = call_external_api(
            opts["api_url"],
            {"vcon": vcon.to_json()},
            opts["api_key"],
        )

        vcon.add_analysis(
            type="external_analysis",
            dialog=0,
            vendor="external_service",
            body=result,
        )
        vcon_redis.store_vcon(vcon)

    except Exception as e:
        logger.error(f"API call failed after retries: {e}")
        raise

    return vcon_uuid
```

## Metrics and Monitoring

```python
from lib.metrics import init_metrics, stats_gauge, stats_count
import time

init_metrics()

def run(vcon_uuid, link_name, opts=default_options):
    start_time = time.time()

    try:
        # Your processing logic
        result = process_vcon(vcon_uuid)

        # Track success
        stats_count(
            "conserver.link.my_link.success",
            tags=[f"link:{link_name}"],
        )

        return vcon_uuid

    except Exception as e:
        # Track failure
        stats_count(
            "conserver.link.my_link.failure",
            tags=[f"link:{link_name}", f"error:{type(e).__name__}"],
        )
        raise

    finally:
        # Track processing time
        elapsed = time.time() - start_time
        stats_gauge(
            "conserver.link.my_link.processing_time",
            elapsed,
            tags=[f"link:{link_name}"],
        )
```

## Project Structure

Organize your link as a Python package:

```
my_custom_link/
├── __init__.py      # Contains run() function
├── utils.py         # Helper functions
├── models.py        # Data models
└── tests/
    ├── __init__.py
    └── test_link.py
```

### `__init__.py`

```python
from lib.logging_utils import init_logger
from lib.vcon_redis import VconRedis
from .utils import process_transcript

logger = init_logger(__name__)

default_options = {
    "model": "default",
    "threshold": 0.5,
}

def run(vcon_uuid, link_name, opts=default_options):
    # Implementation using utils
    ...
```

## Configuration

Configure your link in `config.yml`:

```yaml
links:
  my_custom_link:
    module: links.my_custom_link
    options:
      api_key: ${MY_API_KEY}
      model: "advanced"
      threshold: 0.7
      enabled: true

chains:
  main:
    links:
      - transcribe
      - my_custom_link  # Your link
      - analyze
    # ...
```

## Distribution

### As a PyPI Package

```toml
# pyproject.toml
[project]
name = "conserver-my-link"
version = "1.0.0"
dependencies = [
    "requests>=2.28.0",
]

[project.entry-points."conserver.links"]
my_link = "my_link:run"
```

### As a GitHub Repository

Reference directly in config:

```yaml
imports:
  my_custom_link:
    module: my_custom_link
    pip_name: git+https://github.com/myorg/my-link.git@v1.0.0

links:
  my_link:
    module: my_custom_link
    options:
      api_key: ${API_KEY}
```

## Testing

```python
import pytest
from unittest.mock import Mock, patch

def test_link_processes_vcon():
    # Mock VconRedis
    with patch('my_link.VconRedis') as mock_redis:
        mock_vcon = Mock()
        mock_vcon.dialog = [{"type": "recording", "url": "http://..."}]
        mock_vcon.analysis = []
        mock_redis.return_value.get_vcon.return_value = mock_vcon

        # Run link
        from my_link import run
        result = run("test-uuid", "my_link", {"enabled": True})

        # Verify
        assert result == "test-uuid"
        mock_vcon.add_analysis.assert_called_once()
        mock_redis.return_value.store_vcon.assert_called_once()


def test_link_filters_when_disabled():
    with patch('my_link.VconRedis') as mock_redis:
        from my_link import run
        result = run("test-uuid", "my_link", {"enabled": False})

        assert result == "test-uuid"
        mock_redis.return_value.get_vcon.assert_not_called()
```

## Best Practices

1. **Always merge options with defaults** - Ensures your link works even with partial configuration

2. **Check if processing already done** - Avoid reprocessing if analysis already exists

3. **Use structured logging** - Include vCon UUID and link name in all log messages

4. **Handle errors appropriately** - Raise exceptions to send vCons to DLQ, or return UUID to continue

5. **Add metrics** - Track processing time, success/failure rates

6. **Document your options** - Make it clear what configuration your link accepts

7. **Test thoroughly** - Unit test your processing logic

8. **Be idempotent** - Running twice should produce the same result
