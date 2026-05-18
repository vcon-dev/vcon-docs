# Standard Links

Links are the processing units of the Conserver. Each link performs a specific operation on a vCon as it flows through a chain. Links can analyze content, transform data, route vCons, integrate with external services, and more.

The conserver currently ships **22 standard links**. They are organized in this page by what they do:

| Category | Links |
|----------|-------|
| **Transcription** | `deepgram_link`, `groq_whisper`, `hugging_face_whisper`, `openai_transcribe`, `transcribe`, `wtf_transcribe` |
| **Analysis** | `analyze`, `analyze_vcon`, `analyze_and_label`, `check_and_tag`, `detect_engagement`, `hugging_llm_link` |
| **Routing & filtering** | `sampler`, `jq_link`, `tag_router` |
| **Data management** | `tag`, `diet`, `expire_vcon` |
| **Integration** | `webhook`, `post_analysis_to_slack` |
| **Audit & compliance** | `scitt`, `datatrails` |

All links emit OpenTelemetry metrics (latency, error counts, cache hits where applicable) and trace spans. If you've wired up the [`vcon-mcp-adapters`](../tools/vcon-mcp-adapters.md) OTEL collector, you'll see per-link spans automatically.

## Link Interface

All links implement the same interface:

```python
def run(vcon_uuid: str, link_name: str, opts: dict = default_options) -> str | None:
    """
    Process a vCon through this link.

    Args:
        vcon_uuid: UUID of the vCon to process
        link_name: Name of this link in the configuration
        opts: Configuration options merged with defaults

    Returns:
        vcon_uuid: Continue processing with this vCon UUID
        None: Stop chain processing (filter out this vCon)
    """
```

## Available Links

### Transcription Links

These links convert audio recordings in vCon dialogs to text transcripts.

---

#### deepgram_link

Speech-to-text transcription using the Deepgram API with automatic language detection and confidence scoring.

```yaml
links:
  deepgram:
    module: links.deepgram_link
    options:
      DEEPGRAM_KEY: "your-api-key"
      minimum_duration: 30
      api:
        model: "nova-2"
        smart_format: true
        detect_language: true
```

| Option | Description | Default |
|--------|-------------|---------|
| `DEEPGRAM_KEY` | Deepgram API key | (required) |
| `minimum_duration` | Minimum audio duration in seconds to transcribe | `30` |
| `api.model` | Deepgram model to use | `nova-2` |
| `api.smart_format` | Enable smart formatting | `true` |
| `api.detect_language` | Enable automatic language detection | `true` |

---

#### groq_whisper

Speech-to-text transcription using Groq's implementation of the Whisper ASR model.

```yaml
links:
  groq_whisper:
    module: links.groq_whisper
    options:
      GROQ_API_KEY: "your-api-key"
      model: "whisper-large-v3"
      minimum_duration: 3
```

| Option | Description | Default |
|--------|-------------|---------|
| `GROQ_API_KEY` | Groq API key | (required) |
| `model` | Whisper model to use | `whisper-large-v3` |
| `minimum_duration` | Minimum audio duration in seconds | `3` |

---

#### hugging_face_whisper

Speech-to-text transcription using Hugging Face's Whisper implementation, supporting both API-based and local inference.

```yaml
links:
  hf_whisper:
    module: links.hugging_face_whisper
    options:
      model: "openai/whisper-large-v3"
      minimum_duration: 3
```

| Option | Description | Default |
|--------|-------------|---------|
| `model` | Hugging Face model identifier | `openai/whisper-large-v3` |
| `minimum_duration` | Minimum audio duration in seconds | `3` |

---

#### openai_transcribe

Speech-to-text transcription using OpenAI's Whisper API or Azure OpenAI. Supports automatic chunking for long audio files.

```yaml
links:
  openai_transcribe:
    module: links.openai_transcribe
    options:
      # Public OpenAI
      OPENAI_API_KEY: "sk-..."

      # Or Azure OpenAI
      AZURE_OPENAI_API_KEY: "your-key"
      AZURE_OPENAI_ENDPOINT: "https://your-resource.openai.azure.com"
      AZURE_OPENAI_API_VERSION: "2024-10-21"

      model: "gpt-4o-transcribe"
      language: "en"
      minimum_duration: 3
      max_chunk_duration: 480
      use_silence_chunking: true
      silence_thresh: -40
      silence_len: 2000
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (none) |
| `AZURE_OPENAI_API_KEY` | Azure OpenAI API key | (none) |
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI endpoint URL | (none) |
| `model` | Model to use | `gpt-4o-transcribe` |
| `language` | Language code | `en` |
| `minimum_duration` | Minimum audio duration in seconds | `3` |
| `max_chunk_duration` | Maximum chunk duration for splitting | `480` (8 min) |
| `use_silence_chunking` | Split at silence points | `true` |
| `silence_thresh` | Silence threshold in dBFS | `-40` |
| `silence_len` | Minimum silence length in ms | `2000` |

---

#### transcribe

Local transcription using the vCon library's built-in transcription capabilities.

```yaml
links:
  transcribe:
    module: links.transcribe
    options:
      transcribe_options:
        model_size: "base"
        output_options: ["vendor"]
```

| Option | Description | Default |
|--------|-------------|---------|
| `transcribe_options.model_size` | Model size | `base` |
| `transcribe_options.output_options` | Output format options | `["vendor"]` |

---

#### wtf_transcribe

Transcribes dialog recordings via the `vfun` transcription service and writes a [WTF (World Transcription Format)](../extensions/wtf-transcription.md)-shaped analysis entry. Refactored in May 2026 to decompose `run()` and normalize timeout option names.

```yaml
links:
  wtf:
    module: links.wtf_transcribe
    options:
      vfun-server-url: "https://wtf.example.com/transcribe"
      api-key: "your-vfun-key"
      language: "en"
      diarize: true
      vfun-timeout: 300
      url-timeout: 60
```

| Option | Description | Default |
|--------|-------------|---------|
| `vfun-server-url` | vfun transcription endpoint | (required) |
| `api-key` | Service API key | `None` |
| `language` | BCP-47 language hint | `None` (auto-detect) |
| `diarize` | Emit speaker labels | `false` |
| `vfun-timeout` | Transcription request timeout (s) | `300` |
| `url-timeout` | Media-fetch timeout (s) | `60` |

Writes an `analysis[]` entry with `type: "wtf_transcription"`, `vendor` inferred from the service response, `encoding: "json"`, and a WTF document in `body`. See [WTF Transcription extension](../extensions/wtf-transcription.md) for the body shape.

---

### Analysis Links

These links use AI to analyze and extract insights from vCon content.

---

#### analyze

OpenAI-powered analysis of vCon transcripts with customizable prompts, sampling, and retry mechanisms.

```yaml
links:
  analyze:
    module: links.analyze
    options:
      OPENAI_API_KEY: "sk-..."
      prompt: "Summarize this transcript in 3 bullet points"
      analysis_type: "summary"
      model: "gpt-4-turbo"
      sampling_rate: 1
      temperature: 0.3
      source:
        analysis_type: "transcript"
        text_location: "body.text"
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (required) |
| `prompt` | Analysis prompt | (required) |
| `analysis_type` | Type label for the analysis | `summary` |
| `model` | OpenAI model | `gpt-3.5-turbo-16k` |
| `sampling_rate` | Fraction of vCons to analyze (0-1) | `1` |
| `temperature` | Model temperature | `0.3` |
| `source.analysis_type` | Source analysis type to analyze | `transcript` |
| `source.text_location` | Path to text within source | `body.text` |

---

#### analyze_vcon

AI analysis of entire vCon objects, returning structured JSON output.

```yaml
links:
  analyze_vcon:
    module: links.analyze_vcon
    options:
      OPENAI_API_KEY: "sk-..."
      system_prompt: "You are a conversation analyst."
      prompt: "Analyze this vCon and return insights as JSON."
      analysis_type: "vcon_analysis"
      model: "gpt-4-turbo"
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (required) |
| `system_prompt` | System prompt for the model | (optional) |
| `prompt` | Analysis prompt | (required) |
| `analysis_type` | Type label for the analysis | `vcon_analysis` |
| `model` | OpenAI model | `gpt-4-turbo` |

---

#### detect_engagement

Detects whether both parties actively engaged in a conversation.

```yaml
links:
  engagement:
    module: links.detect_engagement
    options:
      OPENAI_API_KEY: "sk-..."
      prompt: "Did both the customer and the agent speak? Respond with 'true' or 'false'."
      analysis_type: "engagement_analysis"
      model: "gpt-4.1"
      source:
        analysis_type: "transcript"
        text_location: "body.paragraphs.transcript"
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (required) |
| `prompt` | Evaluation prompt | (engagement detection prompt) |
| `analysis_type` | Type label | `engagement_analysis` |
| `model` | OpenAI model | `gpt-4.1` |
| `sampling_rate` | Fraction to process | `1` |

Adds an `engagement` tag with value `true` or `false`.

---

#### analyze_and_label

Combined analysis that extracts labels/categories and applies them as tags.

```yaml
links:
  labeler:
    module: links.analyze_and_label
    options:
      OPENAI_API_KEY: "sk-..."
      prompt: "Analyze this transcript and provide relevant labels."
      analysis_type: "labeled_analysis"
      model: "gpt-4-turbo"
      response_format:
        type: "json_object"
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (required) |
| `prompt` | Label extraction prompt | (categorization prompt) |
| `analysis_type` | Type label | `labeled_analysis` |
| `model` | OpenAI model | `gpt-4-turbo` |
| `response_format` | Response format | `{"type": "json_object"}` |

Returns JSON with `labels` array and applies each label as a tag.

---

#### check_and_tag

Evaluates a condition using AI and applies a tag if the condition is met.

```yaml
links:
  check_complaint:
    module: links.check_and_tag
    options:
      OPENAI_API_KEY: "sk-..."
      tag_name: "complaint"
      tag_value: "detected"
      evaluation_question: "Does this conversation contain a customer complaint?"
      model: "gpt-5"
      source:
        analysis_type: "transcript"
        text_location: "body"
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (required) |
| `tag_name` | Tag name to apply | (required) |
| `tag_value` | Tag value to apply | (required) |
| `evaluation_question` | Question to evaluate | (required) |
| `model` | OpenAI model | `gpt-5` |

---

#### hugging_llm_link

AI analysis using Hugging Face language models, supporting both API and local inference.

```yaml
links:
  hf_analysis:
    module: links.hugging_llm_link
    options:
      model: "mistralai/Mistral-7B-Instruct-v0.2"
      prompt: "Summarize this conversation."
      analysis_type: "hf_summary"
```

| Option | Description | Default |
|--------|-------------|---------|
| `model` | Hugging Face model identifier | (required) |
| `prompt` | Analysis prompt | (required) |
| `analysis_type` | Type label | `hf_analysis` |

---

### Routing and Filtering Links

These links control vCon flow through chains.

---

#### sampler

Selectively processes vCons based on various sampling methods.

```yaml
links:
  sampler:
    module: links.sampler
    options:
      method: "percentage"  # percentage, rate, modulo, time
      percentage: 10        # For percentage method
      rate: 100            # For rate method (1 per N)
      modulo: 5            # For modulo method
```

| Option | Description | Default |
|--------|-------------|---------|
| `method` | Sampling method | `percentage` |
| `percentage` | Percentage to process (0-100) | `100` |
| `rate` | Process 1 out of N | `1` |
| `modulo` | Process if UUID modulo equals 0 | `1` |

Returns `None` for filtered vCons, stopping their chain processing.

---

#### jq_link

Filters vCons using jq expressions for complex content-based filtering.

```yaml
links:
  filter_sales:
    module: links.jq_link
    options:
      expression: '.parties[] | select(.role == "agent")'
      forward_on_match: true
      forward_list: "sales_ingress"
```

| Option | Description | Default |
|--------|-------------|---------|
| `expression` | jq expression to evaluate | (required) |
| `forward_on_match` | Continue chain if expression matches | `true` |
| `forward_list` | Alternative ingress list for matches | (none) |

---

#### tag_router

Routes vCons to additional Redis lists based on tags attached to the vCon. The vCon is pushed onto every matching target list; processing in the current chain continues unless `forward_original` is set to `false`.

```yaml
links:
  router:
    module: links.tag_router
    options:
      tag_routes:
        priority: "priority_queue"
        urgent: "urgent_queue"
        complaint: "complaint_review_queue"
      forward_original: true
```

| Option | Description | Default |
|--------|-------------|---------|
| `tag_routes` | Dict mapping tag value → target Redis list name. The link checks tags in the vCon's `attachments[]` of type `tags` against the keys here. | `{}` |
| `forward_original` | If `true`, continue the current chain after routing. If `false`, return `None` to stop the chain (vCon proceeds only on the routed queues). | `true` |

Returns `vcon_uuid` (chain continues) or `None` (chain stops) per `forward_original`.

---

### Data Management Links

These links modify vCon content.

---

#### tag

Adds configurable tags to vCons.

```yaml
links:
  add_tags:
    module: links.tag
    options:
      tags:
        - name: "source"
          value: "phone"
        - name: "processed"
          value: "true"
```

| Option | Description | Default |
|--------|-------------|---------|
| `tags` | List of tags to add | `[]` |
| `tags[].name` | Tag name | (required) |
| `tags[].value` | Tag value | (required) |

---

#### diet

Reduces vCon size by removing or redirecting elements. Useful for data minimization and privacy.

```yaml
links:
  slim_down:
    module: links.diet
    options:
      remove_dialog_bodies: true
      remove_attachments: true
      remove_analysis_types:
        - "raw_transcript"
      redirect_media_to_storage: "s3"
      remove_system_prompts: true
```

| Option | Description | Default |
|--------|-------------|---------|
| `remove_dialog_bodies` | Remove dialog body content | `false` |
| `remove_attachments` | Remove all attachments | `false` |
| `remove_analysis_types` | Analysis types to remove | `[]` |
| `redirect_media_to_storage` | Move media to storage | (none) |
| `remove_system_prompts` | Remove system prompts | `false` |

---

#### expire_vcon

Sets a Redis TTL on the vCon key so the working copy is cleaned up automatically. The vCon stays in any storage backends configured on the chain — this only affects the Redis hot cache.

```yaml
links:
  set_expiry:
    module: links.expire_vcon
    options:
      seconds: 86400  # 24 hours
```

| Option | Description | Default |
|--------|-------------|---------|
| `seconds` | TTL in seconds applied via `EXPIRE vcon:{uuid}` | `86400` (24 hours) |

---

### Integration Links

These links connect to external services.

---

#### webhook

POSTs the current vCon as JSON to one or more webhook URLs. The same module is available as a [storage backend](storage.md#webhook) if you'd rather invoke webhooks after the chain rather than mid-chain.

```yaml
links:
  notify:
    module: links.webhook
    options:
      webhook-urls:
        - "https://api.example.com/vcon-webhook"
        - "https://backup.example.com/vcon-webhook"
      headers:
        Authorization: "Bearer token123"
        x-conserver-api-token: "your-api-token"
```

| Option | Description | Default |
|--------|-------------|---------|
| `webhook-urls` | List of URLs to POST the vCon JSON to | `[]` |
| `headers` | Headers to attach to each request | `{}` |

Each URL is called sequentially with a `POST` containing the full vCon JSON. Per-call latency and status codes are recorded as OTEL metrics.

---

#### post_analysis_to_slack

Posts vCon analysis results to Slack channels.

```yaml
links:
  slack_notify:
    module: links.post_analysis_to_slack
    options:
      webhook_url: "https://hooks.slack.com/services/..."
      channel: "#vcon-alerts"
      analysis_type: "summary"
      template: "New conversation summary: {body}"
      condition:
        tag_name: "priority"
        tag_value: "high"
```

| Option | Description | Default |
|--------|-------------|---------|
| `webhook_url` | Slack webhook URL | (required) |
| `channel` | Slack channel | (required) |
| `analysis_type` | Analysis type to post | `summary` |
| `template` | Message template | `{body}` |
| `condition` | Optional tag condition | (none) |

---

### Audit and Compliance Links

These links provide integrity and audit trail capabilities.

---

#### datatrails

Creates [DataTrails](https://app.datatrails.ai) Events for each vCon, producing a tamper-evident audit trail via OIDC-authenticated calls. DataTrails statements map onto SCITT envelopes — if you want a vendor-neutral transparency service, prefer the [`scitt`](#scitt) link instead.

```yaml
links:
  audit:
    module: links.datatrails
    options:
      api_url: "https://app.datatrails.ai/archivist"
      auth_url: "https://app.datatrails.ai/archivist/iam/v1/appidp/token"
      client_id: "${DATATRAILS_CLIENT_ID}"
      client_secret: "${DATATRAILS_CLIENT_SECRET}"
      partner_id: "your-partner-id"
      asset_attributes:
        arc_display_type: "vcon_droid"
        conserver_link_version: "auto"
```

| Option | Description | Default |
|--------|-------------|---------|
| `api_url` | DataTrails Archivist API root | `https://app.datatrails.ai/archivist` |
| `auth_url` | OIDC client-credentials token endpoint | `https://app.datatrails.ai/archivist/iam/v1/appidp/token` |
| `client_id` / `client_secret` | OIDC client credentials | (required) |
| `partner_id` | Partner identifier used in event attribution | `not-set` |
| `asset_attributes` | Initial attributes for the DataTrails asset | DataTrails-recommended defaults |

DataTrails is the durable store for the audit data — the vCon itself is not modified.

---

#### scitt

Registers a COSE-signed statement about the current vCon on a [SCRAPI](https://datatracker.ietf.org/doc/draft-ietf-scitt-scrapi/)-compatible SCITT transparency service (such as [scittles](https://github.com/vcon-dev/scittles)), then verifies the returned COSE receipt and (optionally) stores it as an analysis entry on the vCon.

Added in May 2026 (SCITT v0.3.0). The lifecycle event recorded is controlled by `vcon_operation`; combine multiple instances of this link in a chain to record `vcon_created` early and `vcon_enhanced` after transcription.

```yaml
links:
  scitt_created:
    module: links.scitt
    options:
      scrapi_url: "http://scittles:8000"
      signing_key_pem: "${SCITT_SIGNING_KEY_PEM}"   # base64-encoded PEM (preferred for k8s/containers)
      # OR for local development:
      # signing_key_path: "/etc/scitt/signing-key.pem"
      issuer: "conserver"
      key_id: "conserver-key-1"
      vcon_operation: "vcon_created"
      store_receipt: true
```

| Option | Description | Default |
|--------|-------------|---------|
| `scrapi_url` | SCRAPI endpoint for the SCITT transparency service | `http://scittles:8000` |
| `signing_key_pem` | Base64-encoded PEM. Preferred for containers / k8s deployments. | `None` |
| `signing_key_path` | Filesystem path to the signing key (fallback for local development) | `/etc/scitt/signing-key.pem` |
| `issuer` | COSE issuer identifier | `conserver` |
| `key_id` | Key identifier | `conserver-key-1` |
| `vcon_operation` | Lifecycle event recorded (e.g. `vcon_created`, `vcon_enhanced`) | `vcon_created` |
| `store_receipt` | Append the COSE receipt as an analysis entry on the vCon | `true` |

Writes an `analysis[]` entry with `type: "scitt_receipt"`, `vendor: "scittles"`, and a body containing `entry_id`, `cose_receipt`, and `subject`. See [Lifecycle extension](../extensions/lifecycle.md) for how this composes with the vCon lifecycle audit story.

---

## Using Links in Chains

Links are combined into chains in the configuration:

```yaml
chains:
  main_pipeline:
    links:
      - deepgram           # Transcribe audio
      - analyze            # Generate summary
      - check_complaint    # Check for complaints
      - router             # Route based on tags
    storages:
      - postgres
      - s3
    ingress_lists:
      - incoming_calls
    egress_lists:
      - processed_calls
    enabled: 1
    timeout: 300
```

Links execute in order. If any link returns `None`, chain processing stops for that vCon.

## Common Patterns

### Conditional Processing

Use `sampler` or `jq_link` to process only certain vCons:

```yaml
chains:
  sample_analysis:
    links:
      - sampler    # Process 10% of vCons
      - analyze
```

### Multi-stage Analysis

Chain multiple analysis links for comprehensive processing:

```yaml
chains:
  full_analysis:
    links:
      - deepgram           # Step 1: Transcribe
      - analyze            # Step 2: Summarize
      - detect_engagement  # Step 3: Check engagement
      - check_complaint    # Step 4: Detect complaints
      - analyze_and_label  # Step 5: Categorize
```

### Tag-based Routing

Use tags to route vCons to different downstream chains:

```yaml
chains:
  intake:
    links:
      - deepgram
      - analyze
      - tag_router
    ingress_lists: [incoming]
    # No egress_lists - router handles distribution

  priority_handling:
    links:
      - notify_slack
      - priority_storage
    ingress_lists: [priority_queue]
```
